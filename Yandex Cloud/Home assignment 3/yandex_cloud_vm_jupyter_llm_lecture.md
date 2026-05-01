# Практическая лекция: Виртуальная машина + JupyterLab + LLM в Yandex Cloud

**Тема:** как студенту создать виртуальную машину в Yandex Cloud, подключиться к ней по SSH, развернуть JupyterLab, загрузить данные и подключить LLM через Yandex AI Studio.

---

## 0. Что мы хотим получить в конце

В конце занятия у студента должна быть рабочая схема:

```text
Локальный компьютер студента
        ↓ SSH
Виртуальная машина в Yandex Cloud
        ↓
Python + JupyterLab
        ↓
Ноутбуки, данные, ML-код
        ↓
Yandex AI Studio API
        ↓
LLM: qwen2.5-72b-instruct / gemma-3-12b-it
```

Идея простая: студент работает в браузере, но вычисления выполняются не на его ноутбуке, а на удалённой Linux-машине в облаке.

---

## 1. Что заранее нужно студенту

Перед занятием студенту желательно иметь:

1. Аккаунт в **Yandex Cloud**.
2. Активированный платёжный аккаунт.
3. Доступ к **Compute Cloud**.
4. Локальный терминал:
   - macOS/Linux: Terminal;
   - Windows: PowerShell, CMD или Windows Terminal.
5. SSH-клиент:
   - на macOS/Linux он обычно уже есть;
   - на Windows 10/11 OpenSSH обычно тоже доступен из PowerShell.
6. Базовое понимание, что такое:
   - виртуальная машина;
   - терминал;
   - SSH;
   - Python;
   - Jupyter Notebook.

---

## 2. Архитектура занятия

### 2.1. Основные этапы

```text
1. Создать SSH-ключи
2. Создать виртуальную машину в Yandex Cloud
3. Подключиться к ВМ по SSH
4. Обновить Linux и установить базовые пакеты
5. Создать Python virtual environment
6. Установить JupyterLab и библиотеки
7. Запустить JupyterLab на ВМ
8. Пробросить порт 8888 через SSH-туннель
9. Открыть Jupyter в браузере через localhost
10. Загрузить данные и ноутбук
11. Создать .env для ключей
12. Подключить Yandex AI Studio
13. Выполнить минимальный LLM-пример
14. Выполнить анализ текста через LLM
```

---

## 3. Создание SSH-ключей

SSH-ключ состоит из двух частей:

```text
Приватный ключ  — хранится только у студента на компьютере
Публичный ключ  — добавляется в Yandex Cloud / на виртуальную машину
```

**Приватный ключ нельзя отправлять в чат, выкладывать в GitHub, пересылать студентам или хранить в ноутбуках.**

### 3.1. Создание SSH-ключа на macOS / Linux

Открываем Terminal и выполняем:

```bash
ssh-keygen -t ed25519 -C "student-vm"
```

Если система спросит путь для сохранения ключа, можно нажать `Enter`.

Обычно ключи сохраняются здесь:

```bash
~/.ssh/id_ed25519
~/.ssh/id_ed25519.pub
```

Посмотреть публичный ключ:

```bash
cat ~/.ssh/id_ed25519.pub
```

Нужно скопировать всю строку, которая начинается примерно так:

```text
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAA...
```

### 3.2. Создание SSH-ключа на Windows

Открываем PowerShell и выполняем:

```powershell
ssh-keygen -t ed25519 -C "student-vm"
```

Обычно ключи сохраняются здесь:

```text
C:\Users\USERNAME\.ssh\id_ed25519
C:\Users\USERNAME\.ssh\id_ed25519.pub
```

Посмотреть публичный ключ:

```powershell
type C:\Users\USERNAME\.ssh\id_ed25519.pub
```

Где `USERNAME` нужно заменить на имя пользователя Windows.

---

## 4. Создание виртуальной машины в Yandex Cloud

### 4.1. Переход к созданию ВМ

В консоли Yandex Cloud:

```text
Compute Cloud → Виртуальные машины → Создать ВМ
```

### 4.2. Рекомендуемые параметры ВМ для учебной работы

Для обычной учебной работы с Jupyter и Python:

