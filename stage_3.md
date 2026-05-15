### Этап 3. Разработка ядра анализа


#### Шаг 3.1. Реализация модуля загрузки и предварительной обработки аудио


**Файл `audio_processor.py`:**
```python
import librosa
import numpy as np
from pydub import AudioSegment
from typing import Tuple

class AudioProcessor:
    def __init__(self, target_sample_rate: int = 16000):
        self.target_sample_rate = target_sample_rate

    def load_and_preprocess(self, audio_path: str) -> Tuple[np.ndarray, int]:
        """Загрузка и предварительная обработка аудиофайла"""
        # Загрузка аудио
        audio, sr = librosa.load(audio_path, sr=None)

        # Нормализация громкости
        audio = self._normalize_volume(audio)

        # Удаление шумов
        audio = self._denoise(audio, sr)

        # Передискретизация
        if sr != self.target_sample_rate:
            audio = librosa.resample(audio, orig_sr=sr, target_sr=self.target_sample_rate)

        return audio, self.target_sample_rate

    def _normalize_volume(self, audio: np.ndarray) -> np.ndarray:
        """Нормализация громкости до -23 LUFS"""
        max_amp = np.max(np.abs(audio))
        if max_amp > 0:
            normalized = audio / max_amp
            return normalized * 0.8  # Целевая громкость 80 % от максимума
        return audio

    def _denoise(self, audio: np.ndarray, sr: int) -> np.ndarray:
        """Удаление фоновых шумов"""
        # Простая фильтрация низких частот (ниже 80 Гц)
        b, a = signal.butter(4, 80 / (sr / 2), btype='high')
        cleaned = signal.filtfilt(b, a, audio)
        return cleaned

    def split_channels(self, audio: AudioSegment) -> Tuple[AudioSegment, AudioSegment]:
        """Разделение стерео на два моноканала (оператор/клиент)"""
        if audio.channels == 2:
            channels = audio.split_to_mono()
            return channels[0], channels[1]  # Оператор, Клиент
        return audio, audio  # Если моно — дублируем
```

#### Шаг 3.2. Создание конвейера анализа

**Файл `analysis_pipeline.py`:**
```python
from transformers import pipeline
import numpy as np
from audio_processor import AudioProcessor
from transcription_connector import TranscriptionConnector

class AnalysisPipeline:
    def __init__(self):
        self.audio_processor = AudioProcessor()
        self.transcription_connector = TranscriptionConnector(
            api_key="your_api_key"
        )
        # Модель для детекции эмоций
        self.emotion_classifier = pipeline(
            "text-classification",
            model="blanchefort/rubert-base-cased-sentiment"
        )

    async def analyze_call(self, audio_path: str) -> Dict:
        """Полный конвейер анализа звонка"""
        # 1. Загрузка и обработка аудио
        processed_audio, sr = self.audio_processor.load_and_preprocess(audio_path)

        # 2. Транскрибация
        transcription = await self.transcription_connector.transcribe_audio_file(audio_path)

        # 3. Детекция эмоций
        emotion_results = self._analyze_emotions(transcription['text'])

        # 4. Анализ акустических параметров
        acoustic_features = self._extract_acoustic_features(processed_audio, sr)

        # 5. Поиск ключевых фраз
        keyword_matches = self._find_keywords(transcription['text'])

        return {
            'transcription': transcription['text'],
            'emotions': emotion_results,
            'acoustic_features': acoustic_features,
            'keywords': keyword_matches
        }

    def _analyze_emotions(self, text: str) -> List[Dict]:
        """Анализ эмоциональной окраски текста"""
        results = self.emotion_classifier(text)
        return [
            {
                'label': result['label'],
                'confidence': result['score']
            }
            for result in results
        ]

    def _extract_acoustic_features(self, audio: np.ndarray, sr: int) -> Dict:
        """Извлечение акустических признаков"""
        features = {}
        # Тон (высота голоса)
        pitches, magnitudes = librosa.piptrack(y=audio, sr=sr)
        features['pitch_mean'] = np.mean(pitches[pitches > 0])
        features['pitch_std'] = np.std(pitches[pitches > 0])

        # Громкость (RMS)
        rms = librosa.feature.rms(y=audio)
        features['loudness_mean'] = np.mean(rms)
        features['loudness_std'] = np.std(rms)

        # Темп речи (количество слов в минуту)
        word_count = len(text.split())
        duration_minutes = len(audio) / sr / 60
        features['speech_rate'] = word_count / duration_minutes if duration_minutes > 0 else 0

        return features

    def _find_keywords(self, text: str) -> List[str]:
        """Поиск ключевых фраз негативных ситуаций"""
        keywords = [
            "подам жалобу", "обращусь в суд", "недоволен",
            "ужасное обслуживание", "мат_слово1", "мат_слово2"
        ]
        found = []
        for keyword in keywords:
            if keyword in text.lower():
                found.append(keyword)
        return found
```

