# ai-models

## Локальное развёртывание моделей

Ollama фреймворк для локального запуска LLM, работает на CPU, без GPU

### Установка Ollama (doc)

На Linux: https://github.com/ollama/ollama/blob/main/docs/linux.mdx
На Windows: https://github.com/ollama/ollama/blob/main/docs/windows.mdx
На MacOS: https://ollama.com/download/Ollama-darwin.zip
Через Docker: https://github.com/ollama/ollama/blob/main/docs/docker.mdx

Install file https://ollama.com/download

### Модели

https://ollama.com/search

ollama pull qwen2.5-coder:3b - tabAutocomplete (инлайн-подсказки) (потребление 2 ГБ ОЗУ)
ollama pull qwen2.5-coder:7b - Chat (обсуждение, генерация заготовок) (потребление 4,7 ГБ ОЗУ)

ollama run qwen2.5-coder:3b - запуск в консоли

### Ollama CLI commands

* list
* ps
* ls
* rm [model name]
* pull [model name]
* run [model name]
* stop [model name]
* ollama create ai-code-assistant -f ./Modelfile

### Modelfile

```
FROM qwen2.5-coder:3b
PARAMETER temperature 0.1
PARAMETER num_ctx 4096
PARAMETER repeat_penalty 1.5
SYSTEM Ты AI code assistant и помогаешь с написанием кода на Kotlin, Java. Ты используешь лучшие практики написания кода, избегаешь уязвимостей, учитываешь расширяемость, читаемость и оптимизацию кода.
```

temperature - отвечает за случайность генерации. Ниже - генерируются похожие ответы на 1 и тот же вопрос. Выше - разные
ответы на 1 вопрос.
num_ctx 4096 - размер окна контекста (4096 токенов)
repeat_penalty - чтобы модель не повторялась в генерации текста. Т.е. выше наказание модели за повторения в генерации.
SYSTEM - системый промт для модели. Базовые инструкции для модели, исходя из которых она должна действовать.

## Развёртывание LLM на сервере

vLLM - Open-source фреймворк для высокопроизводительного развертывания LLM на GPU. Даёт inference с минимальной
задержкой за счёт оптимизированного управления памятью (PagedAttention). OpenAI-compatible API Server
https://github.com/vllm-project/vllm
дока https://docs.vllm.ai/en/latest/getting_started/quickstart.html

### Установка

Создать venv для Python и активировать её:

```
python3 -m venv env
source env/bin/activate
```

Установка зависимостей:

```
python -m pip install vllm huggingface_hub huggingface_hub[cli]
```

Проверка установки:

```
vllm -h
```

### Модели

https://huggingface.co/MODELS

DeepSeek-Coder-V2-Lite-Instruct - относится к классу моделей со смесью экспертов (Mixture of Experts, MoE).
Это значит, что она не использует все свои 16 миллиардов параметров для каждого запроса, а умно выбирает только
нужную часть — примерно 2.4 миллиарда

### Сервер

https://reg.cloud/cloud/gpu - вариант за 38р\ч с DeepSeek-Coder-V2-Lite-Instruct, потестить
cloud.ru
https://www.runpod.io/?ref=cwrzd7jw
https://cloud.vast.ai/?ref_id=159265

с 1x NVIDIA A5000 (24 ГБ) примерно за 88 руб./час . Этого хватит для любой модели до 14B, а 24 ГБ памяти дают запас на
большой контекст и будущие эксперименты. Брать мощнее (A100, H100) для код-ассистента экономически нецелесообразно.

Просмотр характеристик видеокарт

```
nvidia-smi
```

#### DeepSeek-Coder-V2-Lite-Instruct

Железо:
* NVIDIA 2xA5000 24 Гб
* 2 × GPU A5000
* 128 ГБ DDR4
* 512 ГБ SSD NVMe

Параметры модели:
* Архитектура: Mixture of Experts (MoE), 16B общих параметров, но для каждого токена используется только 2.4B активных.
* Качество и квантизация: Предустановленная версия, скорее всего, FP16 (16-битная). Это означает максимальное качество, но
и больший объём занимаемой памяти. При необходимости можно будет заменить на 4-битную (INT4) для экономии VRAM.
* Контекстное окно: Заявлено до 128 000 токенов.
* Фичи
  * Fill-in-the-Middle (FIM): Может дописывать код не только в конце, но и в середине, между двумя фрагментами. Именно это используют IDE для инлайн-автодополнения.
  * Кодовые «личности»: Она обучалась специально для роли ассистента, который пишет, объясняет и рефакторит код.

### Расчёт потребления GPU моделями

Базовая формула (минимальный размер модели):

```
Объём VRAM = (Кол-во параметров в млрд) * (Битность) / 8
```