```text
ОС: Ubuntu 22.04 LTS или Ubuntu 24.04 LTS
vCPU: 4–8
RAM: 16–32 GB
Диск: network-ssd, 50–100 GB
Публичный IP: включить
Прерываемая ВМ: можно включить для кратких учебных экспериментов
```

Рекомендуемый минимум:

```text
4 vCPU
16 GB RAM
50 GB SSD
```

Более комфортный вариант:

```text
8 vCPU
32 GB RAM
100 GB SSD
```

### 4.3. Рекомендуемые параметры для ML-задач

Если планируются более тяжёлые вычисления:

```text
8–16 vCPU
32–64 GB RAM
100–200 GB SSD
```

Если планируются нейросети, embeddings, PyTorch/TensorFlow, local inference:

```text
GPU: 1 × NVIDIA T4
vCPU: 16–32
RAM: 64–128 GB
SSD: 150–300 GB
```

Важно: GPU полезен только тогда, когда код реально использует CUDA/PyTorch/TensorFlow или библиотеки, которые умеют работать с GPU. Для `pandas`, `sklearn`, обычной логистической регрессии и табличной обработки GPU часто не даёт большого прироста.

### 4.4. Пользователь ВМ

При создании ВМ нужно указать пользователя, например:

```text
student
```

или:

```text
pronevich-o
```

В поле SSH-ключа вставляется **публичный ключ** из файла `id_ed25519.pub`.

### 4.5. Security Group

Для учебного сценария минимально нужно открыть:

```text
TCP 22 — SSH
```

Лучше ограничить доступ своим IP, но для учебного занятия иногда временно ставят:

```text
0.0.0.0/0
```

Порт `8888` для Jupyter **лучше не открывать наружу**, потому что мы будем подключаться безопасно через SSH-туннель.

---

## 5. Подключение к ВМ по SSH

Допустим:

```text
IP ВМ: 158.160.225.211
Пользователь: student
```

### 5.1. Подключение на macOS / Linux

```bash
ssh student@158.160.225.211
```

Если нужно явно указать ключ:

```bash
ssh -i ~/.ssh/id_ed25519 student@158.160.225.211
```

### 5.2. Подключение на Windows

PowerShell / CMD:

```powershell
ssh student@158.160.225.211
```

Если нужно явно указать ключ:

```powershell
ssh -i C:\Users\USERNAME\.ssh\id_ed25519 student@158.160.225.211
```

### 5.3. Как понять, что подключение успешно

Если всё получилось, студент увидит строку примерно такого вида:

```bash
student@compute-vm:~$
```

Это значит, что команды теперь выполняются **внутри виртуальной машины**, а не на локальном компьютере.

---

## 6. Первичная настройка Linux на ВМ

Все следующие команды выполняются **внутри ВМ**.

### 6.1. Обновление системы

```bash
sudo apt update && sudo apt upgrade -y
```

### 6.2. Установка базовых пакетов

```bash
sudo apt install -y python3 python3-pip python3-venv git curl nano htop tmux
```

Что ставим:

```text
python3       — Python
python3-pip   — менеджер Python-пакетов
python3-venv  — виртуальные окружения
git           — работа с репозиториями
curl          — загрузка файлов и запросы
nano          — простой терминальный редактор
htop          — мониторинг ресурсов
tmux          — долгие сессии без потери процесса
```

---

## 7. Создание проекта и виртуального окружения

### 7.1. Создать папку проекта

```bash
mkdir -p ~/ml-jupyter-llm
cd ~/ml-jupyter-llm
```

### 7.2. Создать виртуальное окружение

```bash
python3 -m venv venv
```

### 7.3. Активировать виртуальное окружение

```bash
source venv/bin/activate
```

После активации в начале строки терминала должно появиться:

```text
(venv)
```

### 7.4. Проверить, что используется Python из venv

```bash
which python
which pip
```

Ожидаемый результат должен быть похож на:

```text
/home/student/ml-jupyter-llm/venv/bin/python
/home/student/ml-jupyter-llm/venv/bin/pip
```

---

## 8. Установка JupyterLab, библиотек анализа данных и LLM-библиотек

Выполняем внутри активированного `venv`.

### 8.1. Одна полная команда установки