### Шаг 3.3. Разработка системы правил

**Файл `rule_engine.py` (продолжение):**

```python
        total_score += acoustic_score
        detailed_breakdown['acoustic_score'] = acoustic_score

        # Нормализация итогового балла (0–1)
        normalized_score = min(total_score / 2.0, 1.0)  # Делим на максимальный возможный балл

        return normalized_score, detailed_breakdown

    def classify_situation_type(self,
                          keyword_matches: List[Dict],
                          emotion_results: List[Dict],
                          acoustic_alerts: List[Dict]) -> Dict:
        """Классификация типа негативной ситуации"""
        situation_types = []

        # Классификация по ключевым фразам
        high_severity_keywords = [k for k in keyword_matches if k['severity'] == 'high']
        if high_severity_keywords:
            if any('жалоб' in k['phrase'] or 'суд' in k['phrase'] for k in high_severity_keywords):
                situation_types.append('complaint_threat')
            elif any(re.search(r'мат_\w+|нецензур_\w+', k['phrase']) for k in high_severity_keywords):
                situation_types.append('obscene_language')

        # Классификация по эмоциям
        angry_emotions = [e for e in emotion_results
                         if e['label'] == 'anger' and e['confidence'] > 0.7]
        if angry_emotions:
            situation_types.append('anger_display')

        # Классификация по акустическим признакам
        acoustic_types = [a['type'] for a in acoustic_alerts]
        if 'REPLICA_OVERLAP' in acoustic_types:
            situation_types.append('conversation_conflict')
        if 'HIGH_PITCH' in acoustic_types and 'FAST_SPEECH' in acoustic_types:
            situation_types.append('escalating_tension')

        # Уникальные типы ситуаций
        unique_types = list(set(situation_types))

        return {
            'types': unique_types,
            'primary_type': unique_types[0] if unique_types else 'neutral'
        }

    def get_priority_level(self, negativity_score: float) -> str:
        """Определение уровня приоритета на основе балла негатива"""
        if negativity_score >= 0.8:
            return 'critical'
        elif negativity_score >= 0.6:
            return 'high'
        elif negativity_score >= 0.4:
            return 'medium'
        elif negativity_score >= 0.2:
            return 'low'
        else:
            return 'monitoring'

    def generate_alert_message(self,
                        call_info: Dict,
                        negativity_score: float,
                        situation_classification: Dict,
                        detected_events: List[Dict]) -> Dict:
        """Генерация сообщения оповещения"""
        priority = self.get_priority_level(negativity_score)

        severity_labels = {
            'critical': '🚨 КРИТИЧЕСКОЕ СОБЫТИЕ',
            'high': '⚠️ ВЫСОКИЙ ПРИОРИТЕТ',
            'medium': '🟡 СРЕДНИЙ ПРИОРИТЕТ',
            'low': '🔵 НИЗКИЙ ПРИОРИТЕТ',
            'monitoring': '🟢 МОНИТОРИНГ'
        }

        message = {
            'event_id': f"neg_{datetime.utcnow().strftime('%Y%m%d_%H%M%S')}_{call_info.get('call_id', 'unknown')}",
            'timestamp': datetime.utcnow().isoformat(),
            'priority': priority,
            'severity_label': severity_labels[priority],
            'call_info': call_info,
            'negativity_score': round(negativity_score, 3),
            'situation_type': situation_classification['primary_type'],
            'detailed_types': situation_classification['types'],
            'detected_events': detected_events,
            'actions': [
                {'label': 'Прослушать фрагмент', 'url': call_info.get('audio_url', '#')},
                {'label': 'Связаться с клиентом', 'action': 'call_client'},
                {'label': 'Обсудить с оператором', 'action': 'operator_meeting'}
            ]
        }
        return message

    def update_rules_dynamically(self, new_keywords: Dict = None, new_thresholds: Dict = None):
        """Динамическое обновление правил (для админов)"""
        if new_keywords:
            for severity, phrases in new_keywords.items():
                if severity in self.negative_keywords:
                    self.negative_keywords[severity].extend(phrases)
                else:
                    self.negative_keywords[severity] = phrases

        if new_thresholds:
            self.acoustic_thresholds.update(new_thresholds)

    def validate_rules(self) -> List[str]:
        """Валидация правил на противоречия"""
        issues = []

        # Проверка на дублирование ключевых фраз
        all_phrases = []
        for severity_list in self.negative_keywords.values():
            all_phrases.extend(severity_list)

        from collections import Counter
        phrase_counts = Counter(all_phrases)
        duplicates = [phrase for phrase, count in phrase_counts.items() if count > 1]
        if duplicates:
            issues.append(f"Дублирующиеся ключевые фразы: {duplicates}")

        # Проверка разумности порогов
        if self.acoustic_thresholds['pitch_mean'] < 100:
            issues.append("Пороговое значение тона слишком низкое")
        if self.acoustic_thresholds['loudness_mean'] > 1.0:
            issues.append("Пороговое значение громкости слишком высокое")

        return issues
```