Модель Вес             (16-бит, FP16)        Занимает на деле (+30% KV-кэш)
qwen2.5-coder:3b        ~6 ГБ                ~8 ГБ ✅
qwen2.5-coder:7b        ~14 ГБ                ~18 ГБ ❌ не влезет в 16 ГБ
DeepSeek-Coder-V2-Lite    ~32 ГБ (всего)        ~5 ГБ (только акт. часть MoE) ✅
qwen2.5-coder:14b        ~28 ГБ                ~36 ГБ ❌ не влезет

Лайфхаки

1. Лучше брать квантизованную версию (4-бит, q4_K_M). Она занимает в 4 раза меньше памяти, чем 16-битная, и качество
   почти не страдает.
2. +30% к памяти модели — это запас под KV-кэш (контекст, системный промпт, историю). Без него модель вылетит при первой
   же сложной задаче.
3. Лучше модели с архитектурой MoE. У них активируется только часть параметров, поэтому DeepSeek-Coder-V2-Lite умещается
   и работает предсказуемо быстро.

### Deploy

запуск сервера vLLM
```
vllm serve Qwen/Qwen2.5-Coder-14B-Instruct --max_model_len 16384 --port 8081 --gpu-memory-utilization 0.95
```

vllm serve - запуск сервера
Qwen/Qwen2.5-Coder-14B-Instruct - идентификатор модели (по умолчанию: HuggingfaceHub)
--max_model_len - максимальный размер контекстного окна (в токенах). Задаётся числом в степени 2. Чем > размер, тем >
нужно памяти дял загрузки
модели. Если после загрузки модели остаётся мало памяти, то можно сократить контекстное окно.
--port - порт, на котором разворачивается сервер
--gpu-memory-utilization - доля памяти на одной GPU, которую сервер может занимать

```
vllm serve hugging-quants/Meta-Llama-3.1-70B-Instruct-AWQ-INT4 --max_model_len 16384 --port 8081 --quantization awq --tensor_parallel_size 2
```

--quantization - алгоритм квантизации загружаемой модели
--tensor_parallel_size - количество GPU, которое будет использоваться

С помощью переменной окружения CUDA_VISIBLE_DEVICES можно обозначить, какие именно девайсы будут "видны" для vLLM,
например:
export CUDA_VISIBLE_DEVICES=1 - только 1-й (индексация с нуля)

### ProxyAI (CodeGpt)

API для взаимодействия с моделью: v1/chat/completions
provider: openai (совместимый с ним API)
model: установленная модель
apiBase: хост:порт сервера с моделью

#### Настройки

Локально

1. ollama run [model-name]
2. IDEA -> File -> Settings -> Tools -> CodeGpt
   Service: Custom AI Service
   Preset template: Ollama
   URL: http://localhost:11434/v1/chat/completions
   Body:
   temperature: 0.1
   model: [model-name] из п.1
3. Advanced Settings
   Если таймайутит можно увеличить Connection/Read timeout
4. Configuration
   Max Completion tokens - окно для генерации ответа. Иногда ошибка возникает потому что не умещается ответ в это окно,
   можно расширять.

После изменения настроек чат в IDE нужно пересоздать.

### Continue.Dev

https://plugins.jetbrains.com/plugin/22707-continue

## Context providers
Context Providers - это компоненты, которые предоставляют дополнительную контекстную информацию AI-ассистенту для более точной и релевантной помощи. Они собирают различные типы контекста из вашей среды разработки.

Можно настраивать источники контекста, такие как:

* Открытые файлы в редакторе
* Недавние изменения в коде
* Внешние API, документация и базы знаний
* GitHub
* Внешние API, документация, базы знаний
* @codebase - позволяет искать определённые файлы в проекте и включать их в контекст чата

## Slash-команды
Slash-команды являются "горячими клавишами" для промптов, написанных заранее.

/share - сохранение диалога в md файл
/explain — объяснение кода
/refactor — рефакторинг выделенного фрагмента
/test — генерация тестов для функции
/continue — продолжение написания кода
/cmd - сгенерировать shell команду по заданию
/commit - генерация коммит сообщения на основе git diff
/issue - генерирует новое issue на github
Можно добавлять свои, например:
config.json
```
"customCommands": [
    {
      "name": "optimize",
      "description": "Suggest optimizations for my code",
      "prompt": "{{{ input }}}\n\nAnalyze the highlighted code and suggest optimizations. Focus on:\n- Reducing complexity\n- Improving efficiency\n- Enhancing readability\n- Using best practices\n\nFor each suggestion, explain why it's beneficial in a concise way. Format your response as bullet points."
    }
]
```
config.ts
```
export function modifyConfig(config: Config): Config {
  config.slashCommands?.push({
    name: "summarize-terminal",
    description: "Summarize the contents of the terminal",
    run: async function* (sdk) {
      // Get the terminal output
      const terminalContents = await sdk.ide.getTerminalContents();
      if (!terminalContents.trim()) {
        yield "The terminal is empty.";
        return;
      }

      for await (const summary of sdk.llm.streamComplete(
        `Summarize the following terminal output:\n\n${terminalContents}\n\nProvide a concise summary of what happened.`,
        new AbortController().signal,
        {
          maxTokens: 100,
        },
      )) {
        yield summary;
      }
    },
  });
  return config;
}
```