```bash
pip install --upgrade pip setuptools wheel && pip install jupyterlab notebook ipywidgets widgetsnbextension jupyterlab_widgets tqdm ipykernel pandas numpy matplotlib seaborn scipy scikit-learn statsmodels plotly openpyxl xlsxwriter python-dotenv requests openai holidays sentence-transformers missingno memory-profiler line-profiler jupyterlab-execute-time rich yandex-ai-studio-sdk
```

### 8.2. Что устанавливается

```text
jupyterlab, notebook        — Jupyter-интерфейс
ipywidgets                  — интерактивные виджеты
tqdm                        — progress bar
pandas, numpy               — анализ данных
matplotlib, seaborn, plotly — визуализация
scipy, statsmodels          — статистика
scikit-learn                — классический ML
openpyxl, xlsxwriter        — работа с Excel
python-dotenv               — чтение ключей из .env
requests                    — HTTP-запросы
openai                      — OpenAI-compatible API
holidays                    — календарные признаки
sentence-transformers       — embeddings и NLP
missingno                   — анализ пропусков
memory-profiler             — анализ памяти
line-profiler               — построчный профайлинг
jupyterlab-execute-time     — время выполнения ячеек
rich                        — красивый вывод в терминале
yandex-ai-studio-sdk        — SDK для Yandex AI Studio
```

---

## 9. Проверка установки библиотек

Выполняем:

```bash
python -c "import pandas, numpy, sklearn, matplotlib, openai; print('OK')"
```

Если вывод:

```text
OK
```

значит базовые библиотеки установились.

---

## 10. Запуск JupyterLab на ВМ

### 10.1. Запуск Jupyter

```bash
cd ~/ml-jupyter-llm
source venv/bin/activate
jupyter lab --ip=0.0.0.0 --port=8888 --no-browser
```

Jupyter выдаст ссылку вида:

```text
http://127.0.0.1:8888/lab?token=...
```

Токен понадобится для входа в браузере.

### 10.2. Важные правила

Пока Jupyter работает:

```text
Не нажимать Ctrl+C в терминале с Jupyter.
Не закрывать терминал, где запущен Jupyter.
Не перезапускать kernel во время долгого вычисления.
```

Если нажать `Ctrl+C`, Jupyter-сервер остановится.

---

## 11. Подключение к Jupyter через SSH-туннель

Открываем **второй терминал на локальном компьютере**, не на ВМ.

### 11.1. macOS / Linux

```bash
ssh -L 8888:localhost:8888 student@158.160.225.211
```

Если нужен ключ:

```bash
ssh -i ~/.ssh/id_ed25519 -L 8888:localhost:8888 student@158.160.225.211
```

### 11.2. Windows PowerShell

```powershell
ssh -L 8888:localhost:8888 student@158.160.225.211
```

Если нужен ключ:

```powershell
ssh -i C:\Users\USERNAME\.ssh\id_ed25519 -L 8888:localhost:8888 student@158.160.225.211
```

### 11.3. Открыть Jupyter в браузере

В браузере открыть:

```text
http://localhost:8888
```

И вставить токен, который показал Jupyter в первом терминале.

Или открыть сразу полную ссылку:

```text
http://localhost:8888/lab?token=ВАШ_TOKEN
```

---

## 12. Как сделать так, чтобы Jupyter не падал при закрытии SSH

Для долгих вычислений лучше запускать Jupyter через `tmux`.

### 12.1. Создать tmux-сессию

```bash
tmux new -s jupyter
```

### 12.2. Внутри tmux запустить Jupyter

```bash
cd ~/ml-jupyter-llm
source venv/bin/activate
jupyter lab --ip=0.0.0.0 --port=8888 --no-browser
```

### 12.3. Выйти из tmux, не останавливая Jupyter

Нажать:

```text
Ctrl+B
D
```

### 12.4. Вернуться обратно в tmux

```bash
tmux attach -t jupyter
```

### 12.5. Посмотреть активные tmux-сессии

```bash
tmux ls
```

---

## 13. Загрузка файлов в Jupyter

### 13.1. Способ 1: через интерфейс Jupyter

В JupyterLab:

```text
Левая панель файлов → Upload Files → выбрать .ipynb / .xlsx / .csv / .pkl
```