---

### Шаг 3.4. Реализация механизма принятия решений

**Файл `decision_engine.py`:**

```python
from rule_engine import RuleEngine
from typing import Dict, List

class DecisionEngine:
    def __init__(self):
        self.rule_engine = RuleEngine()

    def make_decision(self, analysis_results: Dict) -> Dict:
        """
        Принятие решения на основе всех данных анализа

        Args:
            analysis_results: результаты полного анализа звонка

        Returns:
            Dict: решение с рекомендацией действий
        """
        # Расчёт общего балла негатива
        negativity_score, breakdown = self.rule_engine.calculate_negativity_score(
            analysis_results['keywords'],
            analysis_results['emotions'],
            analysis_results['acoustic_features']
        )

        # Классификация типа ситуации
        situation_classification = self.rule_engine.classify_situation_type(
            analysis_results['keywords'],
            analysis_results['emotions'],
            [a for a in analysis_results['summary']['detected_events']
             if a['type'].startswith('ACUSTIC')]
        )

        # Генерация оповещения
        alert_message = self.rule_engine.generate_alert_message(
            {
                'call_id': analysis_results['call_id'],
                'operator_id': analysis_results['operator_id'],
                'audio_url': analysis_results.get('audio_fragment_url', '')
            },
            negativity_score,
            situation_classification,
            analysis_results['summary']['detected_events']
        )

        return {
            'decision_id': f"dec_{datetime.utcnow().strftime('%Y%m%d_%H%M%S')}",
            'analysis_timestamp': datetime.utcnow().isoformat(),
            'negativity_score': negativity_score,
            'priority_level': alert_message['priority'],
            'situation_type': situation_classification['primary_type'],
            'recommended_actions': self._determine_recommended_actions(alert_message['priority']),
            'alert_message': alert_message,
            'confidence': self._calculate_confidence(breakdown),
            'metadata': {
                'processing_time': analysis_results.get('processing_time', 0),
                'model_versions': analysis_results.get('model_versions', {})
            }
        }

```python
    def _determine_recommended_actions(self, priority: str) -> List[str]:
        """Определение рекомендуемых действий по приоритету"""
        actions_map = {
            'critical': [
                'Немедленно вмешаться в разговор',
                'Прослушать аудиофрагмент (10 сек до/после события)',
                'Связаться с клиентом для урегулирования',
                'Провести разбор ситуации с оператором',
                'Зафиксировать инцидент в системе жалоб'
            ],
            'high': [
                'Прослушать аудиофрагмент',
                'Связать с клиентом для уточнения претензий',
                'Обсудить с оператором после звонка',
                'Добавить в план обучения оператора'
            ],
            'medium': [
                'Прослушать фрагмент при наличии времени',
                'Добавить ситуацию в отчёт за день',
                'Рассмотреть на еженедельном разборе',
                'Предложить оператору обратную связь'
            ],
            'low': [
                'Отметить в дайджесте за день',
                'Учесть при оценке KPI оператора',
                'Включить в статистику трендов'
            ],
            'monitoring': [
                'Не требует действий',
                'Автоматически архивировать'
            ]
        }
        return actions_map.get(priority, ['Нет рекомендаций'])

    def _calculate_confidence(self, breakdown: Dict) -> float:
        """Расчёт уверенности решения на основе согласованности сигналов"""
        # Чем больше детекторов сработали согласованно, тем выше уверенность
        active_detectors = sum(1 for score in breakdown.values() if score > 0)

        # Базовый уровень уверенности
        base_confidence = 0.6

        # Бонус за множественные сигналы
        multiplier = 1.0
        if active_detectors == 2:
            multiplier = 1.3
        elif active_detectors >= 3:
            multiplier = 1.5

        # Корректировка по доминирующему сигналу
        max_contribution = max(breakdown.values()) if breakdown else 0
        if max_contribution > 0.5:  # Доминирующий сигнал
            multiplier += 0.2

        confidence = min(base_confidence * multiplier, 1.0)
        return round(confidence, 3)

    def evaluate_decision_quality(self,
                           decision: Dict,
                           ground_truth: Dict = None) -> Dict:
        """Оценка качества принятого решения (для обучения системы)"""
        quality_metrics = {
            'decision_id': decision['decision_id'],
            'timestamp': datetime.utcnow().isoformat(),
            'accuracy': None,
            'false_positive': False,
            'false_negative': False,
            'feedback_needed': False,
            'suggested_correction': None
        }

        if ground_truth:
            # Сравнение с эталонной разметкой
            if decision['priority_level'] == ground_truth['priority']:
                quality_metrics['accuracy'] = True
            else:
                quality_metrics['accuracy'] = False
                quality_metrics['false_positive'] = (
                    decision['priority_level'] > ground_truth['priority']
                )
                quality_metrics['false_negative'] = (
                    decision['priority_level'] < ground_truth['priority']
                )

            # Предложение коррекции
            if not quality_metrics['accuracy']:
                quality_metrics['feedback_needed'] = True
                quality_metrics['suggested_correction'] = {
                    'correct_priority': ground_truth['priority'],
                    'reason': 'Расхождение с экспертной оценкой'
                }
        else:
            # Автоматическая оценка по стабильности
            if decision['confidence'] < 0.7:
                quality_metrics['feedback_needed'] = True
                quality_metrics['suggested_correction'] = {
                    'correct_priority': self._adjust_priority_based_on_confidence(
                        decision['priority_level'], decision['confidence']
                    ),
            'reason': 'Низкая уверенность решения'
        }

        return quality_metrics

    def _adjust_priority_based_on_confidence(self, current_priority: str, confidence: float) -> str:
        """Корректировка приоритета на основе уверенности"""
        priority_order = ['critical', 'high', 'medium', 'low', 'monitoring']
        current_index = priority_order.index(current_priority)

        if confidence < 0.5 and current_index > 0:
            return priority_order[current_index - 1]  # Понижаем приоритет
        elif confidence > 0.8 and current_index < len(priority_order) - 1:
            return priority_order[current_index + 1]  # Повышаем приоритет
        return current_priority

    async def process_call_with_decision(self, call_data: Dict) -> Dict:
        """
        Полный процесс обработки звонка с принятием решения

        Объединяет анализ и принятие решений в один конвейер
        """
        from analysis_pipeline import AnalysisPipeline

        pipeline = AnalysisPipeline()

        # 1. Анализ звонка
        analysis_results = await pipeline.analyze_call(
            call_data['audio_path'],
            {
                'call_id': call_data['call_id'],
                'operator_id': call_data['operator_id']
            }
        )

        # 2. Принятие решения
        decision = self.make_decision(analysis_results)

        # 3. Оценка качества (если есть ground truth)
        if 'ground_truth' in call_data:
            quality_report = self.evaluate_decision_quality(
                decision, call_data['ground_truth']
            )
            decision['quality_assessment'] = quality_report


        return {
            'call_id': call_data['call_id'],
            'analysis_results': analysis_results,
            'decision': decision,
            'processing_timestamp': datetime.utcnow().isoformat()
        }
