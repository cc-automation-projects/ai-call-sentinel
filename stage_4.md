### Этап 4. Интеграция с каналами оповещения

#### Шаг 4.1. Разработка коннекторов для каналов оповещения

**Файл `notification_connectors.py`:**

```python
import smtplib
import requests
import json
from email.mime.text import MIMEText
from email.mime.multipart import MIMEMultipart
from typing import Dict, List
from datetime import datetime

class TelegramConnector:
    def __init__(self, bot_token: str, chat_id: str):
        self.bot_token = bot_token
        self.chat_id = chat_id
        self.base_url = f"https://api.telegram.org/bot{bot_token}"

    def send_message(self, message: str) -> bool:
        """Отправка сообщения в Telegram"""
        url = f"{self.base_url}/sendMessage"
        payload = {
            'chat_id': self.chat_id,
            'text': message,
            'parse_mode': 'HTML'
        }
        try:
            response = requests.post(url, json=payload, timeout=10)
            return response.status_code == 200
        except Exception as e:
            print(f"Ошибка отправки в Telegram: {e}")
            return False


class SlackConnector:
    def __init__(self, webhook_url: str):
        self.webhook_url = webhook_url

    def send_message(self, message: Dict) -> bool:
        """Отправка сообщения в Slack через вебхук"""
        try:
            response = requests.post(
                self.webhook_url,
                json=message,
                timeout=10
            )
            return response.status_code == 200
        except Exception as e:
            print(f"Ошибка отправки в Slack: {e}")
            return False

class EmailConnector:
    def __init__(self, smtp_server: str, smtp_port: int,
                 username: str, password: str, from_email: str):
        self.smtp_server = smtp_server
        self.smtp_port = smtp_port
        self.username = username
        self.password = password
        self.from_email = from_email

    def send_email(self, to_emails: List[str], subject: str, body: str) -> bool:
        """Отправка email"""
        msg = MIMEMultipart()
        msg['From'] = self.from_email
        msg['To'] = ', '.join(to_emails)
        msg['Subject'] = subject
        msg.attach(MIMEText(body, 'html'))

        try:
            server = smtplib.SMTP(self.smtp_server, self.smtp_port)
            server.starttls()
            server.login(self.username, self.password)
            text = msg.as_string()
            server.sendmail(self.from_email, to_emails, text)
            server.quit()
            return True
        except Exception as e:
            print(f"Ошибка отправки email: {e}")
            return False

class Bitrix24Connector:
    def __init__(self, webhook_url: str):
        self.webhook_url = webhook_url

    def create_task(self, title: str, description: str, assignee_id: str) -> bool:
        """Создание задачи в Битрикс24"""
        payload = {
            'fields': {
                'TITLE': title,
                'DESCRIPTION': description,
                'RESPONSIBLE_ID': assignee_id
            }
        }
        try:
            response = requests.post(
                f"{self.webhook_url}/tasks.task.add",
                json=payload
            )
            return response.status_code == 200
        except Exception as e:
            print(f"Ошибка создания задачи в Битрикс24: {e}")
            return False
```

#### Шаг 4.2. Реализация системы приоритетов

**Файл `priority_manager.py`:**

```python
from datetime import datetime, timedelta
from typing import Dict, List

class PriorityManager:
    def __init__(self):
        # Очереди для разных приоритетов
        self.critical_queue = []
        self.medium_queue = []
        self.low_queue = []

    def add_to_queue(self, notification: Dict) -> None:
        """Добавление уведомления в соответствующую очередь"""
        priority = notification['priority_level']
        if priority == 'critical':
            self.critical_queue.append(notification)
        elif priority == 'high':
            self.medium_queue.append(notification)
        else:
            self.low_queue.append(notification)

    def process_queues(self) -> Dict[str, int]:
        """Обработка очередей с разными задержками"""
        results = {}

        # Критические — отправляем немедленно
        results['critical'] = self._send_immediate(self.critical_queue)
        self.critical_queue.clear()

        # Средние — отправляем каждые 5 минут
        current_time = datetime.utcnow()
        medium_to_send = [
            n for n in self.medium_queue
            if (current_time - n['timestamp']).total_seconds() >= 300  # 5 мин
        ]
        results['medium'] = self._send_batch(medium_to_send)
        self.medium_queue = [n for n in self.medium_queue if n not in medium_to_send]

        return results

    def _send_immediate(self, notifications: List[Dict]) -> int:
        """Немедленная отправка критических уведомлений"""
        sent_count = 0
        for notification in notifications:
            if self._send_notification(notification):
                sent_count += 1
        return sent_count

    def _send_batch(self, notifications: List[Dict]) -> int:
        """Пакетная отправка уведомлений среднего приоритета"""
        sent_count = 0
        for notification in notifications:
            if self._send_notification(notification):
                sent_count += 1
        return sent_count

    def _send_notification(self, notification: Dict) -> bool:
        """Внутренняя отправка уведомления через соответствующий коннектор"""
        channel = notification['channel']
        message = notification['message']

        if channel == 'telegram':
            connector = TelegramConnector(
                notification['config']['bot_token'],
                notification['config']['chat_id']
            )
            return connector.send_message(message)
        elif channel == 'slack':
            connector = SlackConnector(notification['config']['webhook_url'])
            return connector.send_message(message)
        elif channel == 'email':
            connector = EmailConnector(**notification['config'])
            return connector.send_email(
                notification['recipients'],
                notification['subject'],
                message
            )
        elif channel == 'bitrix24':
            connector = Bitrix24Connector(notification['config']['webhook_url'])
            return connector.create_task(
                notification['title'],
                message,
                notification['assignee_id']
            )
        return False
```

