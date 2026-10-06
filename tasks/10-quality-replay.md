# Работа 10. Повторный прогон и измерение качества

[Основное условие](../responseflow-practice.md#работа-10-повторный-прогон-и-измерение-качества)

## Чему учится разработчик

- сопоставлять эталон и результат по устойчивому ключу;
- считать точность и полноту без готовой библиотеки;
- отдельно оценивать правильность оснований;
- завершать команду ошибкой при нарушении порога;
- сравнивать правила, зафиксированный ответ и настоящий LLM-прогон одинаковым
  способом.

## Как устроена эта работа

Один удачный пример не доказывает качество анализатора. Набор оценки содержит
несколько независимых случаев с ожидаемыми ситуациями и точными основаниями.
Команда сопоставляет зафиксированный результат с эталоном и показывает, что
нашлось верно, что появилось лишним и что было пропущено.

Сопоставление выполняется по типу и устойчивому ключу предмета внутри одного
примера. Текст описания не подходит: модель может перефразировать его, не меняя
смысла, или написать похожий текст для другой проблемы. Основания оцениваются
после совпадения ключа и отвечают на отдельный вопрос — действительно ли вывод
подтверждён нужными сообщениями.

Точность и полнота показывают разные виды ошибок. Низкая точность означает много
лишних срабатываний; низкая полнота — много пропусков. Гармоническое среднее
снижается, если плох хотя бы один из показателей. Доля правильных оснований не
должна влиять на число совпавших ситуаций: неверная ссылка ухудшает собственную
метрику, но не превращает правильный ключ в другой тип ошибки.

Предсказания сохраняются в файл до расчёта метрик. Поэтому один дорогой ручной
LLM-прогон можно оценивать многократно, менять формат отчёта и проверять новые
пороги без повторного сетевого вызова. Метаданные прогона помогают понять,
какая модель и инструкция создали результат.

## Раскладка

```text
src/quality_replay/
  models.py
  dataset.py
  matcher.py
  metrics.py
  report.py
  cli.py
tests/
  test_matcher.py
  test_metrics.py
  test_quality_gate.py
```

## 1. Модели

`models.py`:

```python
from pydantic import AwareDatetime, BaseModel, ConfigDict, Field


class ExpectedFinding(BaseModel):
    model_config = ConfigDict(frozen=True, extra="forbid")
    type: str
    subject_key: str
    required_evidence_message_ids: tuple[str, ...]


class PredictedFinding(BaseModel):
    model_config = ConfigDict(frozen=True, extra="forbid")
    type: str
    subject_key: str
    evidence_message_ids: tuple[str, ...]


class DatasetCase(BaseModel):
    model_config = ConfigDict(frozen=True, extra="forbid")
    case_id: str
    workspace_id: str
    conversation_id: str
    analysis_time: AwareDatetime
    expected: tuple[ExpectedFinding, ...]


class PredictionCase(BaseModel):
    model_config = ConfigDict(frozen=True, extra="forbid")
    case_id: str
    findings: tuple[PredictedFinding, ...]


class Counters(BaseModel):
    true_positive: int = Field(ge=0)
    false_positive: int = Field(ge=0)
    false_negative: int = Field(ge=0)
    correct_evidence: int = Field(ge=0)
    duplicates: int = Field(ge=0)


class Metrics(BaseModel):
    precision: float
    recall: float
    f1: float
    evidence_accuracy: float


class MetricThresholds(BaseModel):
    model_config = ConfigDict(frozen=True, extra="forbid")
    precision: float = Field(ge=0, le=1)
    recall: float = Field(ge=0, le=1)
    evidence_accuracy: float = Field(ge=0, le=1)


class QualityGate(BaseModel):
    model_config = ConfigDict(frozen=True, extra="forbid")
    default: MetricThresholds
    by_type: dict[str, MetricThresholds]

    def for_type(self, finding_type: str) -> MetricThresholds:
        return self.by_type.get(finding_type, self.default)
```

## 2. Чтение набора

`dataset.py`:

```python
import json
from collections.abc import Iterator
from pathlib import Path
from typing import TypeVar

from pydantic import BaseModel, ValidationError

T = TypeVar("T", bound=BaseModel)


def read_jsonl(path: Path, model_type: type[T]) -> Iterator[T]:
    with path.open(encoding="utf-8") as stream:
        for line_number, line in enumerate(stream, 1):
            try:
                yield model_type.model_validate(json.loads(line))
            except (json.JSONDecodeError, ValidationError) as error:
                raise ValueError(f"{path}:{line_number}: {error}") from error
```

Загрузите `dataset.jsonl` как `DatasetCase`, а файл результатов как
`PredictionCase`. После чтения проверьте уникальность `case_id`, затем убедитесь,
что множество идентификаторов в результатах в точности совпадает с множеством
идентификаторов набора. Так не останутся незамеченными ни неизвестные, ни
пропущенные примеры.

## 3. Сопоставление

`matcher.py`:

```python
from collections import Counter

from .models import Counters, ExpectedFinding, PredictedFinding

FindingKey = tuple[str, str]


def expected_key(item: ExpectedFinding) -> FindingKey:
    return item.type, item.subject_key


def predicted_key(item: PredictedFinding) -> FindingKey:
    return item.type, item.subject_key


def match_case(expected: tuple[ExpectedFinding, ...],
               predicted: tuple[PredictedFinding, ...]) -> Counters:
    expected_by_key = {expected_key(item): item for item in expected}
    predicted_counts = Counter(predicted_key(item) for item in predicted)
    predicted_by_key = {predicted_key(item): item for item in predicted}

    expected_keys = set(expected_by_key)
    predicted_keys = set(predicted_by_key)
    matched = expected_keys & predicted_keys

    correct_evidence = 0
    for key in matched:
        required = set(expected_by_key[key].required_evidence_message_ids)
        actual = set(predicted_by_key[key].evidence_message_ids)
        if required <= actual:
            correct_evidence += 1

    # TODO: рассчитать три основных счётчика и лишние повторы.
    raise NotImplementedError
```

Дубликаты считаются отдельно. Для основных показателей один ключ участвует не
более одного раза.

## 4. Формулы

`metrics.py`:

```python
from .models import Counters, Metrics


def safe_divide(numerator: int, denominator: int) -> float:
    return 0.0 if denominator == 0 else numerator / denominator


def calculate_metrics(counters: Counters) -> Metrics:
    precision = safe_divide(
        counters.true_positive,
        counters.true_positive + counters.false_positive,
    )
    recall = safe_divide(
        counters.true_positive,
        counters.true_positive + counters.false_negative,
    )
    f1 = safe_divide(2 * precision * recall, precision + recall)
    evidence_accuracy = safe_divide(
        counters.correct_evidence, counters.true_positive
    )
    return Metrics(
        precision=precision,
        recall=recall,
        f1=f1,
        evidence_accuracy=evidence_accuracy,
    )
```

Не округляйте здесь. Округление принадлежит только формированию отчёта.

## 5. Первый тест математики

```python
def test_one_match_one_extra_one_missing() -> None:
    counters = Counters(
        true_positive=1,
        false_positive=1,
        false_negative=1,
        correct_evidence=1,
        duplicates=0,
    )
    metrics = calculate_metrics(counters)
    assert metrics.precision == 0.5
    assert metrics.recall == 0.5
    assert metrics.f1 == 0.5
    assert metrics.evidence_accuracy == 1.0
```

## 6. Отчёт и порог

`report.py` должен получать готовый объект результата. Одна функция возвращает
JSON, другая — таблицу Markdown. Обе используют одни числа.

Проверка порога возвращает все нарушения:

```python
def check_thresholds(metrics_by_type, gate) -> tuple[str, ...]:
    violations: list[str] = []
    for finding_type, metrics in metrics_by_type.items():
        threshold = gate.for_type(finding_type)
        if metrics.precision < threshold.precision:
            violations.append(f"{finding_type}: недостаточная точность")
        # TODO: полнота и основания.
    return tuple(violations)
```

В `cli.py` после печати нарушений выполните `raise typer.Exit(code=1)`.

Добавьте отдельную команду `run-llm`. Она получает путь для результата, точный
идентификатор модели и явный флаг `--allow-network`. Без флага команда должна
завершиться до создания клиента VseLLM. Сохранённый JSONL затем передаётся
обычной команде `check`, поэтому повторный расчёт метрик не расходует токены.

В метаданных ручного прогона сохраните:

- `provider="vsellm"`;
- точный `model`;
- версию системной инструкции;
- время анализа;
- число входных и выходных токенов;
- состояние вызова и безопасное описание ошибки.

Ключ API и полный текст запроса сохранять запрещено.

## Порядок реализации

1. Формулы на вручную созданных счётчиках.
2. Ключ ситуации.
3. Один совпавший ключ.
4. Лишний и пропущенный ключи.
5. Дубликаты.
6. Основания.
7. Загрузка файлов.
8. Группировка по типам и общий итог.
9. Отчёты.
10. Порог, версия 1 и версия 2.
11. Ручной VseLLM-прогон в отдельный файл и его оценка без сети.

## Как проверять себя по ходу работы

Начните с вычисления метрик по вручную созданным счётчикам. После этого
проверьте сопоставление одного примера и лишь затем суммирование по всему набору.
Так арифметическая ошибка не смешается с ошибкой чтения JSONL.

Обязательно создайте случаи с пустыми ожиданиями, пустыми предсказаниями и
полностью пустым примером. Заранее договоритесь, что деление на ноль даёт `0.0`,
и используйте одну функцию `safe_divide` для всех показателей. Не округляйте
промежуточные значения: округление выполняется только при выводе.

Дубликат не должен дважды увеличивать число верных находок. Один экземпляр
участвует в основных метриках, остальные попадают в отдельный счётчик. Также
проверяйте совпадение множеств `case_id`: лишний или потерянный пример нельзя
молча проигнорировать.

После реализации запустите версии 1 и 2. Недостаточно увидеть только код выхода:
прочитайте отчёт и объясните, какой конкретно пропуск, лишнее срабатывание или
основание изменило каждую метрику. Это и есть основной учебный результат работы.

## Связь со Spine

Набор и метрики можно перенести в средства оценки Spine. Составной ключ,
размеченные основания и отчёт по отдельным типам сохраняют предметную ценность.