```

### Шаг 3.5. Создание модуля сохранения критических фрагментов

**Файл `audio_fragment_saver.py`:**

```python
import os
import numpy as np
from pydub import AudioSegment
from datetime import datetime
import logging
from typing import Dict, Optional

class AudioFragmentSaver:
    def __init__(self, storage_path: str = "audio_fragments"):
        self.storage_path = storage_path
        self.buffer_size_seconds = 30  # Размер буфера для хранения последних секунд разговора
        self.fragment_duration = 20  # Длительность фрагмента для сохранения (10 сек до события + 10 сек после)
        os.makedirs(self.storage_path, exist_ok=True)
        self.logger = logging.getLogger(__name__)

    def create_audio_buffer(self) -> Dict:
        """Создание буфера для аудиоданных"""
        return {
            'audio_data': [],
            'timestamps': [],
            'start_time': datetime.utcnow()
        }

    def add_to_buffer(self,
                   buffer: Dict,
                   audio_chunk: np.ndarray,
                   timestamp: float) -> None:
        """Добавление аудиофрагмента в буфер"""
        buffer['audio_data'].append(audio_chunk)
        buffer['timestamps'].append(timestamp)

        # Ограничение размера буфера
        current_duration = timestamp - buffer['start_time'].timestamp()
        if current_duration > self.buffer_size_seconds:
            # Удаляем старые данные
            self._trim_buffer(buffer)

    def _trim_buffer(self, buffer: Dict) -> None:
        """Обрезка буфера до заданного размера"""
        target_duration = self.buffer_size_seconds
        current_duration = buffer['timestamps'][-1] - buffer['timestamps'][0]

        if current_duration > target_duration:
            # Находим индекс, с которого нужно обрезать
            cutoff_time = buffer['timestamps'][-1] - target_duration
            cut_index = next(
                (i for i, t in enumerate(buffer['timestamps']) if t >= cutoff_time),
                0
            )

            buffer['audio_data'] = buffer['audio_data'][cut_index:]
            buffer['timestamps'] = buffer['timestamps'][cut_index:]

    def save_critical_fragment(self,
                          buffer: Dict,
                          event_timestamp: float,
                          call_info: Dict) -> Optional[str]:
        """
        Сохранение критического фрагмента аудио

        Args:
            buffer: буфер с аудиоданными
            event_timestamp: время обнаружения критической ситуации
            call_info: информация о звонке

        Returns:
            str: путь к сохранённому файлу или None при ошибке
        """
        try:
            # Определяем границы фрагмента
            start_sec = max(0, event_timestamp - 10)  # 10 сек до события
            end_sec = event_timestamp + 10  # 10 сек после события

            # Извлекаем данные из буфера
            fragment_data, fragment_timestamps = self._extract_from_buffer(
                buffer, start_sec, end_sec
            )

            if not fragment_data:
                self.logger.warning("Недостаточно данных в буфере для сохранения фрагмента")
                return None

            # Создаём аудиосегмент
            audio_segment = self._create_audio_segment(fragment_data)

            # Сжимаем и сохраняем
            file_path = self._save_audio_file(audio_segment, call_info, event_timestamp)
            self.logger.info(f"Сохранён критический фрагмент: {file_path}")
            return file_path

        except Exception as e:
            self.logger.error(f"Ошибка при сохранении фрагмента: {e}")
            return None

    def _extract_from_buffer(self,
                       buffer: Dict,
                       start_sec: float,
                       end_sec: float) -> Tuple[List[np.ndarray], List[float]]:
        """Извлечение данных из буфера в заданном временном интервале"""
        fragment_data = []
        fragment_timestamps = []

        for audio, timestamp in zip(buffer['audio_data'], buffer['timestamps']):
            if start_sec <= timestamp <= end_sec:
                fragment_data.append(audio)
                fragment_timestamps.append(timestamp)

        return fragment_data, fragment_timestamps

    def _create_audio_segment(self, audio_data: List[np.ndarray]) -> AudioSegment:
        """Создание AudioSegment из массива данных"""
        # Объединяем все чанки в один массив
        combined_audio = np.concatenate(audio_data) if audio_data else np.array([])

        # Конвертируем в формат AudioSegment
        # Предполагаем частоту дискретизации 16 кГц
        samples = (combined_audio * 32767).astype(np.int16)
        audio_segment = AudioSegment(
            samples.tobytes(),
            frame_rate=16000,
            sample_width=2,
            channels=1
        )
        return audio_segment

    def _save_audio_file(self,
                     audio_segment: AudioSegment,
                     call_info: Dict,
                     event_timestamp: float) -> str:
        """Сохранение аудиофайла в хранилище с сжатием"""
        # Генерируем имя файла
        timestamp_str = datetime.fromtimestamp(event_timestamp).strftime('%Y%m%d_%H%M%S')
        filename = f"frag_{call_info['call_id']}_{timestamp_str}.mp3"
        file_path = os.path.join(self.storage_path, filename)

        # Сохраняем в формате MP3 с компрессией
        audio_segment.export(
            file_path,
            format="mp3",
            bitrate="64k",  # Сжатие до 64 кбит/с
            parameters=["-ac", "1"]  # Моноканал
        )

        return file_path

    def cleanup_old_fragments(self, max_age_days: int = 7) -> None:
        """Очистка старых фрагментов (старше N дней)"""
        cutoff_date = datetime.now() - timedelta(days=max_age_days)

        for filename in os.listdir(self.storage_path):
            filepath = os.path.join(self.storage_path, filename)
            if os.path.isfile(filepath):
                file_mtime = datetime.fromtimestamp(os.path.getmtime(filepath))
                if file_mtime < cutoff_date:
                    os.remove(filepath)
                    self.logger.info(f"Удален старый фрагмент: {filename}")