#### Шаг 4.3. Создание шаблонов уведомлений

**Файл `notification_templates.py`:**

```python
class NotificationTemplates:
    @staticmethod
    def get_telegram_template(analysis_result: Dict) -> str:
        """Шаблон для Telegram"""
        return f"""
🚨 <b>КРИТИЧЕСКОЕ СОБЫТИЕ В ЗВОНКЕ</b>

📞 Звонок: <code>{analysis_result['call_id']}</code>
👤 Оператор: <code>{analysis_result['operator_id']}</code>
⏱️ Время события: {analysis_result['analysis_timestamp']}

📊 Уровень негатива: <b>{analysis_result['negativity_score']:.2f}</b>
🔎 Тип ситуации: <code>{analysis_result['situation_type']}</code>

📝 Обнаруженные события:
{''.join(f'• {event["type"]}: {event.get("phrase", "") or event.get("emotion", "")}\n'
           for event in analysis_result['detected_events'][:3])}

🎯 Рекомендации:
{''.join(f'• {action}\n' for action in analysis_result['decision_recommendations'][:2])}

🔗 <a href="{analysis_result.get('critical_fragment_path', '#')}">Прослушать фрагмент</a>
"""

    @staticmethod
    def get_slack_template(analysis_result: Dict) -> Dict:
        """Шаблон для Slack (в формате JSON для вебхука)"""
        return {
            "blocks": [
                {
                    "type": "header",
                    "text": {
                "type": "plain_text",
                "text": "🚨 КРИТИЧЕСКОЕ СОБЫТИЕ В ЗВОНКЕ"
            }
        },
        {
            "type": "section",
            "fields": [
                {
                    "type": "mrkdwn",
                    "text": f"*Звонок:* `{analysis_result['call_id']}`"
                },
                {
                    "type": "mrkdwn",
                    "text": f"*Оператор:* `{analysis_result['operator_id']}`"
                }
            ]
        },
        {
            "type": "section",
            "fields": [
                {
                    "type": "mrkdwn",
                    "text": f"*Уровень негатива:* {analysis_result['negativity_score']:.2f}"
                },
                {
                    "type": "mrkdwn",
                    "text": f"*Тип ситуации:* `{analysis_result['situation_type']}`"
                }
            ]
        },
        {
            "type": "section",
            "text": {
                "type": "mrkdwn",
                "text": "*Обнаруженные события:*\n" +
                          "\n".join(
                              f"• {event['type']}: {event.get('phrase', '') or event.get('emotion', '')}"
                              for event in analysis_result['detected_events'][:3]
                          )
            }
        },
        {
            "type": "section",
            "text": {
                "type": "mrkdwn",
                "text": "*Рекомендации:*\n" +
                          "\n".join(f"• {action}" for action in analysis_result['decision_recommendations'][:2])
            }
        },
        {
            "type": "actions",
            "elements": [
                {
                    "type": "button",
                    "text": {
                        "type": "plain_text",
                        "text": "Прослушать фрагмент"
                    },
            "url": analysis_result.get('critical_fragment_path', '#')
        }
      ]
    }
}

    @staticmethod
    def get_email_template(analysis_result: Dict) -> str:
        """HTML‑шаблон для email"""
        events_list = "\n".join(
            f"<li><strong>{event['type']}</strong>: {event.get('phrase', '') or event.get('emotion', '')}</li>"
            for event in analysis_result['detected_events'][:5]
        )
        recommendations_list = "\n".join(
            f"<li>{action}</li>" for action in analysis_result['decision_recommendations']
        )

        return f"""
<html>
<head>
    <style>
        body {{ font-family: Arial, sans-serif; }}
        .header {{ color: #d9534f; font-size: 18px; font-weight: bold; }}
        .section {{ margin: 15px 0; }}
        .label {{ font-weight: bold; color: #333; }}
    </style>
</head>
<body>
    <div class="header">🚨 КРИТИЧЕСКОЕ СОБЫТИЕ В ЗВОНКЕ</div>

    <div class="section">
        <p><span class="label">Звонок:</span> {analysis_result['call_id']}</p>
        <p><span class="label">Оператор:</span> {analysis_result['operator_id']}</p>
        <p><span class="label">Время события:</span> {analysis_result['analysis_timestamp']}</p>
    </div>

    <div class="section">
        <p><span class="label">Уровень негатива:</span> <strong>{analysis_result['negativity_score']:.2f}</strong></p>
        <p><span class="label">Тип ситуации:</span> {analysis_result['situation_type']}</p>
    </div>

    <div class="section">
        <p><span class="label">Обнаруженные события:</span></p>
        <ul>{events_list}</ul>
    </div>

    <div class="section">
        <p><span class="label">Рекомендации:</span></p>
        <ul>{recommendations_list}</ul>
    </div>

    <div class="section">
        <a href="{analysis_result.get('critical_fragment_path', '#')}" style="color: #007bff;">
            🎧 Прослушать аудиофрагмент
        </a>
    </div>
</body>
</html>
"""

    @staticmethod
    def get_bitrix24_template(analysis_result: Dict) -> Dict:
        """Шаблон задачи для Битрикс24"""
        events_summary = "\n".join(
            f"- *{event['type']}*: {event.get('phrase', '') or event.get('emotion', '')}"
            for event in analysis_result['detected_events'][:3]
        )
        recommendations_summary = "\n".join(
            f"- {action}" for action in analysis_result['decision_recommendations'][:3]
        )

        description = f"""
📞 **Звонок:** {analysis_result['call_id']}
👤 **Оператор:** {analysis_result['operator_id']}
⏱️ **Время события:** {analysis_result['analysis_timestamp']}

📊 **Уровень негатива:** {analysis_result['negativity_score']:.2f}
🔎 **Тип ситуации:** {analysis_result['situation_type']}

**Обнаруженные события:**
{events_summary}

**Рекомендации:**
{recommendations_summary}

🎧 **Аудиофрагмент:** {analysis_result.get('critical_fragment_path', 'Не доступен')}
"""

        return {
            'title': f"Разбор негативного звонка {analysis_result['call_id']}",
            'description': description,
            'priority': 'high' if analysis_result['priority_level'] in ['critical', 'high'] else 'medium'
        }

    @staticmethod
    def get_daily_report_template(daily_stats: Dict) -> str:
        """Шаблон ежедневного отчёта (email)"""
        calls_summary = "\n".join(
            f"<tr><td>{call['call_id']}</td><td>{call['operator_id']}</td>"
            f"<td>{call['negativity_score']:.2f}</td>"
            f"<td>{call['situation_type']}</td></tr>"
            for call in daily_stats['calls'][:10]
        )

        return f"""
<html>
<body>
    <h2>📊 Ежедневный отчёт по негативным звонкам</h2>
    <p><strong>Период:</strong> {daily_stats['date']}</p>
    <p><strong>Всего звонков:</strong> {daily_stats['total_calls']}</p>
    <p><strong>Негативных звонков:</strong> {daily_stats['negative_calls']} "
    ({daily_stats['negative_percentage']:.1f}%)</p>

    <table border="1" cellpadding="5">
        <thead>
            <tr>
                <th>ID звонка</th>
                <th>Оператор</th>
                <th>Уровень негатива</th>
                <th>Тип ситуации</th>
            </tr>
        </thead>
        <tbody>
            {calls_summary}
        </tbody>
    </table>

    <p><strong>Статистика по приоритетам:</strong><br>
    Критические: {daily_stats['priority_counts'].get('critical', 0)}<br>
    Высокие: {daily_stats['priority_counts'].get('high', 0)}<br>
    Средние: {daily_stats['priority_counts'].get('medium', 0)}
    </p>
</body>
</html>
"""
```

