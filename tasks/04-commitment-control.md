# Работа 04. Обещания и просрочки

[Основное условие](../responseflow-practice.md#работа-04-контроль-обещаний-и-просрочек)

## Чему учится разработчик

- ограничивать поддерживаемый язык явными правилами;
- разбирать относительные сроки от времени сообщения;
- связывать обещание и подтверждение выполнения;
- учитывать редакции без создания нового предмета контроля.

## Как устроена эта работа

Обещание отличается от обычной фразы тем, что в нём одновременно есть действие
сотрудника и проверяемый срок. Фраза «постараюсь прислать» без поддерживаемой
даты не создаёт ситуацию, как и дата без действия. Такое строгое правило
защищает учебный анализатор от догадок, которые невозможно объяснить тестом.

Работа разделена на три части. Разборщик срока превращает текстовую фразу в
конкретный момент времени. Небольшие функции распознают действие и подтверждение
выполнения. Анализатор соединяет результаты, но не должен повторно разбирать
дату или самостоятельно искать отдельные слова. Благодаря этому добавление
новой формы даты не меняет правила закрытия обещания.

Ключ предмета строится по исходному сообщению с обещанием, а не по его тексту и
не по сроку. Текст может измениться в новой редакции, срок тоже может быть
уточнён, но контролируется всё то же обязательство. Поэтому редакция 2 обновляет
результат для `n-001-006`, а не создаёт ещё одну карточку.

Повторные вопросы клиента становятся дополнительными основаниями той же
просрочки, только если они отвечают на исходное обещание и содержат одну из
поддерживаемых фраз. Основания показывают развитие проблемы, но не меняют её
устойчивый ключ. Именно это позволит позднее переоткрыть закрытую карточку при
появлении нового сообщения.

## Раскладка

```text
src/commitment_control/
  models.py
  deadline_parser.py
  completion_rules.py
  detector.py
  cli.py
tests/
  test_deadline_parser.py
  test_completion_rules.py
  test_detector.py
```

## 1. Модели

```python
from typing import Literal

from pydantic import AwareDatetime, BaseModel, ConfigDict


class MessageView(BaseModel):
    model_config = ConfigDict(frozen=True)
    external_id: str
    sender_role: Literal["customer", "employee", "system"]
    sent_at: AwareDatetime
    text: str
    reply_to_external_id: str | None = None


class CommitmentCandidate(BaseModel):
    model_config = ConfigDict(frozen=True)
    type: Literal["commitment", "delay"]
    subject_key: str
    summary: str
    due_at: AwareDatetime
    commitment_message_id: str
    completion_message_id: str | None = None
    evidence_message_ids: tuple[str, ...]
    responsible_role: Literal["account_manager"] = "account_manager"
```

Храните отдельно идентификатор исходного обещания и основания. Это делает
договор понятнее, даже если сейчас значения совпадают.

## 2. Разбор сроков

`deadline_parser.py`:

```python
import re
from datetime import datetime, timedelta

TODAY = re.compile(r"сегодня до (?P<hour>\d{1,2}):(?P<minute>\d{2})", re.I)
TOMORROW = re.compile(r"завтра до (?P<hour>\d{1,2}):(?P<minute>\d{2})", re.I)
HOURS = re.compile(r"в течение (?P<hours>\d+) часов?", re.I)


class InvalidDeadlineError(ValueError):
    pass


def _with_time(base: datetime, hour: int, minute: int) -> datetime:
    try:
        return base.replace(hour=hour, minute=minute, second=0, microsecond=0)
    except ValueError as error:
        raise InvalidDeadlineError(str(error)) from error


def parse_deadline(text: str, sent_at: datetime) -> datetime | None:
    if match := TODAY.search(text):
        return _with_time(sent_at, int(match["hour"]), int(match["minute"]))
    if match := TOMORROW.search(text):
        tomorrow = sent_at + timedelta(days=1)
        return _with_time(tomorrow, int(match["hour"]), int(match["minute"]))
    if match := HOURS.search(text):
        hours = int(match["hours"])
        # TODO: отклонить ноль и прибавить часы.
        raise NotImplementedError
    # TODO: добавить форму полной даты.
    return None
```

Не используйте `datetime.now()`: «сегодня» относится к дате сообщения.

Примеры:

```python
sent_at = datetime.fromisoformat("2026-09-14T12:12:00+03:00")
assert parse_deadline("Пришлю сегодня до 18:00", sent_at) == datetime.fromisoformat(
    "2026-09-14T18:00:00+03:00"
)
assert parse_deadline("Вернусь позже", sent_at) is None
```

## 3. Признаки обещания и выполнения

`completion_rules.py`:

```python
ACTION_WORDS = ("пришлю", "отправлю", "подготовлю", "исправлю", "сообщу")
COMPLETION_WORDS = (
    "отправил", "отправила", "направил", "направила", "готово", "исправлено"
)


def contains_word(text: str, words: tuple[str, ...]) -> bool:
    normalized_words = set(text.lower().replace(".", " ").split())
    return any(word in normalized_words for word in words)


def has_action_word(text: str) -> bool:
    return contains_word(text, ACTION_WORDS)


def is_completion_text(text: str) -> bool:
    return contains_word(text, COMPLETION_WORDS)


FOLLOW_UP_PHRASES = ("не получили", "всё ещё нет", "когда сможете")


def is_unresolved_follow_up(text: str) -> bool:
    normalized = text.lower()
    return any(phrase in normalized for phrase in FOLLOW_UP_PHRASES)
```

Позже можно улучшить обработку запятых, но сначала зафиксируйте ожидаемые формы
тестами.

## 4. Анализатор

```python
from datetime import datetime

from .completion_rules import has_action_word, is_completion_text, is_unresolved_follow_up
from .deadline_parser import parse_deadline
from .models import CommitmentCandidate, MessageView


class CommitmentDetector:
    version = "commitment-rules-1"

    def detect(self, conversation_id: str, messages: tuple[MessageView, ...],
               analysis_time: datetime) -> tuple[CommitmentCandidate, ...]:
        results: list[CommitmentCandidate] = []
        for index, message in enumerate(messages):
            if message.sender_role != "employee" or not has_action_word(message.text):
                continue
            due_at = parse_deadline(message.text, message.sent_at)
            if due_at is None:
                continue
            completion = self._find_completion(messages[index + 1:], analysis_time)
            if completion is not None:
                continue
            item_type = "delay" if due_at <= analysis_time else "commitment"
            follow_ups = self._find_unresolved_follow_ups(
                messages[index + 1:], message.external_id, analysis_time
            )
            evidence_ids = (
                message.external_id,
                *(item.external_id for item in follow_ups),
            )
            # TODO: собрать стабильный subject_key и кандидата с evidence_ids.
        return tuple(results)

    def _find_completion(self, later_messages: tuple[MessageView, ...],
                         analysis_time: datetime) -> MessageView | None:
        for message in later_messages:
            if message.sent_at > analysis_time:
                continue
            if message.sender_role == "employee" and is_completion_text(message.text):
                return message
        return None

    def _find_unresolved_follow_ups(
        self, later_messages: tuple[MessageView, ...], commitment_id: str,
        analysis_time: datetime,
    ) -> tuple[MessageView, ...]:
        return tuple(
            message for message in later_messages
            if message.sent_at <= analysis_time
            and message.sender_role == "customer"
            and message.reply_to_external_id == commitment_id
            and is_unresolved_follow_up(message.text)
        )
```

До передачи сообщений в анализатор выберите последние известные редакции, как
в работе 02.

## Разбор подходов, классов и методов

`CommitmentCandidate` хранит одновременно предметный ключ и специальные поля
обещания. `commitment_message_id` позволяет быстро найти исходное обязательство,
а `completion_message_id` нужен для объяснения выполненного случая, даже если
открытая карточка для него не создаётся. `evidence_message_ids` шире: кроме
обещания туда могут входить повторные обращения клиента.

Регулярные выражения `TODAY`, `TOMORROW` и `HOURS` описывают только разрешённые
формы. Именованные группы `hour`, `minute` и `hours` делают код чтения понятнее,
чем обращение к группам по номерам. Флаг `re.I` позволяет не создавать отдельные
варианты для начала предложения с заглавной буквы.

`_with_time(base, hour, minute)` сосредоточивает проверку часов и минут.
Стандартный `datetime.replace` уже умеет обнаружить 25 часов или 80 минут;
функция переводит такую техническую ошибку в предметную
`InvalidDeadlineError`. Это хороший пример использования возможностей
библиотеки вместо повторного ручного диапазона.

`parse_deadline(text, sent_at)` возвращает конкретный момент или `None`, если
поддерживаемой формы нет. `None` здесь не является ошибкой: большинство
сообщений вообще не содержит срока. Ошибкой является распознанная форма с
невозможным значением. Для полной даты нужно также определить год и сохранить
часовой пояс `sent_at`.

`contains_word(text, words)` выполняет общую механическую проверку, а
`has_action_word` и `is_completion_text` дают ей предметные имена. Анализатор
читает как предложение: «если есть слово действия», а не «если набор токенов
пересекается с константой». Позднее способ нормализации можно улучшить в одном
месте.

`is_unresolved_follow_up(text)` ищет фразы, показывающие, что клиент всё ещё
ждёт результат. Одного совпадения текста недостаточно: метод анализатора
`_find_unresolved_follow_ups` дополнительно проверяет роль, время и
`reply_to_external_id`.

`CommitmentDetector.detect(...)` обходит сообщения сотрудников, проверяет слово
действия и разбирает срок. Затем `_find_completion` просматривает только более
позднюю часть окна и игнорирует сообщения после анализа. Если подтверждение
найдено, открытый кандидат не создаётся. В противном случае срок определяет тип
`commitment` или `delay`.

`_find_completion` и `_find_unresolved_follow_ups` сделаны отдельными методами,
потому что отвечают на разные вопросы и имеют разные правила ролей. Их можно
проверить независимо, не создавая полностью заполненный кандидат.

```text
сообщение сотрудника
  → слово действия + parse_deadline
  → поиск подтверждения
  → поиск повторных вопросов
  → commitment или delay с единым subject_key
```

## Тесты эталонных случаев

```python
def test_overdue_edited_commitment(detector, conversation_n_001) -> None:
    items = detector.detect(
        "conv-n-001", conversation_n_001,
        datetime.fromisoformat("2026-09-15T15:00:00+03:00"),
    )
    delay = next(item for item in items if item.commitment_message_id == "n-001-006")
    assert delay.type == "delay"
    assert delay.due_at.isoformat() == "2026-09-14T18:00:00+03:00"
    assert delay.evidence_message_ids == ("n-001-006", "n-001-010", "n-001-012")


def test_completed_commitment_is_not_open(detector, conversation_n_004) -> None:
    items = detector.detect("conv-n-004", conversation_n_004, ANALYSIS_TIME)
    assert items == ()


def test_new_follow_up_updates_same_commitment(detector, conversation_with_new_evidence) -> None:
    items = detector.detect("conv-n-001", conversation_with_new_evidence, NEXT_ANALYSIS_TIME)
    delay = next(item for item in items if item.commitment_message_id == "n-001-006")
    assert delay.subject_key == "commitment:conv-n-001:n-001-006"
    assert delay.evidence_message_ids[-1] == "n-001-014"
```

## Порядок реализации

1. Одна форма «сегодня».
2. Неверные часы и минуты.
3. «Завтра».
4. Полная дата.
5. Число часов.
6. Слова действия и завершения.
7. Одно открытое обещание.
8. Одна просрочка.
9. Выполненное обещание.
10. Выбор редакции и полный набор.
11. Повторные сообщения клиента как новые основания того же обязательства.

## Как проверять себя по ходу работы

Для каждой формы даты напишите минимум три проверки: допустимая фраза,
невозможное значение и близкая, но неподдерживаемая формулировка. Анализатор не
должен «догадываться» о последней. Если нужна новая форма, сначала явно добавьте
её в условие и проверки.

Отдельно проверьте порядок сообщений. Подтверждение до обещания не выполняет
его; подтверждение после времени анализа ещё не известно; сообщение клиента со
словом «готово» не является подтверждением сотрудника. При открытой просрочке
убедитесь, что все основания существуют в переданном окне и идут в постоянном
порядке.

Полезная ручная проверка — вывести для каждого найденного обещания четыре
значения: идентификатор, разобранный срок, найденное подтверждение и итоговый
тип. Если срок правильный, но тип неверен, проблема находится в сравнении со
временем анализа. Если срок отсутствует, ошибку нужно искать в разборщике, а не
в основном цикле.

## Связь со Spine

В платформе извлечение обещания может выполнять агент, но срок и факт его
наступления лучше проверять кодом. Стабильный ключ позволяет длительному процессу
контролировать одно обязательство после замены анализатора.
