# Работа 11. Сквозное местное приложение ResponseFlow

[Основное условие](../responseflow-practice.md#работа-11-сквозное-приложение-responseflow)

## Чему учится разработчик

- собирать несколько проверенных частей в одно приложение;
- создавать точку сборки зависимостей;
- использовать одну прикладную службу из команды и HTTP-маршрута;
- сохранять отчёт запуска и продолжать после локальной ошибки анализатора.

## Как устроена эта работа

Финальная работа не требует заново изобретать правила предыдущих заданий. Её
цель — собрать проверенные части так, чтобы зависимости были видны в одном
месте, а командная программа и HTTP-приложение использовали одни и те же службы.
Переносите только предметные модели, чистые функции и службы; отдельные команды
ранних работ в общий проект не копируются.

Поток начинается с импорта и построения окна. Несколько анализаторов получают
один договор и возвращают кандидатов. Служба карточек устраняет повторы,
приоритет определяет порядок реакции, а очередь принимает решение человека.
Каждый этап получает готовый результат предыдущего, но не открывает его
хранилище напрямую.

Точка сборки создаёт базу, репозитории, анализаторы и службы. Это единственное
место, которое знает конкретные реализации. Благодаря этому тест может собрать
приложение с временной SQLite, а позднее платформа сможет передать другие
хранилища и поставщика LLM без изменения предметного кода.

Ошибка одного анализатора записывается в отчёт и не останавливает остальные.
Это не означает, что любые ошибки нужно скрывать. Изолируется только граница
независимого анализатора; ошибки инициализации базы, неверных настроек или
сборки зависимостей должны остановить запуск. Безопасное сообщение в отчёте не
содержит ключи, полный запрос LLM и стек вызовов.

Сквозной сценарий особенно важен для правила переоткрытия. Сначала импортируется
основная история, создаётся и закрывается карточка. Повтор того же анализа не
отменяет решение. Только после импорта нового основания следующий запуск
переоткрывает ту же карточку, сохраняя её идентификатор и историю.

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

## Разбор подходов, классов и методов

`SituationDetector` — общий договор всех анализаторов. Свойства `name` и
`version` отделены от метода `detect`, чтобы отчёт мог описать реализацию, не
проверяя её конкретный класс. Анализатор получает готовый `ConversationWindow`
и возвращает кандидатов; он не открывает базу и не сохраняет карточки.

`DetectorRun` описывает работу одного анализатора в одной беседе или в принятой
вами единице запуска. `status` ограничен двумя значениями, `candidate_count`
неотрицателен, а `error` заполняется только при сбое. Полезно добавить
проверяющий метод модели, запрещающий ошибку у `succeeded` и требующий её у
`failed`.

`AnalysisRunReport` объединяет запуски анализаторов и итоговые числа созданных и
обновлённых карточек. Он является наблюдаемым результатом команды анализа.
Отчёт нужно сохранить, но его формирование не должно менять сами карточки после
завершения основного цикла.

`safe_detector_error(error)` намеренно сообщает только класс безопасной ошибки
и общий текст. Подробный стек можно оставить в защищённом техническом журнале,
если такой журнал появится, но учебный предметный отчёт не должен содержать
текст запроса модели или секрет.

`AnalysisService` получает все зависимости через конструктор. Поле `_messages`
перечисляет беседы, `_window_builder` строит контекст, `_detectors` анализируют,
`_findings` сохраняет результат. Метод `run(workspace_id, analysis_time)`
координирует эти действия и не создаёт конкретные реализации самостоятельно.

Внутренний цикл сначала строит одно окно беседы, затем передаёт его каждому
анализатору. Успешные кандидаты по одному записываются через службу карточек, а
её результат сообщает, была карточка создана или обновлена. Исключение
перехватывается вокруг одного вызова `detector.detect`, поэтому следующий
анализатор продолжает работу. Не помещайте в этот широкий перехват сохранение
карточек: ошибка базы не является локальной ошибкой анализатора.

`ApplicationContainer` — подписанный набор готовых служб. Класс с
`@dataclass(frozen=True)` удобнее неименованного словаря: редактор знает типы
полей, а случайно заменить службу после сборки нельзя. Контейнер не является
глобальным объектом; его явно создаёт `build_application`.

`build_application(database_path, fixture_root)` — корень графа зависимостей.
Он создаёт `SqliteDatabase`, инициализирует схему, затем создаёт репозитории,
каталоги, поставщика LLM, анализаторы и службы в направлении снизу вверх. Путь
к базе и корень данных являются параметрами, поэтому тест использует временную
директорию, а команда — рабочие пути.

`container(database)` в командном модуле является маленьким помощником,
который выбирает стандартный путь данных. Команда `analyze` разбирает аргументы,
вызывает `analysis_service.run` и сериализует отчёт. Другие команды следуют тому
же шаблону и не обращаются к репозиториям напрямую.

`create_app(database_path)` собирает тот же контейнер и передаёт
`inbox_service` в `create_router`. Маршрут и команда могут иметь разный формат
входа и выхода, но предметные изменения выполняются одной службой. В тестах
лучше создавать новое приложение на временной базе, а не импортировать заранее
собранный глобальный объект.

В сквозном тесте `build_application` возвращает доступ к службам, а не к
внутренним таблицам. `import_file` загружает основную историю и редакцию,
`analysis_service.run` создаёт карточку, `inbox_service.resolve` выполняет
решение, затем новое основание приводит к `reopened`. Проверка через публичные
методы показывает, что части действительно соединены правильно.

Общий поток приложения:

```text
JSONL → MessageImportService → SQLite
     → WindowBuilder → SituationDetector[]
     → FindingService → PriorityService
     → InboxService → решение менеджера
     → новое основание → повторный анализ → reopened
```

Каждая стрелка является границей, на которой полезно иметь отдельный договор и
тест. Это упрощает будущую замену файлов и SQLite платформенными возможностями.

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

## Как проверять себя по ходу работы

Собирайте приложение вертикальными частями. Сначала команда импорта должна
записать сообщения в общую базу. Затем подключите один анализатор и добейтесь
появления одной карточки. После этого добавьте приоритет, чтение очереди и
решение менеджера. Такой порядок позволяет после каждого шага выполнить
видимый пользовательский сценарий.

После подключения каждого анализатора добавляйте две проверки: успешный запуск
и искусственную локальную ошибку. При ошибке отчёт этого анализатора имеет
`failed`, но другой анализатор всё равно создаёт кандидатов. При повторном
успешном запуске количество карточек не растёт, хотя счётчик обнаружений может
измениться.

Команда и HTTP-маршрут должны обращаться к одной и той же прикладной службе.
Если одинаковое действие реализовано дважды, сравните результаты на одной базе:
различие почти всегда означает, что предметное правило оказалось во внешней
оболочке.

Финальная ручная демонстрация выполняется на чистом временном каталоге по
командам из README. Наставник должен без изменения путей импортировать данные,
запустить анализ, открыть карточку, закрыть её, добавить новое основание и
увидеть переоткрытие. После этого повторите автоматические проверки без сети.

## Связь со Spine

Это приложение является переносимым предметным срезом. При интеграции Spine
заменит импорт, контекст, длительные процессы, подтверждения и хранилище.
ActionFlow сохранит типы ситуаций, анализаторы, приоритеты, рекомендации и
правила переоткрытия.
