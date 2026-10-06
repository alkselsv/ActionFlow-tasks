# Работа 07. Хранилище ситуаций

[Основное условие](../responseflow-practice.md#работа-07-карточка-ситуации-основания-и-устранение-повторов)

## Чему учится разработчик

- хранить карточку отдельно от оснований;
- вычислять ключ повтора в приложении;
- выполнять несколько записей одной транзакцией;
- безопасно переоткрывать закрытую ситуацию.

## Как устроена эта работа

Анализаторы запускаются повторно и могут каждый раз находить одну и ту же
проблему. Хранилище должно превратить поток кандидатов в устойчивые карточки.
Кандидат — это наблюдение в конкретный момент, а карточка — длительное состояние,
которое переживает много запусков анализа и решения менеджера.

Для узнавания одной ситуации используется ключ повтора. Он строится внутри
приложения из рабочей области, беседы, типа и ключа предмета. Анализатор не
передаёт готовый хеш, иначе ошибочный или недоверенный источник сможет случайно
объединить чужие карточки. Исходные значения нормализуются до хеширования, но
сами предметные данные в карточке не переписываются.

Основания хранятся отдельно от карточки. Это позволяет добавлять новое
сообщение к уже существующей ситуации и запрещать повтор одной и той же пары.
История событий отвечает на другой вопрос: не «на каких сообщениях основан
вывод», а «что происходило с карточкой». Смешивать основания и историю в одном
списке не следует.

Карточка, основания и событие одного обнаружения записываются в одной
транзакции. Если добавление основания завершилось ошибкой, карточка без
объяснения не должна остаться в базе. Вариант в памяти имитирует то же поведение,
чтобы предметные тесты не зависели от SQLite.

Переоткрытие требует именно нового основания. Повторный анализ старого окна
может увеличить счётчик обнаружений, но не отменяет решение менеджера о
закрытии. Новое сообщение клиента показывает, что ситуация продолжилась, и
поэтому перевод в `reopened` становится обоснованным.

## Подготовка

SQLite доступен через стандартный модуль `sqlite3`, отдельная библиотека не
нужна. Для начала всё равно создайте вариант в памяти.

```text
src/finding_store/
  models.py
  key_builder.py
  repository.py
  sqlite_repository.py
  service.py
  history.py
tests/
  test_key_builder.py
  test_service.py
  test_sqlite_repository.py
```

## 1. Модели

```python
from typing import Literal
from uuid import UUID

from pydantic import AwareDatetime, BaseModel, ConfigDict, Field

FindingStatus = Literal[
    "open", "acknowledged", "snoozed", "resolved", "reopened", "rejected"
]


class FindingCandidate(BaseModel):
    """Результат анализатора до записи в хранилище."""
    model_config = ConfigDict(frozen=True, extra="forbid")
    workspace_id: str
    conversation_id: str
    project_id: str
    type: str
    confidence: float = Field(ge=0, le=1)
    summary: str
    recommended_action: str
    evidence_message_ids: tuple[str, ...] = Field(min_length=1)
    subject_key: str
    detected_at: AwareDatetime
    due_at: AwareDatetime | None = None
    flags: tuple[str, ...] = ()


class Finding(BaseModel):
    model_config = ConfigDict(frozen=True)
    finding_id: UUID
    workspace_id: str
    conversation_id: str
    type: str
    subject_key: str
    deduplication_key: str
    status: FindingStatus
    summary: str
    recommended_action: str
    confidence: float = Field(ge=0, le=1)
    first_detected_at: AwareDatetime
    last_detected_at: AwareDatetime
    occurrence_count: int = Field(ge=1)


class Evidence(BaseModel):
    finding_id: UUID
    message_external_id: str
    added_at: AwareDatetime


class FindingEvent(BaseModel):
    finding_id: UUID
    event_type: Literal["created", "detected_again", "resolved", "reopened"]
    occurred_at: AwareDatetime
    details: str
```

## 2. Ключ повтора

`key_builder.py`:

```python
from hashlib import sha256


def _normalize(value: str) -> str:
    return value.strip().lower()


def build_deduplication_key(workspace_id: str, conversation_id: str,
                            finding_type: str, subject_key: str) -> str:
    source = "|".join(map(_normalize, (
        workspace_id, conversation_id, finding_type, subject_key
    )))
    return sha256(source.encode("utf-8")).hexdigest()
```

Проверьте длину 64 символа, устойчивость и разные ключи для разных типов.

## 3. Договор хранилища

`repository.py`:

```python
from contextlib import AbstractContextManager
from typing import Protocol
from uuid import UUID

from .models import Evidence, Finding, FindingEvent


class FindingRepository(Protocol):
    def transaction(self) -> AbstractContextManager[None]: ...
    def find_by_key(self, workspace_id: str, key: str) -> Finding | None: ...
    def save_finding(self, finding: Finding) -> None: ...
    def list_evidence(self, finding_id: UUID) -> tuple[Evidence, ...]: ...
    def add_evidence(self, evidence: Evidence) -> None: ...
    def add_event(self, event: FindingEvent) -> None: ...
```

В варианте в памяти `transaction()` может быть простым контекстным менеджером.
В SQLite он должен выполнять `commit` при успехе и `rollback` при исключении.

## 4. Схема SQLite

`sqlite_repository.py` содержит инициализацию:

```python
SCHEMA = """
CREATE TABLE IF NOT EXISTS findings (
    finding_id TEXT PRIMARY KEY,
    workspace_id TEXT NOT NULL,
    deduplication_key TEXT NOT NULL,
    payload_json TEXT NOT NULL,
    UNIQUE (workspace_id, deduplication_key)
);

CREATE TABLE IF NOT EXISTS evidence (
    finding_id TEXT NOT NULL,
    message_external_id TEXT NOT NULL,
    added_at TEXT NOT NULL,
    UNIQUE (finding_id, message_external_id),
    FOREIGN KEY (finding_id) REFERENCES findings(finding_id)
);

CREATE TABLE IF NOT EXISTS finding_events (
    event_id INTEGER PRIMARY KEY AUTOINCREMENT,
    finding_id TEXT NOT NULL,
    occurred_at TEXT NOT NULL,
    payload_json TEXT NOT NULL,
    FOREIGN KEY (finding_id) REFERENCES findings(finding_id)
);
"""
```

Хранить всю карточку JSON допустимо для этой учебной работы, но ключи области и
повтора вынесены в отдельные столбцы для ограничений и поиска.

## 5. Служба

`service.py`:

```python
from uuid import uuid4

from .key_builder import build_deduplication_key
from .models import Evidence, Finding, FindingCandidate, FindingEvent
from .repository import FindingRepository


class FindingService:
    def __init__(self, repository: FindingRepository) -> None:
        self._repository = repository

    def record(self, candidate: FindingCandidate) -> Finding:
        key = build_deduplication_key(
            candidate.workspace_id, candidate.conversation_id,
            candidate.type, candidate.subject_key,
        )
        with self._repository.transaction():
            existing = self._repository.find_by_key(candidate.workspace_id, key)
            if existing is None:
                return self._create(candidate, key)
            return self._update(existing, candidate)

    def _create(self, candidate: FindingCandidate, key: str) -> Finding:
        # TODO: создать UUID, карточку, основания и событие.
        raise NotImplementedError

    def _update(self, existing: Finding, candidate: FindingCandidate) -> Finding:
        # TODO: найти новые основания, обновить счётчик и решить вопрос переоткрытия.
        raise NotImplementedError
```

Пример ожидаемого поведения:

```python
first = service.record(candidate)
second = service.record(candidate)
assert second.finding_id == first.finding_id
assert second.occurrence_count == 2
```

Обратите внимание: `occurrence_count` отсутствует у `FindingCandidate` и есть
только у сохранённой `Finding`. Значение вычисляет служба: первый кандидат даёт
`1`, а каждое повторное обнаружение увеличивает сохранённый счётчик. Нельзя
доверять счётчику из входного файла или анализатора.

## 6. Тест переоткрытия

```python
def test_resolved_finding_reopens_only_with_new_evidence(service, candidate) -> None:
    created = service.record(candidate)
    service.resolve(created.finding_id, at=RESOLUTION_TIME)

    same = service.record(candidate)
    assert same.status == "resolved"

    updated = candidate.model_copy(update={
        "evidence_message_ids": (*candidate.evidence_message_ids, "n-001-014"),
        "detected_at": NEXT_DAY,
    })
    reopened = service.record(updated)
    assert reopened.status == "reopened"
```

## Порядок реализации

1. Ключ и его тесты.
2. Хранилище в памяти.
3. Создание карточки без оснований.
4. Добавление оснований и события одной операцией.
5. Повторное обнаружение.
6. Закрытие.
7. Переоткрытие новым основанием.
8. Схема SQLite.
9. Транзакция и искусственный откат.
10. Полный готовый набор дважды.

## Как проверять себя по ходу работы

Начните с чистой функции ключа: одинаковые значения с лишними пробелами и
разным регистром должны давать один хеш, а изменение типа или рабочей области —
другой. Затем проверьте создание карточки без SQLite. Только после правильного
поведения службы подключайте таблицы.

После каждого вызова `record` проверяйте сразу четыре вещи: идентификатор
карточки, счётчик обнаружений, набор оснований и последнее событие. Проверка
только числа строк в таблице может пропустить ошибку обновления содержимого.

Для испытания транзакции специально заставьте хранилище выбросить исключение
при добавлении основания. После выхода из операции не должно быть ни частично
обновлённой карточки, ни события. В SQLite также включите внешние ключи для
каждого соединения: наличие ограничений в схеме само по себе ещё не включает их
проверку.

Следите за границей ответственности: `FindingCandidate` не содержит
`occurrence_count`, состояния и номера версии. Эти значения принадлежат
хранилищу. Кандидат сообщает только то, что анализатор увидел сейчас.

## Связь со Spine

`Finding`, `Evidence` и история соответствуют платформенным понятиям Spine.
Местное хранилище будет заменено, но правило ключа, отсутствие дубликатов и
безопасное переоткрытие должны сохраниться.