Это самый простой способ для студентов.

### 13.2. Способ 2: через scp

С локального компьютера:

```bash
scp ~/Downloads/notebook.ipynb student@158.160.225.211:/home/student/ml-jupyter-llm/
```

Загрузка Excel:

```bash
scp ~/Downloads/data.xlsx student@158.160.225.211:/home/student/ml-jupyter-llm/
```

Загрузка папки целиком:

```bash
scp -r ~/Downloads/project_folder student@158.160.225.211:/home/student/ml-jupyter-llm/
```

С явным ключом:

```bash
scp -i ~/.ssh/id_ed25519 ~/Downloads/data.xlsx student@158.160.225.211:/home/student/ml-jupyter-llm/
```

---

## 14. Первая проверочная ячейка в Jupyter

Создаём новый ноутбук и выполняем:

```python
import sys
import platform
import pandas as pd
import numpy as np
import sklearn

print("Python:", sys.version)
print("Platform:", platform.platform())
print("pandas:", pd.__version__)
print("numpy:", np.__version__)
print("sklearn:", sklearn.__version__)
```

---

## 15. Информативный стартовый блок для ноутбука

Эта ячейка делает ноутбук удобнее для анализа данных:

```python
%load_ext memory_profiler
%load_ext line_profiler

import warnings
warnings.filterwarnings("ignore")

from tqdm.notebook import tqdm
import time
import logging

logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s | %(levelname)s | %(message)s"
)

print("Notebook environment is ready.")
```

### 15.1. Измерение времени выполнения ячейки

```python
%%time

# ваш код
```

### 15.2. Progress bar

```python
from tqdm.notebook import tqdm
import time

for i in tqdm(range(100)):
    time.sleep(0.05)
```

---

## 16. Подключение Yandex AI Studio

В нашем рабочем сценарии из ноутбука `do_blokcs.ipynb` использовалось подключение к Yandex AI Studio через OpenAI-compatible API.

Основная логика:

```python
from openai import OpenAI

client = OpenAI(
    api_key="...",
    base_url="https://api.yandexcloud.ai/v1/",
)
```

Модель, которая использовалась в рабочем варианте:

```text
qwen2.5-72b-instruct
```

Также встречался вариант:

```text
gemma-3-12b-it
```

---

## 17. Что нужно подготовить в Yandex AI Studio

Для подключения LLM нужно получить:

```text
1. API-ключ
2. Folder ID
3. Название модели
```

В нашем учебном примере будем использовать:

```text
YANDEX_MODEL=qwen2.5-72b-instruct
```

Запасной вариант:

```text
YANDEX_MODEL=gemma-3-12b-it
```

---

## 18. Безопасное хранение ключей через .env

### 18.1. Создать `.env`

На ВМ:

```bash
cd ~/ml-jupyter-llm
nano .env
```

Содержимое файла:

```env
YANDEX_API_KEY=сюда_вставить_API_ключ
YANDEX_FOLDER_ID=сюда_вставить_FOLDER_ID
YANDEX_BASE_URL=https://api.yandexcloud.ai/v1/
YANDEX_MODEL=qwen2.5-72b-instruct
```

Сохранить в `nano`:

```text
Ctrl+O → Enter → Ctrl+X
```

### 18.2. Создать `.gitignore`

```bash
nano .gitignore
```

Содержимое:

```text
.env
venv/
__pycache__/
.ipynb_checkpoints/
*.log
```

---

## 19. Проверка подключения к LLM через Python-скрипт

### 19.1. Создать файл `test_yandex_llm.py`

```bash
cd ~/ml-jupyter-llm
nano test_yandex_llm.py
```

Вставить код:

```python
import os
from dotenv import load_dotenv
from openai import OpenAI

load_dotenv()

YANDEX_API_KEY = os.getenv("YANDEX_API_KEY")
YANDEX_BASE_URL = os.getenv("YANDEX_BASE_URL", "https://api.yandexcloud.ai/v1/")
YANDEX_MODEL = os.getenv("YANDEX_MODEL", "qwen2.5-72b-instruct")

if not YANDEX_API_KEY:
    raise ValueError("Не найден YANDEX_API_KEY. Проверьте файл .env")

client = OpenAI(
    api_key=YANDEX_API_KEY,
    base_url=YANDEX_BASE_URL,
)

response = client.chat.completions.create(
    model=YANDEX_MODEL,
    messages=[
        {
            "role": "system",
            "content": "Ты помощник преподавателя по анализу данных. Отвечай кратко, структурированно и на русском языке."
        },
        {
            "role": "user",
            "content": "Объясни студентам, зачем нужна виртуальная машина для анализа данных и машинного обучения."
        }
    ],
    temperature=0.3,
    max_tokens=1000,
)

print(response.choices[0].message.content)
```

### 19.2. Запустить скрипт

```bash
source venv/bin/activate
python test_yandex_llm.py
```

Если всё настроено правильно, модель вернёт текстовый ответ.

---

## 20. Проверка подключения к LLM в Jupyter Notebook

### 20.1. Первая ячейка

```python
from dotenv import load_dotenv
from openai import OpenAI
import os

load_dotenv()

client = OpenAI(
    api_key=os.getenv("YANDEX_API_KEY"),
    base_url=os.getenv("YANDEX_BASE_URL", "https://api.yandexcloud.ai/v1/"),
)

model_name = os.getenv("YANDEX_MODEL", "qwen2.5-72b-instruct")

print("Model:", model_name)
print("API key loaded:", os.getenv("YANDEX_API_KEY") is not None)
```

### 20.2. Вторая ячейка: минимальный запрос к модели

```python
prompt = """
Объясни простыми словами, что такое виртуальная машина и зачем она нужна студенту,
который изучает анализ данных и машинное обучение.
"""

response = client.chat.completions.create(
    model=model_name,
    messages=[
        {
            "role": "system",
            "content": "Ты преподаватель анализа данных. Отвечай на русском языке, понятно и структурированно."
        },
        {
            "role": "user",
            "content": prompt
        }
    ],
    temperature=0.3,
    max_tokens=1000,
)

print(response.choices[0].message.content)
```

---

## 21. Минимальный пример: анализ текста через LLM

### 21.1. Один текст

```python
text = """
Виртуальная машина позволяет студентам работать с вычислительными ресурсами облака,
запускать Jupyter Notebook, выполнять анализ данных и подключать языковые модели через API.
"""
```

### 21.2. Промпт для анализа текста

```python
prompt = f"""
Проанализируй следующий текст и выдели:

1. Главную мысль
2. Ключевые термины
3. Возможную категорию текста
4. Краткое резюме в 2 предложениях

Текст:
{text}
"""
```

### 21.3. Запрос к LLM

```python
response = client.chat.completions.create(
    model=model_name,
    messages=[
        {
            "role": "system",
            "content": "Ты аналитик текста. Отвечай строго по пунктам и на русском языке."
        },
        {
            "role": "user",
            "content": prompt
        }
    ],
    temperature=0.2,
    max_tokens=1000,
)

print(response.choices[0].message.content)
```

---

## 22. Пример: анализ нескольких текстов из DataFrame

### 22.1. Создаём таблицу

```python
import pandas as pd

df = pd.DataFrame({
    "id": [1, 2, 3],
    "text": [
        "Клиент жалуется на долгую обработку заявки и отсутствие обратной связи.",
        "Пользователь благодарит службу поддержки за быстрое решение проблемы.",
        "В отчете описаны причины роста количества ДТП на городских перекрестках."
    ]
})

df
```

### 22.2. Функция анализа текста

```python
def analyze_text_with_llm(text: str) -> str:
    prompt = f"""
    Проанализируй текст. Определи:

    1. Тональность
    2. Основную тему
    3. Краткое резюме
    4. Есть ли проблема, требующая реакции

    Текст:
    {text}
    """

    response = client.chat.completions.create(
        model=model_name,
        messages=[
            {
                "role": "system",
                "content": "Ты аналитик текстовых данных. Отвечай кратко и структурированно."
            },
            {
                "role": "user",
                "content": prompt
            }
        ],
        temperature=0.2,
        max_tokens=700,
    )

    return response.choices[0].message.content
```

### 22.3. Применить к одной строке

```python
print(analyze_text_with_llm(df.loc[0, "text"]))
```

### 22.4. Применить ко всем строкам