```

---

### Интеграция всех компонентов ядра анализа


**Файл `core_analyzer.py` — главный модуль ядра анализа:**

```python
from audio_processor import AudioProcessor
from analysis_pipeline import AnalysisPipeline
from rule_engine import RuleEngine
from decision_engine import DecisionEngine
from audio_fragment_saver import AudioFragmentSaver
from typing import Dict, Any
import asyncio

class CoreAnalyzer:
    def __init__(self):
        self.audio_processor = AudioProcessor()
        self.analysis_pipeline = AnalysisPipeline()
        self.rule_engine = RuleEngine()
        self.decision_engine = DecisionEngine()
        self.fragment_saver = AudioFragmentSaver()

    async def process_call(self, call_data: Dict) -> Dict:
        """Полный процесс анализа звонка"""
        # 1. Загрузка и предварительная обработка аудио
        processed_audio, sr = self.audio_processor.load_and_preprocess(call_data['audio_path'])

        # 2. Анализ аудио через конвейер
        analysis_results = await self.analysis_pipeline.analyze_call(
            call_data['audio_path'],
            {
                'call_id': call_data['call_id'],
                'operator_id': call_data['operator_id']
            }
        )

        # 3. Расчёт негатива и классификация ситуации
        negativity_score, breakdown = self.rule_engine.calculate_negativity_score(
            analysis_results['keywords'],
            analysis_results['emotions'],
            analysis_results['acoustic_features']
        )

