### Этап 2. Подготовка окружения

#### Шаг 2.1. Создание виртуального окружения Python

1. Откройте терминал или командную строку.
2. Перейдите в директорию проекта:
```bash
cd /path/to/your/project
```
3. Создайте виртуальное окружение:
```bash
python -m venv venv
```
4. Активируйте виртуальное окружение:
* для Linux/Mac:
```bash
source venv/bin/activate
```
* для Windows:
```cmd
venv\Scripts\activate
```
5. Проверьте активацию (в начале строки приглашения появится `(venv)`).

#### Шаг 2.2. Установка зависимостей

1. Создайте файл `requirements.txt` со следующим содержимым:
```txt
fastapi==0.104.1
uvicorn==0.24.0
sqlalchemy==2.0.23
psycopg2-binary==2.9.9
python-dotenv==1.0.0
transformers==4.35.0
torch==2.1.0
librosa==0.10.1
pydub==0.25.1
numpy==1.24.3
pandas==2.1.3
openpyxl==3.1.2
python-multipart==0.0.6
python-telegram-bot==20.7
slack-sdk==3.21.1
requests==2.31.0
aiohttp==3.9.1
asyncio==3.4.3
scikit-learn==1.3.2
joblib==1.3.2
```
2. Установите зависимости:
```bash
pip install -r requirements.txt
```

**Обоснование выбора библиотек:**
* `fastapi` — для создания API с автоматической документацией;
* `uvicorn` — ASGI‑сервер для запуска FastAPI;
* `sqlalchemy` — ORM для работы с PostgreSQL;
* `psycopg2-binary` — драйвер PostgreSQL для Python;
* `transformers` и `torch` — для моделей NLP и эмоциональной детекции;
* `librosa` и `pydub` — для обработки аудио;
* `python-telegram-bot` и `slack-sdk` — для интеграции с мессенджерами;
* `aiohttp` — асинхронные HTTP‑запросы;
* `scikit-learn` — для кастомных классификаторов эмоций.

#### Шаг 2.3. Настройка PostgreSQL

1. Установите PostgreSQL (локально или через Docker).
2. Создайте базу данных:
```sql
CREATE DATABASE negative_detection;
```
3. Создайте пользователя с правами:
```sql
CREATE USER detector_user WITH PASSWORD 'secure_password_123';
GRANT ALL PRIVILEGES ON DATABASE negative_detection TO detector_user;
```
4. Настройте подключение в файле `.env`:
```env
DB_URL=postgresql://detector_user:secure_password_123@localhost/negative_detection
DB_HOST=localhost
DB_PORT=5432
DB_NAME=negative_detection
```
5. Создайте схему БД (файл `database/schema.sql`):
```sql
-- Таблица событий негативных ситуаций
CREATE TABLE negative_events (
    id SERIAL PRIMARY KEY,
    event_id VARCHAR(50) UNIQUE NOT NULL,
    timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    priority VARCHAR(20),
    type VARCHAR(50),
    operator_id INTEGER,
    call_id VARCHAR(50),
    confidence FLOAT,
    audio_fragment_path VARCHAR(255),
    status VARCHAR(20) DEFAULT 'new'
);

-- Таблица уведомлений
CREATE TABLE notifications (
    id SERIAL PRIMARY KEY,
    event_id VARCHAR(50),
    channel VARCHAR(20),
    sent_at TIMESTAMP,
    read_at TIMESTAMP NULL,
    action_taken VARCHAR(50),
    response_time INTEGER
);

-- Таблица настроек пользователей
CREATE TABLE user_preferences (
    user_id INTEGER PRIMARY KEY,
    preferred_channels JSONB,
    working_hours JSONB,
    notification_thresholds JSONB
);
```
6. Примените схему:
```bash
psql -U detector_user -d negative_detection -f database/schema.sql
```

#### Шаг 2.4. Настройка системы логирования и мониторинга

1. Создайте файл `logging_config.py`:
```python
import logging
from logging.handlers import RotatingFileHandler
import os

def setup_logging():
    # Создаём директорию для логов
    os.makedirs('logs', exist_ok=True)

    logger = logging.getLogger('negative_detector')
    logger.setLevel(logging.INFO)

    # Ротирующий файловый обработчик
    file_handler = RotatingFileHandler(
        'logs/negative_detector.log',
        maxBytes=10*1024*1024,  # 10 МБ
        backupCount=5
    )
    file_formatter = logging.Formatter(
        '%(asctime)s - %(name)s - %(levelname)s - %(message)s'
    )
    file_handler.setFormatter(file_formatter)

    # Консольный обработчик для отладки
    console_handler = logging.StreamHandler()
    console_handler.setLevel(logging.DEBUG)
    console_formatter = logging.Formatter('%(levelname)s: %(message)s')
    console_handler.setFormatter(console_formatter)

    logger.addHandler(file_handler)
    logger.addHandler(console_handler)
    return logger

# Инициализация
logger = setup_logging()
```
2. Настройте мониторинг через Prometheus (файл `monitoring/metrics.py`):
```python
from prometheus_client import Counter, Gauge, Histogram

# Метрики производительности
CALLS_PROCESSED = Counter('calls_processed_total', 'Общее количество обработанных звонков')
NEGATIVE_EVENTS = Counter('negative_events_total', 'Количество негативных событий')
NOTIFICATIONS_SENT = Counter('notifications_sent_total', 'Отправленные уведомления')
RESPONSE_TIME = Histogram('notification_response_time_seconds', 'Время реакции на уведомление')
SYSTEM_LOAD = Gauge('system_load_percentage', 'Загрузка системы')
```
3. Добавьте инициализацию метрик в главный модуль.

---

### Проверка готовности окружения

**Скрипт проверки `verify_setup.py`:**
```python
import importlib.util
import sys

required_packages = [
    'fastapi', 'uvicorn', 'sqlalchemy', 'psycopg2',
    'transformers', 'torch', 'librosa', 'pydub'
]

print("Проверка зависимостей...")
for package in required_packages:
    spec = importlib.util.find_spec(package)
    if spec is None:
        print(f"❌ Пакет {package} не установлен!")
        sys.exit(1)
    else:
        print(f"✅ Пакет {package} установлен")

print("\nПроверка подключения к БД...")
try:
    from sqlalchemy import create_engine
    engine = create_engine(os.getenv('DB_URL'))
    connection = engine.connect()
    print("✅ Подключение к БД успешно")
    connection.close()
except Exception as e:
    print(f"❌ Ошибка подключения к БД: {e}")
    sys.exit(1)

print("\n✅ Окружение готово к разработке!")
```

**Запуск проверки:**
```bash
python verify_setup.py
```

---

### Ожидаемые результаты этапа 2

После завершения этапа вы получите:
* **Активированное виртуальное окружение** с изолированными зависимостями.
* **Установленные библиотеки** для всех компонентов системы.
* **Настроенную БД PostgreSQL** с готовой схемой.
* **Систему логирования** с ротацией файлов.
* **Базовые метрики мониторинга** для отслеживания производительности.
* **Скрипт верификации** для быстрой проверки готовности окружения.