```python
from tqdm.notebook import tqdm

tqdm.pandas()

df["llm_analysis"] = df["text"].progress_apply(analyze_text_with_llm)

df
```

---

## 23. Альтернативный вариант через Yandex AI Studio SDK

Этот вариант был в ноутбуке `do_blokcs.ipynb`, но для студентов основной способ лучше показывать через `openai`, потому что он универсальнее.

### 23.1. Пример через SDK

```python
import os
from dotenv import load_dotenv
from yandex_ai_studio_sdk import AIStudio

load_dotenv()

sdk = AIStudio(
    folder_id=os.getenv("YANDEX_FOLDER_ID"),
    auth=os.getenv("YANDEX_API_KEY"),
)

model = sdk.models.completions("gemma-3-12b-it")
model = model.configure(
    temperature=0.3,
    max_tokens=1000,
)

result = model.run([
    {
        "role": "system",
        "text": "Ты помощник преподавателя. Отвечай на русском языке."
    },
    {
        "role": "user",
        "text": "Составь краткий план занятия по виртуальным машинам и Jupyter."
    }
])

print(result)
```

---

## 24. Если модель не работает

### 24.1. Возможные причины

```text
1. Не активирован биллинг.
2. Неверный API-ключ.
3. Неверный Folder ID.
4. Нет доступа к выбранной модели.
5. Неверное название модели.
6. Не тот base_url.
7. .env лежит не в той папке.
8. Kernel Jupyter не перезапущен после изменения .env.
```

### 24.2. Проверка переменных окружения

```python
import os
from dotenv import load_dotenv

load_dotenv()

print("YANDEX_API_KEY:", os.getenv("YANDEX_API_KEY") is not None)
print("YANDEX_FOLDER_ID:", os.getenv("YANDEX_FOLDER_ID"))
print("YANDEX_BASE_URL:", os.getenv("YANDEX_BASE_URL"))
print("YANDEX_MODEL:", os.getenv("YANDEX_MODEL"))
```

### 24.3. Попробовать другую модель

В `.env` можно заменить:

```env
YANDEX_MODEL=qwen2.5-72b-instruct
```

на:

```env
YANDEX_MODEL=gemma-3-12b-it
```

После этого нужно заново выполнить ячейку:

```python
load_dotenv()
```

или перезапустить kernel.

---

## 25. Типовые ошибки студентов

### Ошибка 1. `Permission denied (publickey)`

Причины:

```text
не тот пользователь;
не тот SSH-ключ;
публичный ключ не добавлен в ВМ;
студент подключается не к той ВМ.
```

Решение:

```bash
ssh -i ~/.ssh/id_ed25519 student@IP
```

Проверить, что в Yandex Cloud добавлен именно публичный ключ из `id_ed25519.pub`.

### Ошибка 2. `sudo is not recognized`

Это значит, что студент ввёл Linux-команду в Windows CMD/PowerShell, а не на ВМ.

Правильно:

```powershell
ssh student@IP
```

И только после успешного входа выполнять:

```bash
sudo apt update
```

### Ошибка 3. `externally-managed-environment`

Причина: студент пытается установить pip-пакеты в системный Python.

Правильно:

```bash
python3 -m venv venv
source venv/bin/activate
pip install ...
```

### Ошибка 4. Jupyter просит token

На ВМ:

```bash
jupyter lab list
```

или:

```bash
jupyter server list
```

### Ошибка 5. `Address already in use`

Порт 8888 уже занят.

Решение:

```bash
jupyter lab --ip=0.0.0.0 --port=8889 --no-browser
```

И туннель:

```bash
ssh -L 8889:localhost:8889 student@IP
```

Браузер:

```text
http://localhost:8889
```

### Ошибка 6. `IProgress not found`

Решение:

```bash
pip install ipywidgets widgetsnbextension jupyterlab_widgets
```

Перезапустить Jupyter.

---

## 26. Мини-шпаргалка команд

### 26.1. На ВМ

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y python3 python3-pip python3-venv git curl nano htop tmux

mkdir -p ~/ml-jupyter-llm
cd ~/ml-jupyter-llm

python3 -m venv venv
source venv/bin/activate

