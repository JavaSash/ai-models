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
запуска сервера vLLM
```
vllm serve Qwen/Qwen2.5-Coder-14B-Instruct --max_model_len 16384 --port 8081 --gpu-memory-utilization 0.95
```

vllm serve - запуск сервера
Qwen/Qwen2.5-Coder-14B-Instruct - идентификатор модели (по умолчанию: HuggingfaceHub)
--max_model_len - максимальный размер контекстного окна (в токенах). Задаётся числом в степени 2. Чем > размер, тем > нужно памяти дял загрузки
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

