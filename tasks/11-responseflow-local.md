# Работа 11. Сквозное местное приложение ResponseFlow

[Основное условие](../responseflow-practice.md#работа-11-сквозное-приложение-responseflow)

## Чему учится разработчик

- собирать несколько проверенных частей в одно приложение;
- создавать точку сборки зависимостей;
- использовать одну прикладную службу из команды и HTTP-маршрута;
- сохранять отчёт запуска и продолжать после локальной ошибки анализатора.

## Подготовка

Используйте зависимости предыдущих работ. Не копируйте их командные оболочки.
Переносите модели, чистые функции и службы, затем приспосабливайте хранилища к
общей базе.

```text
src/responseflow_local/
  domain/
    messages.py
    findings.py
  application/
    import_messages.py
    analyze.py
    review.py
    report.py
  detectors/
    base.py
    unanswered.py
    commitments.py
    semantic_llm.py
  llm/
    provider.py
    fake_provider.py
    vsellm_provider.py
  storage/
    sqlite.py
    repositories.py
  api/
    routes.py
    main.py
  container.py
  cli.py
tests/
  test_import_flow.py
  test_analysis_flow.py
  test_review_flow.py
  test_end_to_end.py
```

## 1. Общий договор анализатора

`detectors/base.py`:

```python
from typing import Protocol

from responseflow_local.domain.findings import FindingCandidate
from responseflow_local.domain.messages import ConversationWindow


class SituationDetector(Protocol):
    @property
    def name(self) -> str: ...

    @property
    def version(self) -> str: ...

    def detect(self, window: ConversationWindow) -> tuple[FindingCandidate, ...]: ...
```

Все три анализатора реализуют один договор. Приложение не проверяет их
конкретные классы через `isinstance`.

## 2. Отчёт запуска анализа

`domain/findings.py`:

```python
from typing import Literal

from pydantic import AwareDatetime, BaseModel, ConfigDict, Field


class DetectorRun(BaseModel):
    model_config = ConfigDict(frozen=True)
    detector_name: str
    detector_version: str
    status: Literal["succeeded", "failed"]
    candidate_count: int = Field(ge=0)
    error: str | None = None


class AnalysisRunReport(BaseModel):
    workspace_id: str
    analysis_time: AwareDatetime
    detector_runs: tuple[DetectorRun, ...]
    created_findings: int
    updated_findings: int
```

Ошибка сохраняется как безопасное описание без содержимого секретов и полного
технического стека.

## 3. Прикладная служба анализа

`application/analyze.py`:

```python
from datetime import datetime

from responseflow_local.domain.findings import AnalysisRunReport, DetectorRun
from responseflow_local.detectors.base import SituationDetector


def safe_detector_error(error: Exception) -> str:
    """Не включать в отчёт запрос, ответ модели, ключи и стек вызовов."""
    return f"{type(error).__name__}: анализатор не завершил обработку"


class AnalysisService:
    def __init__(self, message_repository, finding_service,
                 window_builder,
                 detectors: tuple[SituationDetector, ...]) -> None:
        self._messages = message_repository
        self._findings = finding_service
        self._window_builder = window_builder
        self._detectors = detectors

    def run(self, workspace_id: str,
            analysis_time: datetime) -> AnalysisRunReport:
        detector_runs: list[DetectorRun] = []
        created = 0
        updated = 0
        for conversation_id in self._messages.list_conversations(workspace_id):
            window = self._window_builder.build(
                workspace_id, conversation_id, analysis_time
            )
            for detector in self._detectors:
                try:
                    candidates = detector.detect(window)
                    # TODO: сохранить каждого кандидата и обновить счётчики.
                    detector_runs.append(DetectorRun(
                        detector_name=detector.name,
                        detector_version=detector.version,
                        status="succeeded",
                        candidate_count=len(candidates),
                    ))
                except Exception as error:
                    detector_runs.append(DetectorRun(
                        detector_name=detector.name,
                        detector_version=detector.version,
                        status="failed",
                        candidate_count=0,
                        error=safe_detector_error(error),
                    ))
        # TODO: собрать отчёт и сохранить сам запуск.
        raise NotImplementedError
```

Для учебной работы допустим широкий перехват исключения только на границе
независимого анализатора. Внутри правил перехватывайте конкретные ошибки.

## 4. Точка сборки

`container.py`:

```python
from dataclasses import dataclass
from pathlib import Path

from responseflow_local.application.analyze import AnalysisService
from responseflow_local.application.import_messages import MessageImportService
from responseflow_local.application.report import ReportService
from responseflow_local.application.review import InboxService
from responseflow_local.storage.repositories import (
    SqliteFindingRepository,
    SqliteMessageRepository,
)
from responseflow_local.storage.sqlite import SqliteDatabase


@dataclass(frozen=True)
class ApplicationContainer:
    import_service: MessageImportService
    analysis_service: AnalysisService
    inbox_service: InboxService
    report_service: ReportService


def build_application(database_path: Path, fixture_root: Path) -> ApplicationContainer:
    database = SqliteDatabase(database_path)
    database.initialize()

    message_repository = SqliteMessageRepository(database)
    finding_repository = SqliteFindingRepository(database)
    # TODO: создать каталоги, анализаторы и службы, затем вернуть контейнер.
    raise NotImplementedError
```

Это единственное место, где конкретные реализации соединяются друг с другом.
Предметные службы не создают SQLite самостоятельно.

## 5. Командная оболочка

`cli.py`:

```python
from datetime import datetime
from pathlib import Path

import typer

from .container import ApplicationContainer, build_application

app = typer.Typer()


def container(database: Path) -> ApplicationContainer:
    return build_application(database, Path("data/fixtures"))


@app.command("analyze")
def analyze(database: Path, workspace: str, analysis_time: datetime) -> None:
    report = container(database).analysis_service.run(workspace, analysis_time)
    typer.echo(report.model_dump_json(indent=2))
```

Остальные команды должны быть такими же тонкими: разобрать параметры, вызвать
службу, вывести результат.

## 6. HTTP-приложение с теми же службами

`api/main.py`:

```python
from pathlib import Path

from fastapi import FastAPI

from responseflow_local.container import build_application
from .routes import create_router


def create_app(database_path: Path) -> FastAPI:
    container = build_application(database_path, Path("data/fixtures"))
    app = FastAPI(title="ResponseFlow Local")
    app.include_router(create_router(container.inbox_service))
    return app
```

Маршруты получают `InboxService`; они не вызывают функции Typer.

## 7. Сквозной тест

```python
def test_full_reopen_flow(tmp_path, fixture_root) -> None:
    database = tmp_path / "responseflow.sqlite"
    app = build_application(database, fixture_root)

    messages_import = app.import_service.import_file(
        fixture_root / "messages/messages.jsonl"
    )
    revisions_import = app.import_service.import_file(
        fixture_root / "messages/revisions.jsonl"
    )
    assert messages_import.created + revisions_import.created == 44

    app.analysis_service.run("north", ANALYSIS_TIME)
    finding = app.inbox_service.find_by_subject(
        "north", "commitment:conv-n-001:n-001-006"
    )
    app.inbox_service.resolve(
        "north", finding.finding_id,
        expected_version=finding.version,
        author="manager-olga",
        comment="Клиенту отправлена исправленная смета.",
        occurred_at=RESOLUTION_TIME,
    )

    app.analysis_service.run("north", ANALYSIS_TIME)
    assert app.inbox_service.get("north", finding.finding_id).status == "resolved"

    app.import_service.import_file(fixture_root / "messages/new-evidence.jsonl")
    app.analysis_service.run("north", NEXT_ANALYSIS_TIME)
    reopened = app.inbox_service.get("north", finding.finding_id)
    assert reopened.status == "reopened"
```

Число 44 проверяет 43 исходных сообщения и отдельную редакцию. Файл
`new-evidence.jsonl` на первом этапе намеренно не импортируется: он нужен только
после закрытия карточки. Каталоги также не входят в этот счётчик.

## 8. Реализация по этапам

1. `init-db` и повторная инициализация без удаления.
2. Импорт сообщений и повторный импорт.
3. Только анализатор обращений без ответа.
4. Сохранение карточки и приоритета.
5. Анализатор обещаний.
6. `LlmSituationAnalyzer` с заглушкой поставщика по умолчанию.
7. Явная настройка `LLM_PROVIDER=vsellm` для ручного сетевого запуска.
8. Учёт поставщика, модели, версии запроса и токенов в отчёте.
9. Отчёт локальной ошибки одного анализатора.
10. Список и подробная карточка.
11. Решения менеджера.
12. Закрытие, старый повтор и новое основание.
13. Сводный отчёт.
14. Ручной вызов VseLLM для одной синтетической беседы.
15. HTTP-маршруты поверх тех же служб.

## Что не следует объединять

- анализатор не читает SQLite;
- хранилище не рассчитывает приоритет;
- команда не выполняет переход состояния самостоятельно;
- маршрут не создаёт службу на каждый предметный шаг;
- модель Pydantic не открывает файл;
- ключ VseLLM не хранится в базе и не попадает в отчёт;
- отчёт не изменяет данные.

## Связь со Spine

Это приложение является переносимым предметным срезом. При интеграции Spine
заменит импорт, контекст, длительные процессы, подтверждения и хранилище.
ActionFlow сохранит типы ситуаций, анализаторы, приоритеты, рекомендации и
правила переоткрытия.
