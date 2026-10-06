# Работа 08. Приоритет и следующее действие

[Основное условие](../responseflow-practice.md#работа-08-приоритет-и-следующее-действие)

## Чему учится разработчик

- загружать и проверять настройки;
- строить расчёт из маленьких независимых правил;
- объяснять каждую надбавку;
- разделять числовую оценку, уровень и рекомендацию.

## Как устроена эта работа

Приоритет не должен быть скрытой догадкой. Он начинается с базовой оценки типа
ситуации, затем получает независимые надбавки за известные признаки. Каждая
надбавка возвращает не только число, но и понятную причину. Менеджер видит тот
же ход расчёта, который проверяется в тесте.

Настройки вынесены в файл, чтобы менять значения без переписывания функций.
Однако файл нельзя считать правильным только потому, что он читается как JSON.
До первой обработки нужно убедиться, что присутствуют нужные типы, границы
уровней возрастают, сроки положительны, а для каждого типа есть действие.
Ошибка настроек останавливает запуск целиком: это не ошибка отдельной карточки.

Числовая оценка, уровень и действие — разные результаты. Оценка может быть 85,
уровень — `critical`, а действие берётся из таблицы для типа `delay` и
дополняется требованием уведомить руководителя. Не подменяйте одно другим и не
зашивайте готовые фразы внутрь расчёта баллов.

Порядок правил фиксирован кортежем `BONUS_RULES`. Сумма от порядка не зависит,
но список объяснений зависит. Постоянный порядок делает отчёт воспроизводимым и
не создаёт бессмысленных изменений при сравнении результатов.

## Раскладка

```text
src/priority_and_action/
  models.py
  policy_loader.py
  priority_policy.py
  action_rules.py
  explainer.py
  cli.py
tests/
  test_policy_loader.py
  test_priority_policy.py
  test_actions.py
```

## 1. Модели настроек

`models.py`:

```python
from typing import Literal

from pydantic import AwareDatetime, BaseModel, ConfigDict, Field, model_validator

Priority = Literal["low", "medium", "high", "critical"]


class FindingCandidate(BaseModel):
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


class ActionRule(BaseModel):
    role: str = Field(min_length=1)
    text: str = Field(min_length=10)


class PriorityPolicyConfig(BaseModel):
    model_config = ConfigDict(extra="forbid")
    base_scores: dict[str, int]
    bonuses: dict[str, int]
    levels: dict[str, int]
    action_due_minutes: dict[Priority, int]
    actions: dict[str, ActionRule]

    @model_validator(mode="after")
    def validate_boundaries(self) -> "PriorityPolicyConfig":
        # TODO: проверить порядок low < medium < high < critical.
        return self


class PrioritizedFinding(BaseModel):
    finding_key: str
    priority: Priority
    score: int = Field(ge=0, le=100)
    reasons: tuple[str, ...]
    recommended_action: str
    responsible_role: str
    action_due_at: AwareDatetime
```

## 2. Загрузка настроек

`policy_loader.py`:

```python
import json
from pathlib import Path

from .models import PriorityPolicyConfig


def load_policy(path: Path) -> PriorityPolicyConfig:
    raw = json.loads(path.read_text(encoding="utf-8"))
    return PriorityPolicyConfig.model_validate(raw)
```

Валидация должна завершиться до обработки первой карточки. Неверные настройки —
ошибка запуска, а не отдельной ситуации.

## 3. Независимые надбавки

`priority_policy.py`:

```python
from dataclasses import dataclass

from .models import FindingCandidate, PriorityPolicyConfig


@dataclass(frozen=True)
class ScoreChange:
    points: int
    reason: str


def key_client_bonus(candidate: FindingCandidate,
                     config: PriorityPolicyConfig) -> ScoreChange | None:
    if "key_client" not in candidate.flags:
        return None
    return ScoreChange(config.bonuses["key_client"], "Ключевой клиент.")


def overdue_bonus(candidate: FindingCandidate,
                  config: PriorityPolicyConfig) -> ScoreChange | None:
    # TODO: вернуть надбавку только при нужном признаке.
    raise NotImplementedError


BONUS_RULES = (
    key_client_bonus,
    overdue_bonus,
    # TODO: добавить угрозу остановки и повторное обнаружение.
)
```

Кортеж `BONUS_RULES` задаёт постоянный порядок причин.

## 4. Расчёт

```python
from datetime import timedelta


def priority_from_score(score: int, config: PriorityPolicyConfig) -> Priority:
    if score >= config.levels["critical_min"]:
        return "critical"
    if score >= config.levels["high_min"]:
        return "high"
    if score >= config.levels["medium_min"]:
        return "medium"
    return "low"


class PriorityService:
    def __init__(self, config: PriorityPolicyConfig) -> None:
        self._config = config

    def prioritize(self, candidate: FindingCandidate) -> PrioritizedFinding:
        score = self._config.base_scores[candidate.type]
        reasons = [f"Базовая оценка типа {candidate.type}: {score}."]
        for rule in BONUS_RULES:
            change = rule(candidate, self._config)
            if change is not None:
                score += change.points
                reasons.append(change.reason)
        score = min(100, max(0, score))
        priority = priority_from_score(score, self._config)
        action = self._config.actions[candidate.type]
        text = action.text
        if priority == "critical":
            text += " Немедленно уведомить руководителя."
        # TODO: рассчитать action_due_at от candidate.detected_at.
        raise NotImplementedError
```

## 5. Граничные тесты

```python
import pytest


@pytest.mark.parametrize(
    ("score", "expected"),
    [(0, "low"), (29, "low"), (30, "medium"),
     (49, "medium"), (50, "high"), (80, "critical")],
)
def test_priority_boundaries(config, score, expected) -> None:
    assert priority_from_score(score, config) == expected
```

В отдельном тесте рассчитайте просроченную смету вручную и сравните балл, причины
и срок действия.

Для проверки надбавки за повторное обнаружение загрузите соответствующую строку
из `expected/candidate-updates.jsonl` и добавьте ей признак
`repeated_detection` через `model_copy`. Сам файл кандидатов намеренно не хранит
состояние карточки и счётчик повторов: эти сведения появляются в работе 07.

## Пример использования

```python
config = load_policy(Path("data/fixtures/priority-policy.json"))
service = PriorityService(config)
result = service.prioritize(candidate)
print(result.priority, result.score)
for reason in result.reasons:
    print("-", reason)
```

## Порядок реализации

1. Модели настроек.
2. Проверка возрастающих границ.
3. Преобразование балла в уровень.
4. Базовая оценка без надбавок.
5. По одной надбавке и тесту.
6. Ограничение 0–100.
7. Действие и роль.
8. Срок от фиксированного времени.
9. Сортировка очереди.

## Как проверять себя по ходу работы

Сначала проверьте преобразование готового числа в уровень без кандидатов и
настроек из файла. Для каждой границы используйте число перед ней, саму границу
и число после неё. Затем добавьте базовую оценку и подключайте по одной
надбавке, каждый раз проверяя и число, и текст причины.

Создайте отдельный тест, где сумма превышает 100, и убедитесь, что ограничение
применяется после всех надбавок. Аналогично полезно проверить нижнюю границу,
даже если текущие настройки не содержат отрицательных значений: модель явно
обещает диапазон 0–100.

Срок следующего действия рассчитывайте от `candidate.detected_at`, а не от
текущего времени компьютера и не от старого `due_at`. Тогда повторный прогон
синтетических данных остаётся воспроизводимым. При сортировке используйте
уровень, срок и устойчивый ключ, чтобы две карточки с одинаковой оценкой всегда
располагались одинаково.

Если тип отсутствует в таблице, завершайте обработку понятной ошибкой. Нельзя
подставлять случайное действие по умолчанию: менеджер может принять его за
утверждённое предметное правило.

## Связь со Spine

Расчёт может стать платформенной политикой или кодовым шагом процесса. Главное —
сохранить версию настроек, числовой результат и объяснение сработавших правил.
