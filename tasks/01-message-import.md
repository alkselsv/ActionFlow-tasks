# Работа 01. Надёжный импорт сообщений

[Основное условие](../responseflow-practice.md#работа-01-надёжный-импорт-сообщений)

## Чему учится разработчик

- описывать входные данные строгой моделью;
- читать JSONL построчно;
- отделять хранилище от предметного решения;
- различать новую запись, точный повтор и конфликт;
- писать командную оболочку над готовой службой.

## Подготовка

```bash
uv init --package --name message-import --python 3.12
uv add 'pydantic>=2.8,<3' 'typer>=0.12,<1'
uv add --dev 'pytest>=8,<9' 'ruff>=0.6,<1'
mkdir -p tests data/fixtures
```

Pydantic проверяет поля входной строки. Typer создаёт команды без ручного
разбора `sys.argv`. pytest запускает проверки, Ruff находит простые ошибки и
неиспользуемые импорты.

Скопируйте `messages/messages.jsonl`, `messages/revisions.jsonl` и `invalid/`.

## Раскладка

```text
src/message_import/
  __init__.py
  models.py
  errors.py
  reader.py
  repository.py
  service.py
  cli.py
tests/
  test_models.py
  test_service.py
  test_jsonl_repository.py
```

## 1. Модель сообщения

`models.py`:

```python
from datetime import datetime
from typing import Literal

from pydantic import BaseModel, ConfigDict, Field, field_validator


class MessageRevision(BaseModel):
    model_config = ConfigDict(frozen=True, extra="forbid")

    workspace_id: str = Field(min_length=1)
    conversation_id: str = Field(min_length=1)
    external_id: str = Field(min_length=1)
    revision: int = Field(ge=1)
    sender_id: str = Field(min_length=1)
    sender_role: Literal["customer", "employee", "system"]
    sent_at: datetime
    observed_at: datetime
    text: str = Field(min_length=1)
    reply_to_external_id: str | None = None
    source: Literal["telegram", "max", "import"]

    @field_validator("sent_at", "observed_at")
    @classmethod
    def require_timezone(cls, value: datetime) -> datetime:
        # TODO: отклонить дату без часового пояса.
        return value

    @field_validator("text")
    @classmethod
    def reject_blank_text(cls, value: str) -> str:
        # TODO: отклонить строку только из пробелов, не меняя исходный текст.
        return value


class ImportReport(BaseModel):
    read: int = 0
    created: int = 0
    unchanged: int = 0
    rejected: int = 0
```

Модель неизменяема, потому что редакция является наблюдавшимся фактом. Если
нужно исправить сообщение, создаётся новая редакция.

Первый тест:

```python
from datetime import datetime

import pytest
from pydantic import ValidationError

from message_import.models import MessageRevision


def test_datetime_without_timezone_is_rejected(valid_message: dict) -> None:
    valid_message["sent_at"] = datetime(2026, 9, 15, 10, 0)

    with pytest.raises(ValidationError):
        MessageRevision.model_validate(valid_message)
```

Повторяющийся словарь `valid_message` удобно оформить как фикстуру в
`tests/conftest.py`.

## 2. Ошибки и чтение файла

`errors.py`:

```python
class InputLineError(ValueError):
    """Строка входного файла не является допустимым сообщением."""


class MessageConflictError(ValueError):
    """Одинаковый ключ редакции связан с различным содержимым."""
```

`reader.py`:

```python
import json
from collections.abc import Iterator
from pathlib import Path

from pydantic import ValidationError

from .errors import InputLineError
from .models import MessageRevision


def read_messages(path: Path) -> Iterator[MessageRevision]:
    with path.open("r", encoding="utf-8") as stream:
        for line_number, line in enumerate(stream, start=1):
            try:
                raw = json.loads(line)
                yield MessageRevision.model_validate(raw)
            except (json.JSONDecodeError, ValidationError) as error:
                # TODO: добавить путь, номер строки и исходную причину.
                raise InputLineError(...) from error
```

Пример использования:

```python
for message in read_messages(Path("data/fixtures/messages.jsonl")):
    print(message.external_id, message.revision)
```

Не возвращайте список: генератор позволяет обрабатывать большой файл по одной
строке.

## 3. Договор хранилища

`repository.py`:

```python
from typing import Protocol

from .models import MessageRevision

MessageKey = tuple[str, str, str, int]


def message_key(message: MessageRevision) -> MessageKey:
    return (
        message.workspace_id,
        message.conversation_id,
        message.external_id,
        message.revision,
    )


class MessageRepository(Protocol):
    def get(self, key: MessageKey) -> MessageRevision | None: ...
    def add(self, message: MessageRevision) -> None: ...
    def history(
        self, workspace_id: str, conversation_id: str, external_id: str
    ) -> tuple[MessageRevision, ...]: ...


class InMemoryMessageRepository:
    def __init__(self) -> None:
        self._items: dict[MessageKey, MessageRevision] = {}

    def get(self, key: MessageKey) -> MessageRevision | None:
        return self._items.get(key)

    def add(self, message: MessageRevision) -> None:
        # TODO: сохранить по составному ключу.
        raise NotImplementedError

    def history(self, workspace_id: str, conversation_id: str,
                external_id: str) -> tuple[MessageRevision, ...]:
        # TODO: выбрать нужные записи и отсортировать по номеру редакции.
        raise NotImplementedError
```

`Protocol` позволяет одной службе работать и со словарём, и с файлом.

## 4. Служба импорта

`service.py`:

```python
from pathlib import Path

from .errors import MessageConflictError
from .models import ImportReport, MessageRevision
from .reader import read_messages
from .repository import MessageRepository, message_key


class MessageImportService:
    def __init__(self, repository: MessageRepository) -> None:
        self._repository = repository

    def import_one(self, message: MessageRevision) -> str:
        existing = self._repository.get(message_key(message))
        if existing is None:
            self._repository.add(message)
            return "created"
        if existing == message:
            return "unchanged"
        raise MessageConflictError(
            f"Конфликт редакции {message.external_id}/{message.revision}"
        )

    def import_file(self, path: Path) -> ImportReport:
        # TODO: вызвать read_messages, import_one и собрать счётчики.
        raise NotImplementedError
```

Пример проверки службы:

```python
repository = InMemoryMessageRepository()
service = MessageImportService(repository)

assert service.import_one(message) == "created"
assert service.import_one(message) == "unchanged"
assert len(repository.history("north", "conv-n-001", "n-001-006")) == 1
```

## 5. Файловое хранилище и команда

Сначала доведите до готовности вариант в памяти. Затем добавьте
`JsonlMessageRepository`, который при создании читает имеющийся файл, а при
`add` дописывает одну строку. Предметная служба при этом не меняется.

`cli.py`:

```python
from pathlib import Path

import typer

from .repository import JsonlMessageRepository
from .service import MessageImportService

app = typer.Typer()


@app.command("import-file")
def import_file(
    source: Path,
    storage: Path = typer.Option(..., "--storage", help="Путь к JSONL-хранилищу"),
) -> None:
    service = MessageImportService(JsonlMessageRepository(storage))
    report = service.import_file(source)
    typer.echo(report.model_dump_json(indent=2))


if __name__ == "__main__":
    app()
```

Запуск:

```bash
uv run python -m message_import.cli import-file \
  data/fixtures/messages.jsonl --storage data/messages-store.jsonl
```

## Порядок завершения

1. Модель и её тесты.
2. Чтение одной и повреждённой строки.
3. Хранилище в памяти.
4. Импорт одной записи и повтора.
5. Импорт файла и счётчики.
6. Файловое хранилище.
7. Команды `import-file` и `history`.
8. Полный ручной сценарий из основного условия.

## Связь со Spine

При будущем переносе файловый источник заменится подключением канала, а
хранилище — платформенным. Модель редакции, составной внешний идентификатор и
правило безопасного повтора должны сохраниться.
