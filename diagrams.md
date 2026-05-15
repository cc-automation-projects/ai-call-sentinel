### Диаграммы в формате PlantUML

Диаграммы в формате PlantUML для описания системы выявления негативных ситуаций.

**Диаграмма 1. Общая архитектура системы**
```plantuml
@startuml
!theme plain
title Архитектура системы выявления негативных ситуаций

package "Система" {
  [Аудиоисточник] as audio_source
  [Модуль обработки аудио] as audio_processor
  [ASR-модель] as asr
  [Детектор эмоций] as emotion_detector
  [Анализатор ключевых слов] as keyword_analyzer
  [Механизм принятия решений] as decision_engine
  [Хранилище данных] as storage
  [API Gateway] as api
  [Дашборд мониторинга] as dashboard
  [Система оповещений] as alert_system
}

audio_source --> audio_processor
audio_processor --> asr
audio_processor --> emotion_detector
audio_processor --> keyword_analyzer
asr --> decision_engine
emotion_detector --> decision_engine
keyword_analyzer --> decision_engine
decision_engine --> storage
decision_engine --> alert_system
storage --> api
api --> dashboard
alert_system --> [Telegram]
alert_system --> [Slack]
alert_system --> [Битрикс24]
@enduml
```

**Диаграмма 2. Поток данных при обработке звонка**
```plantuml
@startuml
title Поток данных: обработка звонка и детекция негативных ситуаций

actor "Колл‑центр" as call_center
participant "Аудиопоток" as audio
participant "Предварительная обработка" as preprocess
participant "Транскрибация" as transcription
participant "Анализ эмоций" as emotion
participant "Поиск ключевых слов" as keywords
participant "Принятие решения" as decision
participant "Оповещение" as alert
participant "Дашборд" as dashboard

call_center -> audio: Аудиопоток звонка
audio -> preprocess: RAW аудио
preprocess -> transcription: Очищенный аудиосигнал
transcription -> emotion: Текст разговора
transcription -> keywords: Текст разговора
emotion -> decision: Уровень негатива (0–1)
keywords -> decision: Список триггеров
decision -> alert: Событие детекции
decision -> dashboard: Статистика
alert -> [Telegram]: Уведомление
alert -> [Slack]: Уведомление
@enduml
```

**Диаграмма 3. Компоненты ядра анализа**
```plantuml
@startuml
title Ядро анализа: компоненты и взаимодействие

package "Ядро анализа" {
  [Буфер аудио] as audio_buffer
  [Нормализатор громкости] as volume_norm
  [Шумоподавитель] as noise_reduction
  [Разделитель каналов] as channel_split
  [Транскрибатор] as transcriber
  [Детектор эмоций] as emotion_det
  [Анализатор фраз] as phrase_analyzer
  [Агрегатор сигналов] as aggregator
  [Триггер записи] as recorder_trigger
}

audio_buffer --> volume_norm
volume_norm --> noise_reduction
noise_reduction --> channel_split
channel_split --> transcriber
channel_split --> emotion_det
transcriber --> phrase_analyzer
phrase_analyzer --> aggregator
emotion_det --> aggregator
aggregator --> recorder_trigger
@enduml
```

**Диаграмма 4. Система оповещений**
```plantuml
@startuml
title Система оповещений: каналы и приоритеты

package "Оповещения" {
  [Менеджер приоритетов] as priority_mgr
  [Telegram Connector] as telegram
  [Slack Connector] as slack
  [Email Connector] as email
  [Битрикс24 Connector] as bitrix
  [UI Highlighter] as ui_highlight
}

[Механизм принятия решений] --> priority_mgr: Событие + Приоритет
priority_mgr --> telegram: Критический приоритет
priority_mgr --> slack: Средний приоритет
priority_mgr --> email: Низкий приоритет (отчёт)
priority_mgr --> bitrix: Критический + Средний
priority_mgr --> ui_highlight: Все события
@enduml
```