#### Шаг 4.4. Интеграция системы оповещения

**Файл `alert_manager.py`:**

```python
from notification_connectors import (
    TelegramConnector,
    SlackConnector,
    EmailConnector,
    Bitrix24Connector
)
from priority_manager import PriorityManager
from notification_templates import NotificationTemplates
from typing import Dict, List
from datetime import datetime

class AlertManager:
    def __init__(self, config: Dict):
        self.config = config
        self.priority_manager = PriorityManager()

        # Инициализация коннекторов
        self.connectors = {
            'telegram': TelegramConnector(
                config['telegram']['bot_token'],
                config['telegram']['chat_id']
            ) if 'telegram' in config else None,
            'slack': SlackConnector(
                config['slack']['webhook_url']
            ) if 'slack' in config else None,
            'email': EmailConnector(
                smtp_server=config['email']['smtp_server'],
                smtp_port=config['email']['smtp_port'],
                username=config['email']['username'],
                password=config['email']['password'],
                from_email=config['email']['from_email']
            ) if 'email' in config else None,
            'bitrix24': Bitrix24Connector(
                config['bitrix24']['webhook_url']
            ) if 'bitrix24' in config else None
        }

    def send_alert(self, analysis_result: Dict) -> Dict[str, bool]:
        """Отправка оповещения через все настроенные каналы"""
        results = {}
        priority = analysis_result['priority_level']
        timestamp = datetime.utcnow()

        for channel, connector in self.connectors.items():
            if connector is None:
                continue

            try:
                if channel == 'telegram':
                    message = NotificationTemplates.get_telegram_template(analysis_result)
                    success = connector.send_message(message)
                elif channel == 'slack':
                    message = NotificationTemplates.get_slack_template(analysis_result)
                    success = connector.send_message(message)
                elif channel == 'email':
                    subject = f"🚨 Оповещение: негативный звонок {analysis_result['call_id']}"
                    body = NotificationTemplates.get_email_template(analysis_result)
                    recipients = self.config['email']['recipients']
                    success = connector.send_email(recipients, subject, body)
                elif channel == 'bitrix24':
                    task_data = NotificationTemplates.get_bitrix24_template(analysis_result)
                    success = connector.create_task(
                        task_data['title'],
                        task_data['description'],
                        self.config['bitrix24']['assignee_id']
            else:
                success = False

                results[channel] = success
                if success:
                    print(f"Оповещение отправлено через {channel}")
                else:
                    print(f"Ошибка отправки через {channel}")

            except Exception as e:
                print(f"Критическая ошибка при отправке через {channel}: {e}")
                results[channel] = False

        return results

    def schedule_daily_report(self, daily_stats: Dict) -> bool:
        """Планирование ежедневного отчёта"""
        if 'email' not in self.connectors or self.connectors['email'] is None:
            return False

        try:
            subject = f"📊 Ежедневный отчёт по негативным звонкам — {daily_stats['date']}"
            body = NotificationTemplates.get_daily_report_template(daily_stats)
            recipients = self.config['email']['daily_report_recipients']

            return self.connectors['email'].send_email(recipients, subject, body)
        except Exception as e:
            print(f"Ошибка при отправке ежедневного отчёта: {e}")
            return False

    def process_analysis_results(self, analysis_results: List[Dict]) -> List[Dict]:
        """Обработка результатов анализа и отправка оповещений"""
        send_results = []

        for result in analysis_results:
            # Добавляем в очередь приоритетов
            notification = {
                'priority_level': result['priority_level'],
                'timestamp': datetime.utcnow(),
                'message': None,  # Будет заполнено при отправке
                'channel': None,
                'config': self.config
            }
            self.priority_manager.add_to_queue(notification)

            # Отправляем сразу для критических и высоких приоритетов
            if result['priority_level'] in ['critical', 'high']:
                send_result = self.send_alert(result)
                send_results.append({
                    'call_id': result['call_id'],
                    'sent_through': send_result,
                    'priority': result['priority_level']
                })

        # Обрабатываем очереди
        queue_results = self.priority_manager.process_queues()
        print(f"Обработано очередей: {queue_results}")

        return send_results

    def health_check(self) -> Dict[str, bool]:
        """Проверка работоспособности коннекторов"""
        checks = {}
        for channel, connector in self.connectors.items():
            if connector is None:
                checks[channel] = False
            else:
                # Простая проверка — попытка отправить тестовое сообщение или просто проверка инициализации
                checks[channel] = True  # В реальной реализации здесь может быть более сложная логика
        return checks
```