## Rules

Файл .continuerules позволяет задавать кастомные правила поведения ассистента:

* Изменение стилистики ответов
* Определение формата предложений
* Автоматизация повторяющихся задач
(Примеры)[https://docs.continue.dev/customize/deep-dives/rules]

Он заменяет системный промт - аналог AGENTS.md

## Development data

Данные об использовании плагина хранятся локально. Они помогают людям, ответственным за внедрение AI ассистента,
находить паттерны использования, выявлять узкие места и даже обучать LLM особенностям вашего проекта, передавая ему:

* Примеры автодополнений которые вы приняли или отвергли
* Истории диалога в режиме чата
* Успешные примеры из режима inline

### Specification-Driven Development
Хорошая спецификация для AI содержит:
* Цель и контекст — зачем этот компонент нужен, в какой системе живёт
* Функциональные требования — что именно должен делать код (в виде глаголов: принимает, валидирует, возвращает)
* Модели данных — структуры, типы, ограничения полей
* API-контракт — сигнатуры функций / эндпоинты, форматы входа и выхода
* Ограничения и нефункциональные требования — производительность, стек технологий, стиль кода
* Non-goals — явно то, что не входит в задачу

Спеку можно хранить как SPEC.md в папке задачи или фичи. Её можно передавать AI через системный промпт, прикреплять как файл контекста в Continue.dev или вставлять в начало чата.

### TDD — Test-Driven Development с AI
Классический цикл TDD: Red → Green → Refactor.

Red — пишем падающий тест, который описывает желаемое поведение
Green — пишем минимальную реализацию, чтобы тест прошёл
Refactor — улучшаем код, не нарушая тесты
С AI этот цикл меняется: AI берёт на себя шаг Green (и иногда Refactor), а ты остаёшься автором тестов — то есть автором требований. Это важный сдвиг: тесты становятся спецификацией, а не проверкой.

Паттерн 1: Ты пишешь тесты — AI пишет реализацию.
Описываешь поведение через тесты, скармливаешь их AI, получаешь реализацию, которая проходит все кейсы. Работает особенно хорошо для чистых функций и utility-кода.

Паттерн 2: Спецификация → AI генерирует тесты → AI генерирует реализацию.
Пишешь спецификацию (SDD), просишь AI сначала сгенерировать тест-кейсы, проверяешь их, затем просишь сгенерировать реализацию. Плюс: ты видишь, как AI понял задачу, ещё до того, как он написал код.

Паттерн 3: Агентный цикл (Red → Green в автономном режиме).
AI-агент (например, в режиме Agent в Continue.dev или Claude Code) сам запускает тесты, читает вывод, правит код и повторяет — до зелёного состояния. Ты задаёшь тесты и наблюдаешь за процессом.

### Model Context Protocol (MCP)
MCP это open-source протокол для стандартизации передачи контекста в LLM. MCP обеспечивает стандартизированный способ подключения моделей ИИ к различным источникам данных и инструментам.
* Позволяет приложениям подключать LLM к различным источникам данных и инструментам;
* Упрощает создание агентов и сложных рабочих процессов на основе LLM;
* Обеспечивает гибкость при смене провайдеров и поставщиков LLM.

**MCP использует клиент-серверную модель:**
* MCP Hosts – программы (например, IDE, AI-инструменты), которые получают данные через MCP;
* MCP Clients – протокольные клиенты, поддерживающие соединения с серверами (зачастую встроены в MCP Hosts);
* MCP Servers – сервисы, предоставляющие функциональность через MCP;
* Локальные источники данных – файлы, базы данных, сервисы на вашем компьютере;
* Удаленные сервисы – внешние API и облачные платформы, доступные через интернет.

**MCP Protocol** – это протокол взаимодействия между клиентом и сервером. Доступно два стандартных протокола: stdio и SSE.

**stdio** – клиент-серверная коммуникация посредством стандартного ввода-вывода:
Подходит для CLI инструментов
Эффективен при локальных интеграциях
Простой процесс коммуникации

**Server-Sent Events (SSE)** – поддерживает streaming посредством HTTP POST запросов:
Может работать в закрытых сетях
Подходит для распределенных интеграций

