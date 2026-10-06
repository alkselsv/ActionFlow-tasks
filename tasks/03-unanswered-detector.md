# Работа 03. Обращения без ответа

[Основное условие](../responseflow-practice.md#работа-03-обнаружение-обращения-без-ответа)

## Чему учится разработчик

- раскладывать одно правило на маленькие чистые функции;
- считать рабочее время;
- передавать время анализа явно;
- создавать объяснимый кандидат со ссылкой на основание.

## Как устроена эта работа

Поиск обращения без ответа кажется простой проверкой текста, но в нём есть три
независимых вопроса. Сначала нужно понять, является ли сообщение обращением.
Затем определить, последовал ли ответ сотрудника. И только после этого решить,
истёк ли допустимый срок. Если соединить всё в одном цикле, ошибка в расчёте
времени будет выглядеть как ошибка распознавания текста.

Правила фраз намеренно ограничены. Это не попытка понять любой русский текст, а
прозрачный учебный механизм. Студент должен уметь показать, почему конкретная
строка была признана обращением. Позднее эту часть можно заменить LLM, сохранив
расчёт срока и проверку оснований обычным кодом.

Ответом считается первое последующее сообщение сотрудника. Сообщение системы не
подходит, потому что назначение ответственного ещё не даёт клиенту
содержательного ответа. Новое сообщение клиента тоже не отвечает на предыдущий
вопрос. Сравнение выполняется только внутри уже упорядоченного окна одной
беседы.

Рабочее время вынесено в отдельный модуль, потому что это самостоятельное
правило. Функция прибавления минут должна уметь начать до рабочего дня, внутри
него и после него, а также пройти через выходные. Обратная функция подсчёта
минут обязана использовать те же границы. Иначе срок и отображаемое время
ожидания будут противоречить друг другу.

## Подготовка и раскладка

Дополнительные библиотеки не нужны: достаточно Pydantic, Typer и pytest.

```text
src/unanswered_detector/
  models.py
  question_rules.py
  business_time.py
  detector.py
  cli.py
tests/
  test_question_rules.py
  test_business_time.py
  test_detector.py
```

## 1. Модели

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


class DetectionContext(BaseModel):
    workspace_id: str
    conversation_id: str
    response_sla_minutes: int = Field(gt=0)
    analysis_time: AwareDatetime
    messages: tuple[MessageView, ...]


class FindingCandidate(BaseModel):
    model_config = ConfigDict(frozen=True)
    type: Literal["unanswered"] = "unanswered"
    subject_key: str
    summary: str
    evidence_message_ids: tuple[str, ...]
    waiting_minutes: int = Field(ge=0)
    due_at: AwareDatetime
```

`DetectionContext` удобен тем, что функция получает один проверенный объект, а
не пять несвязанных аргументов.

## 2. Распознавание обращения

`question_rules.py`:

```python
from .models import MessageView

REQUEST_PREFIXES = ("подскажите", "уточните", "пришлите", "когда", "можно")


def is_customer_request(message: MessageView) -> bool:
    if message.sender_role != "customer":
        return False
    normalized = message.text.strip().lower()
    return (
        "?" in normalized
        or normalized.startswith(REQUEST_PREFIXES)
        or "ждём ответ" in normalized
    )
```

Эта функция не знает о сроке, других сообщениях или хранилище.

Примеры:

```python
assert is_customer_request(customer_message("Когда будет файл?")) is True
assert is_customer_request(customer_message("Спасибо")) is False
assert is_customer_request(employee_message("Когда будет файл?")) is False
```

## 3. Рабочее время

`business_time.py`:

```python
from datetime import datetime, time, timedelta

WORK_START = time(9, 0)
WORK_END = time(18, 0)


def is_working_day(value: datetime) -> bool:
    return value.isoweekday() in {1, 2, 3, 4, 5}


def add_working_minutes(start: datetime, minutes: int) -> datetime:
    """Прибавить минуты, учитывая рабочие дни и время 09:00–18:00."""
    # TODO: двигаться по доступным отрезкам рабочего времени.
    raise NotImplementedError


def working_minutes_between(start: datetime, end: datetime) -> int:
    """Посчитать полные рабочие минуты между двумя моментами."""
    # TODO: использовать те же границы, что add_working_minutes.
    raise NotImplementedError
```

Начните с одного дня, затем добавьте переход через вечер, после этого выходные.
Не пытайтесь сразу покрыть все случаи одним сложным циклом.

Первый тест:

```python
from datetime import datetime

from unanswered_detector.business_time import add_working_minutes


def test_minutes_continue_next_workday() -> None:
    start = datetime.fromisoformat("2026-09-14T17:45:00+03:00")
    assert add_working_minutes(start, 30) == datetime.fromisoformat(
        "2026-09-15T09:15:00+03:00"
    )
```

## 4. Анализатор

`detector.py`:

```python
from .business_time import add_working_minutes, working_minutes_between
from .models import DetectionContext, FindingCandidate, MessageView
from .question_rules import is_customer_request


def first_employee_reply_after(
    messages: tuple[MessageView, ...], request_index: int
) -> MessageView | None:
    for message in messages[request_index + 1:]:
        if message.sender_role == "employee":
            return message
    return None


class UnansweredDetector:
    version = "unanswered-rules-1"

    def detect(self, context: DetectionContext) -> tuple[FindingCandidate, ...]:
        candidates: list[FindingCandidate] = []
        for index, message in enumerate(context.messages):
            if not is_customer_request(message):
                continue
            if first_employee_reply_after(context.messages, index) is not None:
                continue
            due_at = add_working_minutes(
                message.sent_at, context.response_sla_minutes
            )
            if context.analysis_time < due_at:
                continue
            # TODO: создать FindingCandidate и добавить в список.
        return tuple(sorted(candidates, key=lambda item: (item.due_at, item.subject_key)))
```

Пример вызова:

```python
detector = UnansweredDetector()
items = detector.detect(context)
for item in items:
    print(item.subject_key, item.waiting_minutes)
```

## 5. Проверка полного сценария

```python
def test_expected_message_is_detected(full_context) -> None:
    items = UnansweredDetector().detect(full_context)
    by_key = {item.subject_key: item for item in items}

    key = "unanswered:conv-n-001:n-001-012"
    assert key in by_key
    assert by_key[key].evidence_message_ids == ("n-001-012",)
```

Отдельно проверьте, что `n-006-002` не обнаруживается, поскольку срок ещё не
истёк, а системное сообщение после него не является ответом.

## Порядок реализации

1. Модели.
2. Признак обращения и пять маленьких тестов.
3. Прибавление минут внутри одного дня.
4. Переход через конец дня.
5. Переход через выходные.
6. Поиск ответа после позиции.
7. Один кандидат без сортировки.
8. Несколько кандидатов и устойчивый порядок.
9. Команда и полный набор данных.

## Как проверять себя по ходу работы

Сначала добейтесь правильной работы на одном рабочем дне. Затем добавьте один
переход через 18:00 и только после него — выходные. Для каждой границы полезно
проверить три значения: за секунду до срока, ровно в срок и через секунду после.
Так становится видно, используется ли нужное сравнение `analysis_time < due_at`.

Кандидат должен объяснять сам себя: `subject_key` указывает на конкретное
обращение, `evidence_message_ids` содержит его идентификатор, `due_at` показывает
рассчитанный срок, а `waiting_minutes` — фактическое ожидание в рабочих минутах.
Не включайте в основания все сообщения окна: это затруднит проверку и создаст
ложное впечатление, что каждое из них повлияло на вывод.

Частая ошибка — считать ответом сообщение сотрудника, отправленное до вопроса,
или сообщение из будущего относительно анализа. Другая ошибка — использовать
системное текущее время. Все тесты должны передавать `analysis_time` явно, чтобы
один и тот же набор давал одинаковый результат в любой день запуска.

## Связь со Spine

В Spine расписание и правила срока могут стать политикой платформы. Анализатор
останется версионируемой способностью, получающей готовый контекст и возвращающей
кандидатов с основаниями.