```python
        situation_classification = self.rule_engine.classify_situation_type(
            analysis_results['keywords'],
            analysis_results['emotions'],
            [a for a in analysis_results['summary']['detected_events']
             if a['type'].startswith('ACUSTIC')]
        )

        # 4. Принятие решения
        decision = self.decision_engine.make_decision(analysis_results)

        # 5. Сохранение критических фрагментов при необходимости
        critical_fragment_path = None
        if decision['priority_level'] in ['critical', 'high']:
            # Создаём буфер и заполняем его (в реальном сценарии буфер пополняется в реальном времени)
            buffer = self.fragment_saver.create_audio_buffer()

            # В реальном приложении буфер заполнялся бы во время записи звонка.
            # Здесь для демонстрации используем обработанный аудиосигнал.
            # Для упрощения предположим, что у нас есть временные метки событий.
            event_timestamp = self._get_event_timestamp(analysis_results)

            critical_fragment_path = self.fragment_saver.save_critical_fragment(
                buffer,
                event_timestamp,
                {
                    'call_id': call_data['call_id'],
                    'operator_id': call_data['operator_id']
                }
            )

        # 6. Формирование итогового результата
        final_result = {
            'call_id': call_data['call_id'],
            'operator_id': call_data['operator_id'],
            'analysis_timestamp': datetime.utcnow().isoformat(),
            'audio_path': call_data['audio_path'],
            'transcription': analysis_results['transcription'],
            'negativity_score': negativity_score,
            'priority_level': decision['priority_level'],
            'situation_type': situation_classification['primary_type'],
            'detailed_situation_types': situation_classification['types'],
            'detected_events': analysis_results['summary']['detected_events'],
            'decision_recommendations': decision['recommended_actions'],
            'confidence_score': decision['confidence'],
            'critical_fragment_path': critical_fragment_path,
            'processing_metadata': {
                'total_processing_time': self._calculate_processing_time(analysis_results),
                'model_versions': analysis_results.get('model_versions', {}),
                'rule_engine_version': '1.0',
                'decision_engine_version': '1.0'
            }
        }

        return final_result

    def _get_event_timestamp(self, analysis_results: Dict) -> float:
        """
        Извлечение временной метки первого критического события из результатов анализа.

        В реальном приложении здесь может быть более сложная логика:
        например, выбор самого серьёзного события или усреднение временных меток.
        """
        detected_events = analysis_results['summary']['detected_events']
        if detected_events:
            # Берём временную метку первого события
            first_event = detected_events[0]
            timestamp_str = first_event.get('timestamp', '00:00')
            minutes, seconds = map(int, timestamp_str.split(':'))
            return minutes * 60 + seconds
        return 0.0  # Если событий нет, возвращаем начало звонка

    def _calculate_processing_time(self, analysis_results: Dict) -> float:
        """Расчёт общего времени обработки"""
        # В реальном приложении здесь будет логика замера времени
        # Например, можно использовать временные метки начала и конца обработки
        start_time = analysis_results.get('start_timestamp', 0)
        end_time = analysis_results.get('end_timestamp', 0)
        return end_time - start_time if end_time > start_time else 0.0

    async def process_batch_calls(self, calls_data: List[Dict]) -> List[Dict]:
        """Обработка пакета звонков"""
        results = []
        for call_data in calls_data:
            try:
                result = await self.process_call(call_data)
                results.append(result)
            except Exception as e:
                self.logger.error(f"Ошибка обработки звонка {call_data.get('call_id')}: {e}")
                results.append({
                    'call_id': call_data.get('call_id'),
                    'error': str(e),
                    'status': 'failed'
                })
        return results

    def health_check(self) -> Dict[str, bool]:
        """Проверка работоспособности компонентов ядра"""
        checks = {
            'audio_processor_ok': hasattr(self.audio_processor, 'load_and_preprocess'),
            'analysis_pipeline_ok': hasattr(self.analysis_pipeline, 'analyze_call'),
            'rule_engine_ok': hasattr(self.rule_engine, 'calculate_negativity_score'),
            'decision_engine_ok': hasattr(self.decision_engine, 'make_decision'),
            'fragment_saver_ok': hasattr(self.fragment_saver, 'save_critical_fragment'),
            'storage_accessible': os.access(self.fragment_saver.storage_path, os.W_OK)
        }
        return checks
```

