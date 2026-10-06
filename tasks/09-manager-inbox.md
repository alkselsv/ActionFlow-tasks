# Работа 09. Очередь менеджера

[Основное условие](../responseflow-practice.md#работа-09-очередь-менеджера)

## Чему учится разработчик

- сначала создавать прикладную службу, затем HTTP-оболочку;
- проверять допустимые переходы состояний;
- защищаться от одновременного изменения версии;
- скрывать существование данных другой рабочей области.

## Подготовка и библиотеки

```bash
uv add 'fastapi>=0.115,<1' 'uvicorn>=0.30,<1'
uv add --dev 'httpx>=0.27,<1' 'pytest-asyncio>=0.23,<2'
```

FastAPI связывает маршруты с моделями Pydantic. Uvicorn нужен только для ручного
запуска. httpx вызывает приложение в тесте без открытия сетевого порта.

```text
src/manager_inbox/
  schemas.py
  errors.py
  repository.py
  service.py
  dependencies.py
  routes.py
  main.py
tests/
  test_service.py
  test_routes.py
```

## 1. Входные схемы решений

`schemas.py`:

```python
from typing import Literal
from uuid import UUID

from pydantic import AwareDatetime, BaseModel, ConfigDict, Field

RejectionReason = Literal[
    "not_actionable", "wrong_type", "already_resolved", "insufficient_evidence"
]


class DecisionBase(BaseModel):
    author: str = Field(min_length=1)
    expected_version: int = Field(ge=1)
    occurred_at: AwareDatetime


class AcknowledgeRequest(DecisionBase):
    comment: str = "Принято в работу"


class SnoozeRequest(DecisionBase):
    until: AwareDatetime
    reason: str = Field(min_length=3)


class ResolveRequest(DecisionBase):
    comment: str = Field(min_length=10)


class RejectRequest(DecisionBase):
    reason_code: RejectionReason
    comment: str = Field(min_length=10)


class EvidenceView(BaseModel):
    message_external_id: str
    sent_at: AwareDatetime
    author: str
    text: str


class DecisionEvent(BaseModel):
    action: Literal["acknowledge", "snooze", "resolve", "reject"]
    author: str
    at: AwareDatetime
    comment: str
    from_status: str
    to_status: str
    version: int


class FindingSummary(BaseModel):
    finding_id: UUID
    workspace_id: str
    version: int
    conversation_id: str
    project_id: str
    status: Literal[
        "open", "acknowledged", "snoozed", "resolved", "reopened", "rejected"
    ]
    priority: Literal["low", "medium", "high", "critical"]
    summary: str


class FindingDetail(FindingSummary):
    organization_name: str
    project_name: str
    type: str
    score: int
    priority_reasons: tuple[str, ...]
    recommended_action: str
    confidence: float
    detector_version: str
    evidence: tuple[EvidenceView, ...]
    history: tuple[DecisionEvent, ...]


class InboxSeed(BaseModel):
    model_config = ConfigDict(extra="forbid")
    analysis_time: AwareDatetime
    findings: tuple[FindingDetail, ...]
    rejection_reason_codes: tuple[RejectionReason, ...]
```

Успешное отклонение переводит карточку в отдельное конечное состояние
`rejected`. Оно отличается от `resolved`: в первом случае менеджер считает
срабатывание неверным или не требующим действия, во втором подтверждает, что
реальная ситуация разрешена.

Проверку «дата откладывания находится в будущем относительно `occurred_at`»
можно сделать валидатором модели либо в службе. Выберите одно место и не
дублируйте правило.

## 2. Предметные ошибки

`errors.py`:

```python
class FindingNotFoundError(LookupError):
    pass


class InvalidTransitionError(ValueError):
    pass


class VersionConflictError(ValueError):
    pass
```

Эти ошибки ничего не знают о кодах HTTP. Их преобразует внешний слой.

`repository.py`:

```python
from pathlib import Path
from typing import Protocol
from uuid import UUID

from .schemas import DecisionEvent, FindingDetail, InboxSeed


class InboxRepository(Protocol):
    def list_for_workspace(self, workspace_id: str) -> tuple[FindingDetail, ...]: ...
    def get(self, workspace_id: str, finding_id: UUID) -> FindingDetail | None: ...
    def save_with_event(self, finding: FindingDetail,
                        event: DecisionEvent) -> None: ...


class InMemoryInboxRepository:
    """Учебное хранилище, загружаемое из inbox/seed.json."""

    @classmethod
    def from_seed(cls, path: Path) -> "InMemoryInboxRepository":
        # TODO: прочитать JSON, проверить через InboxSeed и построить словарь
        # по паре (workspace_id, finding_id).
        raise NotImplementedError

    # TODO: реализовать методы протокола и возвращать копии моделей.
```

## 3. Служба до создания маршрутов

`service.py`:

```python
from uuid import UUID

from .errors import FindingNotFoundError, InvalidTransitionError, VersionConflictError
from .repository import InboxRepository
from .schemas import AcknowledgeRequest, FindingDetail, FindingSummary

PRIORITY_ORDER = {"critical": 0, "high": 1, "medium": 2, "low": 3}


class InboxService:
    def __init__(self, repository: InboxRepository) -> None:
        self._repository = repository

    def list_findings(self, workspace_id: str, status: str | None = None,
                      priority: str | None = None,
                      project_id: str | None = None) -> tuple[FindingSummary, ...]:
        items = self._repository.list_for_workspace(workspace_id)
        # TODO: применить отбор, значения по умолчанию и сортировку.
        raise NotImplementedError

    def get_finding(self, workspace_id: str, finding_id: UUID) -> FindingDetail:
        item = self._repository.get(workspace_id, finding_id)
        if item is None:
            raise FindingNotFoundError(str(finding_id))
        return item

    def acknowledge(self, workspace_id: str, finding_id: UUID,
                    command: AcknowledgeRequest) -> FindingDetail:
        item = self.get_finding(workspace_id, finding_id)
        if item.version != command.expected_version:
            raise VersionConflictError("Карточка уже была изменена")
        if item.status not in {"open", "reopened"}:
            raise InvalidTransitionError(f"Нельзя подтвердить из {item.status}")
        # TODO: добавить историю, увеличить версию и сохранить одной операцией.
        raise NotImplementedError
```

Сначала напишите проверки непосредственно для `InboxService`. FastAPI пока не
создавайте.

## 4. Сборка зависимостей

`dependencies.py`:

```python
from functools import lru_cache
from pathlib import Path

from .repository import InMemoryInboxRepository
from .service import InboxService


@lru_cache
def get_inbox_service() -> InboxService:
    repository = InMemoryInboxRepository.from_seed(
        Path("data/fixtures/seed.json")
    )
    return InboxService(repository)
```

В тестах зависимость заменяется, поэтому тесты не используют общие изменяемые
данные между запусками.

## 5. Маршруты

`routes.py`:

```python
from uuid import UUID

from fastapi import APIRouter, Depends

from .dependencies import get_inbox_service
from .schemas import AcknowledgeRequest, FindingDetail
from .service import InboxService

router = APIRouter(prefix="/workspaces/{workspace_id}/findings")


@router.get("")
def list_findings(workspace_id: str,
                  service: InboxService = Depends(get_inbox_service)):
    return service.list_findings(workspace_id)


@router.post("/{finding_id}/acknowledge", response_model=FindingDetail)
def acknowledge(workspace_id: str, finding_id: UUID,
                command: AcknowledgeRequest,
                service: InboxService = Depends(get_inbox_service)):
    return service.acknowledge(workspace_id, finding_id, command)
```

`main.py` создаёт `FastAPI()`, подключает маршруты и обработчики исключений.

## 6. Проверка маршрута в памяти

```python
import pytest
from httpx import ASGITransport, AsyncClient

from manager_inbox.main import app


@pytest.mark.asyncio
async def test_north_cannot_read_south_finding() -> None:
    transport = ASGITransport(app=app)
    async with AsyncClient(transport=transport, base_url="http://test") as client:
        response = await client.get(
            "/workspaces/north/findings/20000000-0000-4000-8000-000000000001"
        )
    assert response.status_code == 404
```

## Порядок реализации

1. Модели входа и ответа.
2. Хранилище в памяти.
3. Список и сортировка через службу.
4. Получение карточки и изоляция области.
5. Один переход и версия.
6. Остальные переходы.
7. История.
8. Сборка зависимостей.
9. Маршруты чтения.
10. Маршруты решений и коды ошибок.

## Связь со Spine

Позже очередь может предоставляться платформой, а решение человека стать шагом
подтверждения Spine. Прикладные переходы, причина решения и защита версии всё
равно должны быть явно определены.