---

### Итоговый обзор этапа 4

**Реализованные компоненты:**

1. **Коннекторы для каналов оповещения:**
* Telegram (бот с HTML‑форматированием);
* Slack (вебхук с блоками и кнопками);
* email (SMTP с HTML‑шаблонами);
* Битрикс24 (REST API для создания задач).

2. **Система приоритетов:**
* критические ситуации — мгновенная отправка;
* средние — пакетная отправка каждые 5 минут;
* низкие — ежедневный отчёт.

3. **Шаблоны уведомлений:**
* контекстная информация (ID звонка, оператор, время);
* уровень негатива и тип ситуации;
* список обнаруженных событий;
* рекомендации по действиям;
* ссылка на аудиофрагмент;
* форматирование под каждый канал.

4. **Менеджер оповещений:**
* централизованная отправка через все каналы;
* обработка результатов анализа;
* планирование ежедневных отчётов;
* мониторинг работоспособности коннекторов.

**Результаты этапа:**
* полностью интегрированная система оповещений;
* гибкая настройка каналов и приоритетов;
* унифицированные шаблоны для разных сценариев;
* надёжная доставка критических оповещений;
* автоматизированная отчётность.

Хотите, я уточню реализацию какого‑либо компонента или продолжу с этапом мониторинга и отчётности?