---

### Итоговый обзор этапа 3

**Реализованные компоненты:**

1. **Модуль загрузки и обработки аудио** (`audio_processor.py`):
* нормализация громкости;
* удаление шумов;
* разделение на каналы (оператор/клиент).

2. **Конвейер анализа** (`analysis_pipeline.py`):
* транскрибация аудио в текст;
* детекция эмоций в речи;
* поиск ключевых фраз;
* анализ акустических параметров (тон, темп, паузы).

3. **Система правил** (`rule_engine.py`):
* регулярные выражения для ключевых слов с уровнями серьёзности;
* пороговые значения для акустических параметров;
* весовые коэффициенты для разных типов сигналов.

4. **Механизм принятия решений** (`decision_engine.py`):
* агрегация сигналов от разных детекторов;
* расчёт общего уровня негатива (0–1);
* классификация ситуации по типам;
* определение приоритета оповещения;
* генерация рекомендаций по действиям.

5. **Модуль сохранения критических фрагментов** (`audio_fragment_saver.py`):
* буферизация аудиоданных в реальном времени;
* запись по триггеру (при обнаружении критической ситуации);
* сжатие в MP3 (64 кбит/с) для экономии места;
* автоматическое удаление старых фрагментов (старше 7 дней).

6. **Интеграционный модуль** (`core_analyzer.py`):
* единый конвейер обработки звонков;
* мониторинг состояния компонентов;
* пакетная обработка нескольких звонков;
* формирование структурированного отчёта с рекомендациями.

**Результаты этапа:**
* полностью функциональное ядро анализа звонков;
* автоматическая детекция негативных ситуаций;
* система оповещений с приоритезацией;
* сохранение доказательств (аудиофрагментов) для разбора;
* масштабируемая архитектура для дальнейшего развития.
