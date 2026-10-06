# Работа 02. Сборка окна переписки

[Основное условие](../responseflow-practice.md#работа-02-сборка-окна-переписки)

## Чему учится разработчик

- объединять данные из нескольких файлов;
- выбирать редакцию, известную на заданный момент;
- строить воспроизводимый ограниченный контекст;
- отделять внутреннюю модель сообщения от представления для анализатора.

## Как устроена эта работа

Анализатору нельзя отдавать все когда-либо полученные записи. В хранилище могут
одновременно находиться старые редакции, сообщения из будущего относительно
выбранного времени анализа и служебные поля, которые не помогают принять
решение. Сборщик окна превращает такую историю в небольшой, однозначный снимок
переписки.

Порядок действий принципиален. Сначала ограничьте данные рабочей областью и
беседой, затем выберите редакции, известные к моменту анализа, и только после
этого сортируйте и применяйте предел. Если сначала взять последние 12 строк, а
потом удалить будущие сообщения, окно может оказаться короче и потерять нужный
контекст. Если выбрать редакцию только по максимальному номеру, можно случайно
использовать изменение, которое во время анализа ещё не было известно.

`MessageRevision` описывает хранимый факт, а `MessageView` — безопасное
представление для анализатора. В представлении появляется понятное имя
участника, но исчезают внутренние сведения, не нужные для анализа. Такое
преобразование помогает контролировать объём контекста и не передавать наружу
лишние данные.

Особое правило относится к ответам. Последнее сообщение может ссылаться на
более раннее сообщение, выпавшее из ограничения. Тогда исходное сообщение
нужно вернуть в окно, иначе человек или модель увидит ответ без вопроса.
Вернувшаяся строка остаётся на своём месте по времени; её нельзя просто
добавлять в конец.

## Подготовка

Используйте те же основные библиотеки, что в работе 01. Typer понадобится только
после готовности сборщика. Скопируйте каталоги участников, проектов, сообщения и
редакции.

```text
src/conversation_window/
  models.py
  catalog.py
  window_builder.py
  serializer.py
  cli.py
tests/
  test_catalog.py
  test_window_builder.py
  test_serializer.py
```

## 1. Модели каталога и результата

`models.py`:

```python
from typing import Literal

from pydantic import AwareDatetime, BaseModel, ConfigDict, Field


class Participant(BaseModel):
    model_config = ConfigDict(frozen=True, extra="forbid")
    workspace_id: str
    participant_id: str
    display_name: str
    role: Literal["customer", "employee", "system"]
    organization_id: str | None


class Project(BaseModel):
    model_config = ConfigDict(frozen=True, extra="forbid")
    workspace_id: str
    project_id: str
    conversation_id: str
    organization_id: str
    organization_name: str
    project_name: str
    service_tier: Literal["key", "standard"]
    response_sla_minutes: int = Field(gt=0)
    responsible_role: str


class MessageRevision(BaseModel):
    """Та же проверенная входная запись, что и в работе 01."""
    model_config = ConfigDict(frozen=True, extra="forbid")
    workspace_id: str
    conversation_id: str
    external_id: str
    revision: int = Field(ge=1)
    sender_id: str
    sender_role: Literal["customer", "employee", "system"]
    sent_at: AwareDatetime
    observed_at: AwareDatetime
    text: str = Field(min_length=1)
    reply_to_external_id: str | None = None
    source: Literal["telegram", "max", "import"]


class MessageView(BaseModel):
    model_config = ConfigDict(frozen=True)
    external_id: str
    revision: int
    sender_role: str
    sender_name: str
    sent_at: AwareDatetime
    text: str
    reply_to_external_id: str | None


class ConversationWindow(BaseModel):
    model_config = ConfigDict(frozen=True)
    workspace_id: str
    conversation_id: str
    project_id: str
    analysis_time: AwareDatetime
    messages: tuple[MessageView, ...]
    omitted_before: int
```

`MessageView` намеренно не содержит `observed_at`, внутренний идентификатор
организации и другие данные, которые анализатору не нужны.

## 2. Загрузка каталогов

`catalog.py`:

```python
import json
from pathlib import Path
from typing import TypeVar

from pydantic import BaseModel, TypeAdapter

from .models import Participant, Project

T = TypeVar("T", bound=BaseModel)


def load_list(path: Path, model_type: type[T]) -> tuple[T, ...]:
    raw = json.loads(path.read_text(encoding="utf-8"))
    return TypeAdapter(tuple[model_type, ...]).validate_python(raw)


class Catalog:
    def __init__(self, participants: tuple[Participant, ...],
                 projects: tuple[Project, ...]) -> None:
        # TODO: построить индексы и проверить повторяющиеся ключи.
        raise NotImplementedError

    def participant(self, workspace_id: str, participant_id: str) -> Participant:
        # TODO: не искать участника другой рабочей области.
        raise NotImplementedError

    def project_for_conversation(self, workspace_id: str,
                                 conversation_id: str) -> Project:
        raise NotImplementedError
```

Пример:

```python
participants = load_list(Path("data/fixtures/participants.json"), Participant)
projects = load_list(Path("data/fixtures/projects.json"), Project)
catalog = Catalog(participants, projects)
project = catalog.project_for_conversation("north", "conv-n-001")
assert project.project_id == "borey"
```

## 3. Чистые функции выбора

`window_builder.py`:

```python
from collections.abc import Iterable
from datetime import datetime

from .catalog import Catalog
from .models import ConversationWindow, MessageView
from .models import MessageRevision


def select_latest_known_revisions(
    messages: Iterable[MessageRevision], analysis_time: datetime
) -> tuple[MessageRevision, ...]:
    """Вернуть последнюю известную редакцию каждого внешнего сообщения."""
    # TODO: сначала отфильтровать observed_at, затем выбрать max(revision).
    raise NotImplementedError


def sort_messages(messages: Iterable[MessageRevision]) -> tuple[MessageRevision, ...]:
    return tuple(sorted(
        messages,
        key=lambda item: (item.sent_at, item.external_id, item.revision),
    ))


class WindowBuilder:
    def __init__(self, catalog: Catalog, limit: int = 12) -> None:
        self._catalog = catalog
        self._limit = limit

    def build(self, workspace_id: str, conversation_id: str,
              messages: Iterable[MessageRevision],
              analysis_time: datetime) -> ConversationWindow:
        # TODO: выполнить шаги из основного условия в заданном порядке.
        raise NotImplementedError

    def _to_view(self, workspace_id: str,
                 message: MessageRevision) -> MessageView:
        participant = self._catalog.participant(workspace_id, message.sender_id)
        return MessageView(
            external_id=message.external_id,
            revision=message.revision,
            sender_role=message.sender_role,
            sender_name=participant.display_name,
            sent_at=message.sent_at,
            text=message.text,
            reply_to_external_id=message.reply_to_external_id,
        )
```

Рекомендуемый порядок внутри `build`:

1. выбрать только нужную область и беседу;
2. найти проект;
3. выбрать известные редакции;
4. удалить сообщения из будущего;
5. отсортировать;
6. применить предел;
7. при необходимости добавить исходное сообщение ответа;
8. преобразовать в `MessageView`;
9. собрать `ConversationWindow`.

## 4. Первый содержательный тест

```python
from datetime import datetime


def test_latest_revision_and_future_filter(builder, all_messages) -> None:
    analysis_time = datetime.fromisoformat("2026-09-15T15:00:00+03:00")

    window = builder.build("north", "conv-n-001", all_messages, analysis_time)

    by_id = {message.external_id: message for message in window.messages}
    assert by_id["n-001-006"].revision == 2
    assert "сегодня до 18:00" in by_id["n-001-006"].text
    assert "n-001-013" not in by_id
```

## 5. Стабильная сериализация

`serializer.py`:

```python
from .models import ConversationWindow


def window_to_json(window: ConversationWindow) -> str:
    return window.model_dump_json(indent=2)
```

Вызовите функцию два раза для одного объекта и сравните строки. Если в модель
попало текущее время или случайный идентификатор, сравнение обнаружит проблему.

Сценарий, где ответ ссылается на сообщение за пределами последних 12 строк,
соберите прямо в тесте. Создайте небольшую фабрику `make_message(number,
reply_to=None)`, затем подготовьте 13 обычных сообщений и четырнадцатое сообщение
сотрудника, которое отвечает на первое. В результате первое сообщение должно
вернуться в окно как контекст ответа, `omitted_before` должно учитывать только
действительно скрытые сообщения. Отдельный файл синтетических данных для этого
граничного случая не требуется: фабрика делает условие теста видимым рядом с
проверкой.

## Ручная проверка

```bash
uv run python -m conversation_window.cli build \
  --workspace north \
  --conversation conv-n-001 \
  --analysis-time '2026-09-15T15:00:00+03:00'
```

Проверьте редакцию `n-001-006`, отсутствие `n-001-013`, имена участников и
`omitted_before`.

## Как проверять себя по ходу работы

Проверяйте каждое преобразование отдельно. Для выбора редакций достаточно двух
версий одного сообщения и двух времён анализа. Для сортировки специально
передайте строки в обратном порядке. Для ограничения создайте маленький набор с
пределом 3 — такой тест проще прочитать, чем пример из 13 строк.

`omitted_before` означает число скрытых сообщений до начала итогового окна, а
не просто разницу между размером входного списка и пределом. Старые редакции,
будущие сообщения и записи другой беседы вообще не являются частью выбранной
истории и не должны увеличивать этот счётчик. Зафиксируйте это отдельным тестом,
иначе показатель будет выглядеть правдоподобно, но вводить пользователя в
заблуждение.

Если два запуска с одинаковыми входными данными дают различный JSON, найдите
источник случайности: текущее время, неупорядоченный словарь или случайный
идентификатор. Снимок переписки должен быть воспроизводимым, потому что позднее
именно по нему будут сравниваться ответы анализаторов и LLM.

## Связь со Spine

В дальнейшем каталоги заменятся каноническими сущностями Spine, а сбор контекста
может выполнять платформенная служба. Контракт `ConversationWindow` и явное
время анализа позволяют заменить источник без изменения анализаторов.