**Диаграмма 5. Структура базы данных**
```plantuml
@startuml
title Схема базы данных

entity "Calls" as calls {
  * call_id : UUID
  start_time : DateTime
  end_time : DateTime
  operator_id : String
  client_id : String
}

entity "Events" as events {
  * event_id : UUID
  call_id : UUID
  timestamp : DateTime
  type : String
  severity : Integer
  negativity_score : Float
}

entity "Audio Fragments" as fragments {
  * fragment_id : UUID
  event_id : UUID
  audio_data : ByteArray
  duration : Integer
}

entity "Alerts" as alerts {
  * alert_id : UUID
  event_id : UUID
  channel : String
  sent_at : DateTime
  status : String
}

calls ||--o{ events
events ||--o{ fragments
events ||--o{ alerts
@enduml
```

**Диаграмма 6. API Endpoints**
```plantuml
@startuml
title API Endpoints системы

node "API Gateway" as api {
  component "POST /analyze" as analyze
  component "GET /events" as get_events
  component "GET /dashboard" as dashboard
  component "POST /alerts" as send_alerts
  component "GET /health" as health
}

cloud "Внешние системы" as external {
  [Колл‑центр] as call_center
  [BI‑система] as bi
  [Супервизор] as supervisor
}

call_center --> analyze: Аудиофайл
bi --> get_events: Запрос статистики
supervisor --> dashboard: Просмотр дашборда
api --> health: Проверка здоровья
@enduml
```

**Диаграмма 7. Развёртывание (Docker Compose)**
```plantuml
@startuml
title Инфраструктура развёртывания

node "Docker Host" as host {
  component "Analyzer Service" as analyzer
  component "Dashboard Service" as dashboard
  component "PostgreSQL" as db
  component "Redis" as redis
  component "Prometheus" as prometheus
  component "Grafana" as grafana
}

analyzer --> db: Хранит события
analyzer --> redis: Очереди обработки
dashboard --> analyzer: Запросы API
prometheus --> analyzer: Сбор метрик
grafana --> prometheus: Визуализация
@enduml
```

**Диаграмма 8. Мониторинг и оповещения о сбоях**
```plantuml
@startuml
title Мониторинг системы и оповещения о сбоях

component "Prometheus" as prom
component "Alert Manager" as alert_mgr
component "Telegram" as telegram
component "Email" as email

component "Analyzer" as analyzer
component "Dashboard" as dashboard
component "Database" as db

analyzer --> prom: Метрики производительности
dashboard --> prom: Метрики UI
db --> prom: Метрики БД
prom --> alert_mgr: Алерты
alert_mgr --> telegram: Сбои сервисов
alert_mgr --> email: Предупреждения
@enduml
```

**Диаграмма 9. Процесс пилотного внедрения**
```plantuml
@startuml
title Процесс пилотного внедрения

start
:Запуск на 5–10 операторах;
:Сбор обратной связи от супервизоров;
:Корректировка порогов детекции;
:Оптимизация производительности;
if (Результаты удовлетворительны?) then (да)
  :Масштабирование на весь колл‑центр;
else (нет)
  :Возврат к корректировке параметров;
endif
stop
@enduml
```

**Диаграмма 10. Взаимодействие пользователей с системой**
```plantuml
@startuml
title Роли пользователей и их взаимодействие

actor "Оператор колл‑центра" as operator
actor "Супервизор" as supervisor
actor "Администратор" as admin

rectangle "Система" as system {
  rectangle "Обработка звонков" as call_processing
  rectangle "Дашборд мониторинга" as monitoring
  rectangle "Настройка правил" as rule_config
}

operator --> call_processing: Совершает звонки
supervisor --> monitoring: Отслеживает негативные ситуации
admin --> rule_config: Настраивает правила детекции
monitoring --> supervisor: Оповещения о конфликтах
rule_config --> call_processing: Обновлённые правила
@enduml
```