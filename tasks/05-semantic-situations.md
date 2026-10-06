# Работа 05. Претензии и запросы документов

[Основное условие](../responseflow-practice.md#работа-05-претензии-и-запросы-документов)

## Чему учится разработчик

- скрывать внешнюю модель за собственным договором;
- применять программную заглушку;
- проверять структурированный ответ;
- не терять хорошие результаты из-за одной ошибочной записи.

## Подготовка и библиотеки

Pydantic проверяет ответы анализатора. Typer нужен для команды. Библиотека для
языковой модели не устанавливается: работа должна полностью проходить по
`fake-results.json`.

```text
src/semantic_situations/
  models.py
  analyzer.py
  fake_analyzer.py
  normalizer.py
  service.py
  cli.py
tests/
  test_fake_analyzer.py
  test_normalizer.py
  test_service.py
```

## 1. Модель и договор

`models.py`:

```python
from typing import Literal

from pydantic import AwareDatetime, BaseModel, ConfigDict, Field


class MessageView(BaseModel):
    model_config = ConfigDict(frozen=True)
    external_id: str
    sender_role: Literal["customer", "employee", "system"]
    sent_at: AwareDatetime
    text: str


class ConversationWindow(BaseModel):
    model_config = ConfigDict(frozen=True)
    workspace_id: str
    conversation_id: str
    project_id: str
    analysis_time: AwareDatetime
    messages: tuple[MessageView, ...]


class SemanticResult(BaseModel):
    model_config = ConfigDict(frozen=True, extra="forbid")
    type: Literal["complaint", "document_request", "manager_attention"]
    confidence: float = Field(ge=0, le=1)
    summary: str = Field(min_length=10, max_length=240)
    recommended_action: str = Field(min_length=10, max_length=300)
    evidence_message_ids: tuple[str, ...]
    subject_key: str = Field(min_length=1)


class AnalysisReport(BaseModel):
    accepted: tuple[SemanticResult, ...]
    below_threshold: int
    invalid: int
    errors: tuple[str, ...]
```

`analyzer.py`:

```python
from typing import Protocol

from .models import ConversationWindow, SemanticResult


class SemanticAnalyzer(Protocol):
    def analyze(self, window: ConversationWindow) -> tuple[SemanticResult, ...]: ...
```

Служба будет зависеть от `SemanticAnalyzer`, а не от имени конкретной модели.

## 2. Заглушка

`fake_analyzer.py`:

```python
import json
from pathlib import Path

from pydantic import BaseModel, TypeAdapter

from .models import ConversationWindow, SemanticResult


class FixtureEntry(BaseModel):
    workspace_id: str
    conversation_id: str
    window_checksum: str
    items: tuple[SemanticResult, ...]


class FakeSemanticAnalyzer:
    def __init__(self, fixture_path: Path) -> None:
        raw = json.loads(fixture_path.read_text(encoding="utf-8"))
        entries = TypeAdapter(tuple[FixtureEntry, ...]).validate_python(raw["results"])
        self._items = {
            (entry.workspace_id, entry.conversation_id): entry.items
            for entry in entries
        }
        if len(self._items) != len(entries):
            raise ValueError("В заглушке повторяется рабочая область и беседа")

    def analyze(self, window: ConversationWindow) -> tuple[SemanticResult, ...]:
        return tuple(self._items.get(
            (window.workspace_id, window.conversation_id), ()
        ))
```

Пример замены реализации в тесте:

```python
class OneResultAnalyzer:
    def __init__(self, result: SemanticResult) -> None:
        self._result = result

    def analyze(self, window: ConversationWindow) -> tuple[SemanticResult, ...]:
        return (self._result,)
```

Для небольшого теста удобнее `OneResultAnalyzer`, а для полного набора — готовая
файловая заглушка.

## 3. Нормализация

`normalizer.py`:

```python
from .models import SemanticResult


class UnknownEvidenceError(ValueError):
    pass


def unique_preserving_order(values: tuple[str, ...]) -> tuple[str, ...]:
    return tuple(dict.fromkeys(values))


def validate_and_normalize(
    result: SemanticResult, available_message_ids: set[str]
) -> SemanticResult:
    evidence = unique_preserving_order(result.evidence_message_ids)
    if not evidence:
        raise UnknownEvidenceError("Анализатор не указал ни одного основания")
    missing = [item for item in evidence if item not in available_message_ids]
    if missing:
        raise UnknownEvidenceError(f"В окне нет оснований: {missing}")
    return result.model_copy(update={"evidence_message_ids": evidence})
```

## 4. Служба независимой обработки

`service.py`:

```python
from .analyzer import SemanticAnalyzer
from .models import AnalysisReport, ConversationWindow, SemanticResult
from .normalizer import UnknownEvidenceError, validate_and_normalize


class SemanticSituationService:
    def __init__(self, analyzer: SemanticAnalyzer, threshold: float = 0.60) -> None:
        self._analyzer = analyzer
        self._threshold = threshold

    def analyze(self, window: ConversationWindow) -> AnalysisReport:
        accepted: list[SemanticResult] = []
        errors: list[str] = []
        below_threshold = 0
        invalid = 0
        available_ids = {message.external_id for message in window.messages}

        for result in self._analyzer.analyze(window):
            if result.confidence < self._threshold:
                below_threshold += 1
                continue
            try:
                accepted.append(validate_and_normalize(result, available_ids))
            except UnknownEvidenceError as error:
                invalid += 1
                errors.append(str(error))
        # TODO: устойчиво отсортировать accepted и собрать AnalysisReport.
        raise NotImplementedError
```

## 5. Первый тест ошибки одного элемента

```python
def test_invalid_evidence_does_not_remove_valid_result(window, valid_result) -> None:
    invalid = valid_result.model_copy(update={
        "subject_key": "broken",
        "evidence_message_ids": ("missing-message",),
    })
    service = SemanticSituationService(ListAnalyzer((invalid, valid_result)))

    report = service.analyze(window)

    assert report.invalid == 1
    assert report.accepted == (valid_result,)
```

## Порядок реализации

1. Строгая модель результата.
2. Маленький подставной анализатор в тесте.
3. Удаление повторов оснований.
4. Проверка отсутствующего основания.
5. Порог уверенности.
6. Независимая обработка результатов.
7. Файловая заглушка.
8. Сравнение с эталонным JSONL.

## Связь со Spine

Позже заглушку можно заменить версионируемой способностью Spine. Служба
проверки оснований и порога останется прикладной защитой ActionFlow.
