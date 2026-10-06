# Работа 05. Претензии и запросы документов

[Основное условие](../responseflow-practice.md#работа-05-претензии-и-запросы-документов)

## Чему учится разработчик

- скрывать внешнюю модель за собственным договором;
- применять программную заглушку;
- проверять структурированный ответ;
- не терять хорошие результаты из-за одной ошибочной записи.

## Как устроена эта работа

Эта работа вводит смысловой анализ, но пока не подключает сеть. Вместо внешней
модели используется заглушка с заранее подготовленными ответами. Такой подход
позволяет изучить договор анализатора, проверку результата и обработку ошибок
без зависимости от ключа, стоимости и случайности ответа LLM.

`SemanticAnalyzer` — это обещание о поведении: любой анализатор получает окно и
возвращает набор результатов одного формата. Службе не важно, прочитал ли
результат файл, вычислил ли его набор правил или вернула языковая модель. Это
важная граница: замена способа анализа не должна менять проверку оснований и
порог уверенности.

Ответ анализатора ещё не считается истиной. Служба убеждается, что каждое
основание действительно присутствует в окне, удаляет повторяющиеся
идентификаторы и отбрасывает результаты ниже порога. Ошибка одной смысловой
ситуации не должна уничтожать другую правильную ситуацию из того же ответа.
Поэтому отчёт отдельно считает принятые, слабые и недопустимые элементы.

Заглушка выбирает ответ по рабочей области и беседе. Контрольная сумма окна
может использоваться как дополнительная защита: если подготовленные сообщения
изменились, старый зафиксированный ответ уже нельзя считать подходящим. Для
первой реализации разрешено только прочитать это поле, затем добавьте явную
проверку и понятную ошибку.

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

## Разбор подходов, классов и методов

`SemanticResult` описывает одну предполагаемую ситуацию. Ограниченный
`Literal` для `type` не даёт анализатору незаметно придумать новый тип.
Ограничения длины для `summary` и `recommended_action` защищают очередь от
пустых и чрезмерно длинных текстов. `confidence` показывает уверенность
источника, но не заменяет проверку оснований.

`AnalysisReport` нужен потому, что результат работы службы шире списка принятых
ситуаций. Поля `below_threshold` и `invalid` позволяют различить слабый, но
корректный ответ и ответ с нарушением договора. `errors` хранит безопасные
пояснения для недопустимых элементов. Так вызывающий код видит частичный успех,
а не только исключение всего запуска.

`SemanticAnalyzer` объявлен как `Protocol`. Это означает структурное
соответствие: классу не нужно наследоваться от протокола, достаточно иметь
метод `analyze` с подходящей сигнатурой. Такой договор облегчает маленькие
заглушки в тестах и не заставляет предметный код знать конкретный поставщик.

`FixtureEntry` описывает одну запись готовой заглушки. `FakeSemanticAnalyzer`
читает файл один раз при создании и строит словарь по паре рабочей области и
беседы. Метод `analyze(window)` после этого выполняет простой поиск и возвращает
пустой кортеж, если ответа нет. Важно не читать файл при каждом вызове: это
смешало бы анализ с вводом-выводом и замедлило проверки.

`OneResultAnalyzer` показывает минимальную реализацию протокола. Она полезна,
когда тест посвящён одному правилу службы и файловая заглушка только отвлекает.
По такому же принципу можно создать `ListAnalyzer`, возвращающий несколько
элементов в заданном порядке.

`unique_preserving_order(values)` удаляет повторы через словарь и сохраняет
первое появление каждого идентификатора. Обычное множество здесь не подходит:
оно потеряет порядок, а отчёты станут нестабильными. Функция не проверяет
существование сообщений — это отдельная обязанность.

`validate_and_normalize(result, available_message_ids)` сначала нормализует
основания, затем требует хотя бы одно и проверяет каждое по множеству доступных
идентификаторов. Она возвращает копию модели с обновлённым полем, а не изменяет
исходный замороженный объект. `UnknownEvidenceError` позволяет службе отличить
предметную ошибку основания от неожиданного программного сбоя.

`SemanticSituationService` получает анализатор через конструктор. Параметр
`threshold` также передаётся снаружи, поэтому тест может явно проверить границу.
Метод `analyze(window)` один раз строит множество доступных сообщений, затем
обрабатывает каждый результат независимо. После цикла принятые элементы нужно
отсортировать по стабильному ключу и собрать `AnalysisReport`.

Последовательность вызовов выглядит так:

```text
ConversationWindow → SemanticAnalyzer.analyze
                   → SemanticResult[]
                   → порог уверенности
                   → validate_and_normalize
                   → AnalysisReport
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

## Как проверять себя по ходу работы

Не начинайте с полного файла. Создайте в тесте один `SemanticResult`, передайте
его через маленький `OneResultAnalyzer` и убедитесь, что служба возвращает тот
же объект. Затем по одному добавьте низкую уверенность, повтор основания и
неизвестное основание. Только после этого подключайте файловую заглушку.

Счётчики отчёта должны сходиться: число возвращённых анализатором элементов
равно сумме принятых, отброшенных по порогу и недопустимых. Зафиксируйте это в
тесте. Текст ошибки должен содержать неизвестный идентификатор, но не весь текст
переписки.

Частая ошибка — проверять порог внутри заглушки. Тогда другая реализация
анализатора может вести себя иначе. Заглушка имитирует внешний источник, а все
обязательные защитные правила находятся в службе и одинаково применяются к
любому источнику.

## Связь со Spine

Позже заглушку можно заменить версионируемой способностью Spine. Служба
проверки оснований и порога останется прикладной защитой ActionFlow.