pip install --upgrade pip setuptools wheel && pip install jupyterlab notebook ipywidgets widgetsnbextension jupyterlab_widgets tqdm ipykernel pandas numpy matplotlib seaborn scipy scikit-learn statsmodels plotly openpyxl xlsxwriter python-dotenv requests openai holidays sentence-transformers missingno memory-profiler line-profiler jupyterlab-execute-time rich yandex-ai-studio-sdk
```

### 26.2. Создать `.env`

```bash
nano .env
```

```env
YANDEX_API_KEY=сюда_ключ
YANDEX_FOLDER_ID=сюда_folder_id
YANDEX_BASE_URL=https://api.yandexcloud.ai/v1/
YANDEX_MODEL=qwen2.5-72b-instruct
```

### 26.3. Запустить Jupyter

```bash
cd ~/ml-jupyter-llm
source venv/bin/activate
jupyter lab --ip=0.0.0.0 --port=8888 --no-browser
```

### 26.4. Второй терминал на компьютере

```bash
ssh -L 8888:localhost:8888 student@IP
```

### 26.5. Браузер

```text
http://localhost:8888/lab?token=TOKEN
```

---

## 27. Что студент должен сдать после занятия

Студент может приложить:

1. Скриншот созданной ВМ в Yandex Cloud.
2. Скриншот успешного SSH-подключения.
3. Скриншот запущенного JupyterLab.
4. Ноутбук `.ipynb`, где:
   - выполнена проверочная ячейка с версиями библиотек;
   - загружены тестовые данные;
   - выполнен минимальный анализ данных;
   - выполнен запрос к LLM;
   - получен ответ модели.
5. Краткий текстовый отчёт:
   - какие ресурсы ВМ были выбраны;
   - какая модель LLM использовалась;
   - какие ошибки возникли и как они были исправлены.

---

## 28. Контрольные вопросы для студентов

1. Чем виртуальная машина отличается от локального компьютера?
2. Зачем нужен SSH?
3. Почему приватный ключ нельзя отправлять другим людям?
4. Почему не стоит открывать Jupyter напрямую в интернет?
5. Что делает SSH-туннель?
6. Зачем нужен `venv`?
7. Почему возникает ошибка `externally-managed-environment`?
8. Для чего нужен файл `.env`?
9. Чем `qwen2.5-72b-instruct` отличается от локальной LLM?
10. Когда GPU действительно ускоряет вычисления?
11. Почему `pandas` и `sklearn` не всегда ускоряются на GPU?
12. Что делает `tmux`?
13. Что такое `base_url` при подключении к OpenAI-compatible API?
14. Почему API-ключи нельзя хранить прямо в ноутбуке?
15. Как проверить, что переменные окружения загрузились?

---

## 29. Итоговая логика занятия

```text
Сначала студент создаёт удалённую Linux-машину.
Потом подключается к ней по SSH.
Дальше создаёт Python-окружение и ставит Jupyter.
Затем открывает Jupyter через безопасный localhost-туннель.
После этого загружает данные и запускает Python-код.
Финальный этап — подключение Yandex AI Studio и выполнение LLM-запроса.
```

В результате студент понимает не только “куда нажать”, но и всю архитектуру:

```text
Локальный компьютер → SSH → ВМ → Python/Jupyter → данные/ML → Yandex AI Studio → LLM-ответ
```

---

## 30. Полезные источники

- Yandex Cloud: создание Linux VM и подключение по SSH:  
  https://yandex.cloud/en/docs/compute/quickstart/quick-create-linux

- Yandex Cloud: подключение к Linux VM по SSH:  
  https://yandex.cloud/en/docs/compute/operations/vm-connect/ssh

- Yandex Cloud: security groups:  
  https://yandex.cloud/en/docs/vpc/concepts/security-groups

- Yandex AI Studio: документация:  
  https://aistudio.yandex.ru/docs/en/

- Yandex AI Studio: quickstart:  
  https://aistudio.yandex.ru/docs/en/ai-studio/quickstart/index.html

- Yandex AI Studio SDK:  
  https://aistudio.yandex.ru/docs/en/ai-studio/sdk/

- Yandex AI Studio SDK GitHub:  
  https://github.com/yandex-cloud/yandex-ai-studio-sdk

