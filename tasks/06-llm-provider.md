# Работа 06. Подключение LLM через VseLLM

[Основное условие](../responseflow-practice.md#работа-06-подключение-llm-через-vsellm)

## Чему учится разработчик

- подключать LLM через собственный интерфейс;
- использовать OpenAI-совместимый API прокси VseLLM;
- передавать модели только необходимый контекст;
- получать результат в виде вызова функции с JSON Schema;
- проверять ответ через Pydantic;
- обрабатывать тайм-ауты, временные ошибки и неверный ответ;
- вести учёт версии запроса, модели и числа токенов;
- проверять весь код без сети с помощью заглушки.

## Общая картина работы

LLM здесь используется только как анализатор текста. Она не записывает карточки,
не отправляет сообщения и не принимает решения от имени менеджера. На вход ей
передаётся ограниченное окно синтетической переписки, а на выходе ожидается
строго описанный набор ситуаций. Все действия после ответа выполняет обычный
проверяемый код.

Путь данных состоит из нескольких защитных границ. Построитель запроса выбирает
только разрешённый контекст. Собственный интерфейс скрывает библиотеку и
поставщика. Переходник выполняет сетевой вызов. Разборщик превращает JSON в
строгую модель. Предметный анализатор проверяет основания, порог и ограничения.
Если соединить эти обязанности в одной функции, сетевую часть будет трудно
заменить заглушкой, а ошибки ответа смешаются с ошибками предметных правил.

Структурированный ответ уменьшает число неоднозначностей, но не делает результат
доверенным. Модель может вернуть неизвестный тип, лишнее поле, отсутствующее
сообщение или слишком много элементов. Pydantic проверяет форму, а предметный
код — связь ответа с конкретным окном. Обе проверки обязательны.

Параметры сети и модели передаются снаружи, потому что они меняются независимо
от программы. При этом версия системной инструкции является частью результата:
без неё нельзя понять, почему два запуска одной модели дали разные правила
классификации.

## Почему VseLLM

VseLLM предоставляет единый OpenAI-совместимый API для моделей разных
поставщиков. Для обычного клиента используется базовый адрес
`https://api.vsellm.ru/v1`. Точное имя модели нужно брать из актуального каталога
или `GET /v1/models`, потому что набор моделей может меняться.

Документация: <https://vsellm.ru/docs>.

В учебной работе используется вызов функции: модель должна вызвать
`report_customer_situations` и передать аргументы, соответствующие JSON Schema.
VseLLM преобразует такой вызов в нативный формат выбранной модели и возвращает
его в едином OpenAI-формате.

## Правила безопасности и стоимости

- ключ хранится только в переменной `VSELLM_API_KEY`;
- ключ нельзя записывать в `.env.example`, Git, журнал или текст ошибки;
- настоящие данные клиентов запрещены;
- сетевой запуск выполняется только вручную с синтетическими данными;
- обычный `pytest` не делает платных запросов;
- один ручной запуск обрабатывает одну беседу;
- имя модели, ограничение токенов и тайм-аут задаются настройками;
- полный текст запроса и ответа не пишется в обычный журнал;
- перед отправкой показывается оценка количества символов контекста.

## Подготовка и библиотеки

```bash
uv init --package --name llm-situation-analyzer --python 3.12
uv add 'pydantic>=2.8,<3' 'openai>=1.50,<3' 'typer>=0.12,<1'
uv add --dev 'pytest>=8,<9' 'ruff>=0.6,<1'
mkdir -p tests data/fixtures
```

Пакет `openai` здесь является клиентом OpenAI-совместимого API. Предметный код
не должен импортировать его: импорт располагается только в переходнике VseLLM.

Добавьте в `pyproject.toml`:

```toml
[tool.pytest.ini_options]
pythonpath = ["src"]
markers = [
  "llm: ручная проверка с настоящей языковой моделью и расходованием баланса",
]
```

## Раскладка

```text
06-llm-situation-analyzer/
  pyproject.toml
  .env.example
  README.md
  data/fixtures/
    system-prompt-v1.md
    fake-llm-responses.json
  src/llm_situation_analyzer/
    models.py
    provider.py
    fake_provider.py
    vsellm_provider.py
    prompt_builder.py
    tool_schema.py
    response_parser.py
    analyzer.py
    settings.py
    cli.py
  tests/
    test_prompt_builder.py
    test_response_parser.py
    test_analyzer.py
    test_vsellm_manual.py
```

`.env.example`:

```dotenv
VSELLM_API_KEY=
VSELLM_BASE_URL=https://api.vsellm.ru/v1
VSELLM_MODEL=openai/gpt-5
VSELLM_TIMEOUT_SECONDS=30
VSELLM_MAX_OUTPUT_TOKENS=1200
```

Значение модели является примером. Перед ручным запуском студент сверяет
доступность идентификатора в документации или списке моделей.

## 1. Предметные модели

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


class LlmSituation(BaseModel):
    model_config = ConfigDict(frozen=True, extra="forbid")
    type: Literal[
        "unanswered", "commitment", "complaint",
        "document_request", "manager_attention"
    ]
    confidence: float = Field(ge=0, le=1)
    summary: str = Field(min_length=10, max_length=240)
    recommended_action: str = Field(min_length=10, max_length=300)
    evidence_message_ids: tuple[str, ...] = Field(min_length=1)
    subject_key: str = Field(min_length=1, max_length=160)


class LlmAnalysisPayload(BaseModel):
    model_config = ConfigDict(frozen=True, extra="forbid")
    situations: tuple[LlmSituation, ...]


class LlmUsage(BaseModel):
    input_tokens: int | None = None
    output_tokens: int | None = None


class LlmCallResult(BaseModel):
    provider: str
    model: str
    raw_arguments: str
    usage: LlmUsage


class AnalysisResult(BaseModel):
    model_config = ConfigDict(frozen=True)
    situations: tuple[LlmSituation, ...]
    provider: str
    model: str
    prompt_version: str
    usage: LlmUsage
```

`raw_arguments` содержит JSON-строку аргументов вызова функции. Она ещё не
считается доверенным предметным результатом.

## 2. Собственный интерфейс поставщика

`provider.py`:

```python
from typing import Protocol

from pydantic import BaseModel, ConfigDict

from .models import LlmCallResult


class LlmMessage(BaseModel):
    model_config = ConfigDict(frozen=True)
    role: str
    content: str


class LlmRequest(BaseModel):
    model_config = ConfigDict(frozen=True)
    messages: tuple[LlmMessage, ...]
    tool_name: str
    tool_description: str
    tool_parameters: dict
    max_output_tokens: int


class LlmProvider(Protocol):
    def complete(self, request: LlmRequest) -> LlmCallResult: ...
```

Анализатор зависит только от `LlmProvider`. Типы библиотеки `openai` не должны
появляться в `analyzer.py`, `models.py` или `response_parser.py`.

## 3. JSON Schema инструмента

Pydantic умеет построить схему из модели:

```python
from .models import LlmAnalysisPayload


def situation_tool_schema() -> dict:
    return {
        "type": "function",
        "function": {
            "name": "report_customer_situations",
            "description": (
                "Вернуть только ситуации, требующие реакции сотрудника, "
                "с точными идентификаторами сообщений-оснований."
            ),
            "parameters": LlmAnalysisPayload.model_json_schema(),
        },
    }
```

Поле `required` формируется из обязательных полей модели. Не добавляйте туда
необязательные поля вручную.

## 4. Подготовка запроса

`prompt_builder.py`:

```python
from pathlib import Path

from .models import ConversationWindow
from .provider import LlmMessage, LlmRequest
from .tool_schema import situation_tool_schema


def render_conversation(window: ConversationWindow) -> str:
    lines: list[str] = []
    for message in window.messages:
        safe_text = message.text.replace("\x00", "")
        lines.append(
            f"[{message.external_id}] {message.sent_at.isoformat()} "
            f"{message.sender_role}: {safe_text}"
        )
    return "\n".join(lines)


def build_request(window: ConversationWindow, project_context: str,
                  system_prompt_path: Path,
                  max_output_tokens: int) -> LlmRequest:
    system_prompt = system_prompt_path.read_text(encoding="utf-8")
    user_content = (
        "ПРОЕКТНЫЙ КОНТЕКСТ\n"
        f"{project_context}\n\n"
        "ПЕРЕПИСКА\n"
        f"{render_conversation(window)}"
    )
    tool = situation_tool_schema()["function"]
    return LlmRequest(
        messages=(
            LlmMessage(role="system", content=system_prompt),
            LlmMessage(role="user", content=user_content),
        ),
        tool_name=tool["name"],
        tool_description=tool["description"],
        tool_parameters=tool["parameters"],
        max_output_tokens=max_output_tokens,
    )
```

Не вставляйте имя клиента, документы или сообщения, которые не относятся к
выбранной беседе. Ограничение контекста выполняется до вызова LLM.

## 5. Переходник VseLLM

`vsellm_provider.py`:

```python
from openai import (
    APIConnectionError,
    APITimeoutError,
    InternalServerError,
    OpenAI,
    RateLimitError,
)

from .models import LlmCallResult, LlmUsage
from .provider import LlmProvider, LlmRequest


class TemporaryLlmError(RuntimeError):
    pass


class InvalidLlmResponseError(RuntimeError):
    pass


class VseLlmProvider(LlmProvider):
    def __init__(self, api_key: str, base_url: str, model: str,
                 timeout_seconds: float) -> None:
        self._model = model
        self._client = OpenAI(
            api_key=api_key,
            base_url=base_url,
            timeout=timeout_seconds,
            max_retries=0,
        )

    def complete(self, request: LlmRequest) -> LlmCallResult:
        try:
            response = self._client.chat.completions.create(
                model=self._model,
                messages=[item.model_dump() for item in request.messages],
                tools=[{
                    "type": "function",
                    "function": {
                        "name": request.tool_name,
                        "description": request.tool_description,
                        "parameters": request.tool_parameters,
                    },
                }],
                tool_choice={
                    "type": "function",
                    "function": {"name": request.tool_name},
                },
                max_tokens=request.max_output_tokens,
            )
        except (
            APIConnectionError,
            APITimeoutError,
            InternalServerError,
            RateLimitError,
        ) as error:
            raise TemporaryLlmError("Временная ошибка вызова LLM") from error

        message = response.choices[0].message
        if not message.tool_calls:
            raise InvalidLlmResponseError("Модель не вызвала обязательную функцию")
        call = message.tool_calls[0]
        if call.function.name != request.tool_name:
            raise InvalidLlmResponseError("Модель вызвала неизвестную функцию")
        usage = response.usage
        return LlmCallResult(
            provider="vsellm",
            model=self._model,
            raw_arguments=call.function.arguments,
            usage=LlmUsage(
                input_tokens=usage.prompt_tokens if usage else None,
                output_tokens=usage.completion_tokens if usage else None,
            ),
        )
```

Повторные попытки добавляются отдельной оболочкой только для временных ошибок.
Ошибки проверки ответа повторять автоматически не нужно. Создайте
`retrying_provider.py`:

```python
from collections.abc import Callable
from time import sleep

from .provider import LlmProvider, LlmRequest
from .models import LlmCallResult
from .vsellm_provider import TemporaryLlmError


class RetryingLlmProvider(LlmProvider):
    def __init__(self, delegate: LlmProvider, attempts: int = 3,
                 wait: Callable[[float], None] = sleep) -> None:
        if attempts < 1:
            raise ValueError("Число попыток должно быть положительным")
        self._delegate = delegate
        self._attempts = attempts
        self._wait = wait

    def complete(self, request: LlmRequest) -> LlmCallResult:
        for attempt in range(self._attempts):
            try:
                return self._delegate.complete(request)
            except TemporaryLlmError:
                if attempt == self._attempts - 1:
                    raise
                self._wait(2 ** attempt)
        raise AssertionError("Недостижимая ветвь")
```

В проверке передайте вместо `sleep` функцию, которая только записывает задержки
в список. Так тест не будет ждать настоящие секунды. Проверьте задержки `1` и
`2`, успех с третьей попытки и проброс ошибки после третьего сбоя.

## 6. Заглушка и разбор ответа

`fake_provider.py`:

```python
from .models import LlmCallResult
from .provider import LlmRequest


class FakeLlmProvider:
    def __init__(self, result: LlmCallResult) -> None:
        self._result = result
        self.calls: list[LlmRequest] = []

    def complete(self, request: LlmRequest) -> LlmCallResult:
        self.calls.append(request)
        return self._result
```

`response_parser.py`:

```python
import json

from pydantic import ValidationError

from .models import LlmAnalysisPayload, LlmCallResult


class InvalidStructuredOutputError(ValueError):
    pass


def parse_payload(result: LlmCallResult) -> LlmAnalysisPayload:
    try:
        raw = json.loads(result.raw_arguments)
        return LlmAnalysisPayload.model_validate(raw)
    except (json.JSONDecodeError, ValidationError) as error:
        raise InvalidStructuredOutputError("Неверный структурированный ответ LLM") from error
```

Тесты должны передать: правильный JSON, оборванный JSON, лишнее поле,
неизвестный тип, уверенность 1.2 и пустой список оснований.

## 7. Предметный анализатор

`analyzer.py`:

```python
from pathlib import Path

from .models import AnalysisResult, ConversationWindow, LlmSituation
from .prompt_builder import build_request
from .provider import LlmProvider
from .response_parser import parse_payload


class UnknownEvidenceError(ValueError):
    pass


class LlmSituationAnalyzer:
    def __init__(self, provider: LlmProvider, system_prompt_path: Path,
                 max_output_tokens: int = 1200,
                 confidence_threshold: float = 0.60,
                 max_situations: int = 20) -> None:
        self._provider = provider
        self._system_prompt_path = system_prompt_path
        self._max_output_tokens = max_output_tokens
        self._confidence_threshold = confidence_threshold
        self._max_situations = max_situations

    def analyze(self, window: ConversationWindow,
                project_context: str) -> AnalysisResult:
        request = build_request(
            window, project_context, self._system_prompt_path,
            self._max_output_tokens,
        )
        call = self._provider.complete(request)
        payload = parse_payload(call)
        if len(payload.situations) > self._max_situations:
            raise ValueError("LLM вернула слишком много ситуаций")
        available = {message.external_id for message in window.messages}
        accepted: list[LlmSituation] = []
        for situation in payload.situations:
            evidence = tuple(dict.fromkeys(situation.evidence_message_ids))
            missing = [item for item in evidence if item not in available]
            if missing:
                raise UnknownEvidenceError(
                    f"В окне нет сообщений-оснований: {missing}"
                )
            if situation.confidence < self._confidence_threshold:
                continue
            accepted.append(situation.model_copy(update={
                "evidence_message_ids": evidence,
            }))
        return AnalysisResult(
            situations=tuple(accepted),
            provider=call.provider,
            model=call.model,
            prompt_version="system-prompt-v1",
            usage=call.usage,
        )
```

Проверка оснований намеренно выполняется до фильтра по уверенности: модель не
должна получать возможность спрятать выдуманную ссылку в результате с низким
баллом. Если выбран иной порядок, студент обязан зафиксировать его отдельным
тестом и объяснить решение.

Модель не определяет ключ устранения повторов. `subject_key` считается частью
кандидата, но окончательный хеш строит приложение в следующей работе.

## 8. Настройки из окружения

`settings.py`:

```python
import os
from pydantic import BaseModel, Field


class LlmSettings(BaseModel):
    api_key: str = Field(min_length=1)
    base_url: str = "https://api.vsellm.ru/v1"
    model: str
    timeout_seconds: float = Field(gt=0, default=30)
    max_output_tokens: int = Field(gt=0, le=4000, default=1200)


def load_settings() -> LlmSettings:
    return LlmSettings(
        api_key=os.environ["VSELLM_API_KEY"],
        base_url=os.getenv("VSELLM_BASE_URL", "https://api.vsellm.ru/v1"),
        model=os.environ["VSELLM_MODEL"],
        timeout_seconds=float(os.getenv("VSELLM_TIMEOUT_SECONDS", "30")),
        max_output_tokens=int(os.getenv("VSELLM_MAX_OUTPUT_TOKENS", "1200")),
    )
```

Не используйте `python-dotenv` как обязательную зависимость. Переменные можно
установить в терминале; `.env.example` служит документацией.

## Разбор подходов, классов и методов

Предметные модели отделены от сетевых моделей. `MessageView` и
`ConversationWindow` описывают разрешённый контекст. `LlmSituation` описывает
одну ситуацию, а `LlmAnalysisPayload` — корневой объект ответа функции. Корневая
обёртка нужна, потому что вызов инструмента должен передать JSON-объект с
именованным полем, а не свободный массив.

`LlmUsage` допускает `None`, поскольку не каждый поставщик или ошибочный ответ
возвращает статистику токенов. `LlmCallResult` содержит ещё не доверенную строку
`raw_arguments`; хранить её отдельно от проверенных ситуаций важно, чтобы
случайно не использовать данные до разбора. `AnalysisResult`, напротив, уже
содержит проверенные ситуации и безопасные сведения о происхождении результата.

`LlmMessage` — одна роль и содержимое запроса. `LlmRequest` собирает всё, что
нужно поставщику: сообщения, имя и описание функции, её схему и предел ответа.
Он не содержит адрес API и секретный ключ — эти параметры относятся к
конкретной реализации поставщика. `LlmProvider.complete(request)` возвращает
единый `LlmCallResult`, каким бы ни был внешний сервис.

`situation_tool_schema()` получает схему непосредственно из
`LlmAnalysisPayload`. Это предотвращает расхождение ручного JSON Schema и
модели Pydantic. Если поле добавляется в модель, схема меняется вместе с ним.
Внешняя оболочка с `type="function"` соответствует формату клиента, а
`parameters` содержит предметную структуру аргументов.

`render_conversation(window)` создаёт явный текстовый формат: идентификатор,
время, роль и сообщение. Идентификатор обязательно передаётся модели, иначе она
не сможет вернуть проверяемые основания. Удаление нулевого символа является
минимальной технической очисткой; исправлять орфографию или смысл сообщения
нельзя.

`build_request(...)` объединяет системную инструкцию, проектный контекст и
переписку. Путь инструкции передаётся аргументом, что позволяет тесту
использовать временный файл. `max_output_tokens` проходит через все уровни до
сетевого вызова. Построитель не создаёт клиента и не знает секретов.

`VseLlmProvider.__init__` сохраняет выбранную модель и один раз создаёт клиент.
`max_retries=0` отключает скрытые повторы библиотеки: учебная оболочка должна
сама решать, какие ошибки повторять. Метод `complete` переводит собственный
`LlmRequest` в формат клиента, принудительно выбирает нужную функцию и
преобразует ответ обратно в `LlmCallResult`.

`TemporaryLlmError` обозначает сбой, при котором повтор может помочь:
тайм-аут, разрыв соединения, ограничение частоты или временная ошибка сервера.
`InvalidLlmResponseError` означает, что сетевой ответ получен, но договор не
выполнен. Вторую ошибку повторять автоматически не нужно: та же модель может
снова вернуть такой же неверный ответ и потратить баланс.

`RetryingLlmProvider` является оболочкой над любым `LlmProvider`. Параметр
`delegate` — настоящий поставщик, `attempts` — общее число попыток, `wait` —
функция ожидания. Передача `wait` через конструктор позволяет тесту записать
задержки без реального сна. Значение `2 ** attempt` создаёт паузы 1, 2 и далее
секунд между попытками.

`FakeLlmProvider` возвращает заранее заданный результат и сохраняет все
полученные запросы в `calls`. Эта история позволяет проверить не только итог,
но и то, что анализатор передал правильную инструкцию, функцию и предел
токенов. Заглушка не должна читать окружение или создавать настоящий клиент.

`parse_payload(result)` выполняет два перехода доверия. `json.loads` проверяет
синтаксис строки, `LlmAnalysisPayload.model_validate` — структуру и ограничения.
Обе технические ошибки превращаются в одну предметную
`InvalidStructuredOutputError`, которую внешний код умеет обработать.

`LlmSituationAnalyzer.analyze(...)` координирует полный путь. Он строит запрос,
вызывает поставщика, разбирает ответ, ограничивает число ситуаций, проверяет
каждое основание и применяет порог. Список доступных идентификаторов строится по
тому же окну, которое было отправлено модели. В конце метод сохраняет поставщика,
модель, версию инструкции и статистику токенов.

`LlmSettings` является проверенной конфигурацией. `load_settings()` — единственное
место чтения переменных окружения. Отсутствующий обязательный ключ или модель
должны привести к ранней понятной ошибке до построения анализатора. Служба и
предметные модели не должны самостоятельно вызывать `os.getenv`.

Полный поток имеет следующий вид:

```text
ConversationWindow
  → build_request → LlmRequest
  → RetryingLlmProvider → VseLlmProvider или FakeLlmProvider
  → LlmCallResult.raw_arguments
  → parse_payload → LlmAnalysisPayload
  → проверка оснований и порога → AnalysisResult
```

Такое количество небольших классов оправдано разными причинами изменения:
формат контекста, сетевой поставщик, схема ответа, правила повторов и предметная
проверка могут развиваться независимо.

## 9. Проверки

Обычная проверка всегда использует `FakeLlmProvider`:

```python
from pathlib import Path

from llm_situation_analyzer.analyzer import LlmSituationAnalyzer
from llm_situation_analyzer.fake_provider import FakeLlmProvider

PROMPT_PATH = Path("data/fixtures/system-prompt-v1.md")


def test_analyzer_validates_evidence(window, valid_call_result) -> None:
    provider = FakeLlmProvider(valid_call_result)
    analyzer = LlmSituationAnalyzer(provider, PROMPT_PATH)
    result = analyzer.analyze(window, "Ключевой клиент")
    assert len(provider.calls) == 1
    assert result.situations[0].evidence_message_ids == (
        "n-002-002", "n-002-003"
    )
```

Загрузку готового ответа вынесите в `tests/conftest.py`, чтобы тест не содержал
длинную JSON-строку:

```python
import json
from pathlib import Path

import pytest

from llm_situation_analyzer.models import LlmCallResult


@pytest.fixture
def valid_call_result() -> LlmCallResult:
    path = Path("data/fixtures/fake-llm-responses.json")
    raw = json.loads(path.read_text(encoding="utf-8"))
    return LlmCallResult.model_validate(raw["valid"])
```

Ручная сетевая проверка:

```python
import os
import pytest


@pytest.mark.llm
def test_vsellm_manual(synthetic_window) -> None:
    if not os.getenv("VSELLM_API_KEY"):
        pytest.skip("VSELLM_API_KEY не задан")
    settings = load_settings()
    provider = VseLlmProvider(
        api_key=settings.api_key,
        base_url=settings.base_url,
        model=settings.model,
        timeout_seconds=settings.timeout_seconds,
    )
    result = LlmSituationAnalyzer(
        provider,
        PROMPT_PATH,
        max_output_tokens=settings.max_output_tokens,
    ).analyze(
        synthetic_window, "Учебный проект"
    )
    assert all(item.evidence_message_ids for item in result.situations)
```

Команды:

```bash
uv run pytest -q -m 'not llm'
VSELLM_API_KEY='...' VSELLM_MODEL='openai/gpt-5' \
  uv run pytest -q -m llm
```

## Порядок реализации

1. Скопируйте `prompts/system-prompt-v1.md` и
   `analyzer/fake-llm-responses.json` из набора синтетических данных в
   `data/fixtures/` под именами, указанными в раскладке.
2. Предметные модели ответа.
3. `LlmProvider` и заглушка.
4. Разбор правильного и повреждённого JSON.
5. JSON Schema функции.
6. Построение запроса и снимок ожидаемого текста.
7. Проверка оснований и порога.
8. Учёт модели, версии запроса и токенов.
9. Настройки без ключа в файлах.
10. Переходник VseLLM и оболочка повторных попыток.
11. Ручной вызов одной синтетической беседы.

## Готово, если

Все обычные тесты проходят без сети. При наличии ключа ручная проверка получает
структурированный ответ, проверяет его и показывает модель и расход токенов, но
не печатает ключ и полный закрытый контекст.

## Как проверять себя по ходу работы

Сначала полностью пройдите путь с `FakeLlmProvider`: построение запроса, запись
вызова заглушкой, разбор ответа, проверка оснований и сбор итогового объекта.
Если этот путь не работает без сети, настоящий вызов только усложнит поиск
ошибки.

Для каждого слоя проверяйте свой вид сбоя. Разборщик отвечает за повреждённый
JSON и нарушение схемы. Предметный анализатор отвечает за неизвестные
основания и низкую уверенность. Переходник отвечает за тайм-аут, сетевое
соединение, ограничение частоты и отсутствие обязательного вызова функции.
Оболочка повторных попыток повторяет только временные ошибки и никогда не
повторяет ответ, который уже оказался структурно неверным.

Перед ручным запуском выведите только безопасную сводку: имя поставщика, модель,
версию инструкции, число сообщений и приблизительное число символов. Не
печатайте содержимое запроса и ключ. После запуска проверьте, что в результате
сохранились сведения о модели и токенах, а обычная команда `pytest -m 'not llm'`
по-прежнему не создаёт сетевой клиент.

Если настоящий поставщик вернул ошибку, сначала воспроизведите поведение через
заглушку. Например, для отсутствующего вызова функции создайте `LlmCallResult`
или поддельный ответ клиента с тем же нарушением. Так исправление останется
быстрым и воспроизводимым и не потребует повторно расходовать баланс.

## Связь со Spine

Переходник VseLLM позднее может быть заменён каталогом моделей или способностью
Spine. Предметный договор, версия запроса, проверка оснований и измерение качества
остаются частью ActionFlow.
