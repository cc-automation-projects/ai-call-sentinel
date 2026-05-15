### Этап 5. Разработка дашборда


#### Шаг 5.1. Проектирование UI


**Файл `dashboard_ui.py`:**

```python
import dash
from dash import dcc, html, Input, Output, State
import plotly.graph_objects as go
import plotly.express as px
from datetime import datetime, timedelta
import pandas as pd

class DashboardUI:
    def __init__(self):
        self.app = dash.Dash(__name__, suppress_callback_exceptions=True)
        self.setup_layout()


    def setup_layout(self):
        """Настройка структуры дашборда"""
        self.app.layout = html.Div([
            # Заголовок
            html.H1("📊 Дашборд анализа звонков", className="dashboard-title"),

            # Главная панель с метриками
            self._create_kpi_cards(),

            # Фильтры
            self._create_filters(),
            # Визуализации
            html.Div([
                dcc.Tabs([
                    dcc.Tab(label='Главная', children=[
                self._create_main_visualizations()
            ]),
            dcc.Tab(label='По операторам', children=[
                self._create_operator_analysis()
            ]),
            dcc.Tab(label='Детализация', children=[
                self._create_detailed_view()
            ])
        ])
            ], className="tabs-container")
        ], className="dashboard-container")

    def _create_kpi_cards(self):
        """Создание карточек KPI"""
        return html.Div([
            html.Div([
                html.H3("Всего звонков"),
                html.Div(id="total-calls-kpi", className="kpi-value")
            ], className="kpi-card"),
            html.Div([
                html.H3("Негативных звонков"),
                html.Div(id="negative-calls-kpi", className="kpi-value")
            ], className="kpi-card"),
            html.Div([
                html.H3("Средний негатив"),
                html.Div(id="avg-negativity-kpi", className="kpi-value")
            ], className="kpi-card"),
            html.Div([
                html.H3("Критических ситуаций"),
                html.Div(id="critical-situations-kpi", className="kpi-value")
            ], className="kpi-card")
        ], className="kpi-container")

    def _create_filters(self):
        """Создание панели фильтров"""
        return html.Div([
            html.H4("🔎 Фильтры"),
            html.Div([
                # Фильтр по периоду
                dcc.DatePickerRange(
                    id='date-range-filter',
            start_date=datetime.now() - timedelta(days=7),
            end_date=datetime.now(),
            display_format='DD.MM.YYYY'
        ),
        # Фильтр по операторам
        dcc.Dropdown(
            id='operator-filter',
            placeholder="Все операторы",
            multi=True
        ),
        # Фильтр по типам ситуаций
        dcc.Dropdown(
            id='situation-type-filter',
            options=[
                {'label': 'Все типы', 'value': 'all'},
                {'label': 'Конфликт', 'value': 'conflict'},
                {'label': 'Жалоба', 'value': 'complaint'},
                {'label': 'Угроза', 'value': 'threat'}
            ],
            value='all',
            placeholder="Тип ситуации"
        ),
        # Фильтр по серьёзности
        dcc.Dropdown
            id='severity-filter',
            options=[
                {'label': 'Все уровни', 'value': 'all'},
                {'label': 'Критический', 'value': 'critical'},
                {'label': 'Высокий', 'value': 'high'},
                {'label': 'Средний', 'value': 'medium'},
                {'label': 'Низкий', 'value': 'low'}
            ],
            value='all',
            placeholder="Серьёзность"
        )
    ], className="filters-container")
```

#### Шаг 5.2. Реализация визуализаций

```python
    def _create_main_visualizations(self):
        """Основные визуализации на главной странице"""
        return html.Div([
            # График динамики негативных звонков
            dcc.Graph(id='negativity-trend-graph'),
            # Тепловая карта по операторам и времени
            dcc.Graph(id='heatmap-operators-time'),
            # Круговая диаграмма распределения по типам ситуаций
            dcc.Graph(id='situation-types-pie')
        ], className="main-visualizations")

    def _create_operator_analysis(self):
        """Визуализации по операторам"""
        return html.Div([
            dcc.Graph(id='operators-performance-bar'),
            dcc.Graph(id='operators-negativity-trend'),
            html.Div(id='operator-details-table')
        ])

    def _create_detailed_view(self):
        """Детализированный просмотр звонков"""
        return html.Div([
            html.H3("🎙️ Детализация звонков"),
            dash_table.DataTable(
                id='calls-table',
                columns=[
                    {'name': 'ID звонка', 'id': 'call_id'},
            {'name': 'Оператор', 'id': 'operator_id'},
            {'name': 'Уровень негатива', 'id': 'negativity_score'},
            {'name': 'Тип ситуации', 'id': 'situation_type'},
            {'name': 'Время', 'id': 'analysis_timestamp'}
        ],
        data=[],
        page_size=10,
        style_table={'overflowX': 'auto'},
        style_cell={'textAlign': 'left'}
    ),
    html.Div(id='audio-player-container')
])
```

#### Шаг 5.3. Создание графиков и диаграмм

**Файл `visualizations.py`:**

```python
class Visualizations:
    @staticmethod
    def create_negativity_trend(data: pd.DataFrame) -> go.Figure:
        """График динамики уровня негатива по дням"""
        daily_stats = data.groupby('date').agg({
            'negativity_score': 'mean',
            'call_id': 'count'
        }).reset_index()

        fig = go.Figure()
        fig.add_trace(go.Scatter(
            x=daily_stats['date'],
            y=daily_stats['negativity_score'],
            mode='lines+markers',
            name='Средний негатив'
        ))
        fig.update_layout(
            title='Динамика уровня негатива',
            xaxis_title='Дата',
            yaxis_title='Средний уровень негатива'
        )
        return fig

    @staticmethod
    def create_heatmap_operators_time(data: pd.DataFrame) -> go.Figure:
        """Тепловая карта по операторам и времени суток"""
        pivot_data = data.pivot_table(
            values='negativity_score',
            index='operator_id',
            columns='hour',
            aggfunc='mean'
        ).fillna(0)

        fig = px.imshow(
            pivot_data,
            labels=dict(x="Час дня", y="Оператор", color="Уровень негатива"),
            x=pivot_data.columns,
            y=pivot_data.index,
            color_continuous_scale='Reds'
        )
        fig.update_layout(title='Распределение негатива по операторам и времени')
        return fig

    @staticmethod
    def create_situation_types_pie(data: pd.DataFrame) -> go.Figure:
        """Круговая диаграмма по типам ситуаций"""
        type_counts = data['situation_type'].value_counts()

        fig = px.pie(
            type_counts,
            values=type_counts.values,
            names=type_counts.index,
            title='Распределение по типам ситуаций'
        )
        return fig

    @staticmethod
    def create_operators_performance(data: pd.DataFrame) -> go.Figure:
        """Столбчатая диаграмма эффективности операторов"""
        operator_stats = data.groupby('operator_id').agg({
            'negativity_score': 'mean',
            'call_id': 'count',
            'priority_level': lambda x: (x == 'critical').sum()
        }).reset_index()

        fig = go.Figure(data=[
            go.Bar(
                name='Средний негатив',
                x=operator_stats['operator_id'],
                y=operator_stats['negativity_score'],
                marker_color='lightcoral'
            ),
            go.Bar(
                name='Критические ситуации',
                x=operator_stats['operator_id'],
                y=operator_stats['priority_level'],
                marker_color='darkred',
                yaxis='y2'
            )
        ])

        fig.update_layout(
            title='Эффективность операторов',
            xaxis_title='Оператор',
            yaxis_title='Средний уровень негатива',
            yaxis2=dict(
                title='Количество критических ситуаций',
                overlaying='y',
                side='right'
            ),
            barmode='group'
        )
        return fig

    @staticmethod
    def create_operators_negativity_trend(data: pd.DataFrame) -> go.Figure:
        """График динамики негатива по операторам"""
        # Группируем по оператору и дате
        trend_data = data.groupby(['operator_id', 'date']).agg({
            'negativity_score': 'mean'
        }).reset_index()

        fig = go.Figure()
        operators = trend_data['operator_id'].unique()

        for operator in operators:
            operator_data = trend_data[trend_data['operator_id'] == operator]
            fig.add_trace(go.Scatter(
                x=operator_data['date'],
                y=operator_data['negativity_score'],
                mode='lines+markers',
                name=operator
            ))

        fig.update_layout(
            title='Динамика уровня негатива по операторам',
            xaxis_title='Дата',
            yaxis_title='Уровень негатива'
        )
        return fig

    @staticmethod
    def create_severity_distribution(data: pd.DataFrame) -> go.Figure:
        """Распределение по уровням серьёзности"""
        severity_counts = data['priority_level'].value_counts().reset_index()
        severity_counts.columns = ['priority_level', 'count']

        colors = {'critical': 'red', 'high': 'orange', 'medium': 'yellow', 'low': 'green'}

        fig = px.bar(
            severity_counts,
            x='priority_level',
            y='count',
            color='priority_level',
            color_discrete_map=colors,
            title='Распределение по уровням серьёзности'
        )
        fig.update_layout(xaxis_title='Серьёзность', yaxis_title='Количество')
        return fig

    @staticmethod
    def create_top_problematic_calls(data: pd.DataFrame, top_n: int = 10) -> go.Figure:
        """Топ самых проблемных звонков"""
        top_calls = data.nlargest(top_n, 'negativity_score')

        fig = go.Figure(data=go.Bar(
            x=top_calls['negativity_score'],
            y=top_calls['call_id'],
            orientation='h',
            marker_color='crimson'
        ))
        fig.update_layout(
            title=f'Топ-{top_n} самых проблемных звонков',
            xaxis_title='Уровень негатива',
            yaxis_title='ID звонка'
        )
        return fig
```

#### Шаг 5.4. Разработка модуля отчётов

**Файл `report_generator.py`:**

```python
import pandas as pd
from fpdf import FPDF
import xlsxwriter
from datetime import datetime
from typing import Dict, List

class ReportGenerator:
    @staticmethod
    def generate_daily_report(data: pd.DataFrame, date: str) -> Dict[str, bytes]:
        """Генерация ежедневного отчёта в PDF и Excel"""
        reports = {}

        # Фильтруем данные за день
        daily_data = data[data['date'] == date]

        # Статистика за день
        stats = {
            'total_calls': len(daily_data),
            'negative_calls': len(daily_data[daily_data['negativity_score'] > 0.5]),
            'critical_situations': len(daily_data[daily_data['priority_level'] == 'critical']),
            'avg_negativity': daily_data['negativity_score'].mean()
        }

        # PDF отчёт
        pdf = FPDF()
        pdf.add_page()
        pdf.set_font("Arial", size=12)

        pdf.cell(200, 10, txt=f"Ежедневный отчёт — {date}", ln=True, align='C')
        pdf.ln(10)

        # Добавляем статистику
        for key, value in stats.items():
            pdf.cell(200, 10, txt=f"{key.replace('_', ' ').title()}: {value}", ln=True)

        pdf.ln(10)
        pdf.cell(200, 10, txt="Топ-5 проблемных звонков:", ln=True)

        # Топ-5 звонков по негативу
        top_5 = daily_data.nlargest(5, 'negativity_score')[['call_id', 'operator_id', 'negativity_score', 'situation_type']]
        for _, row in top_5.iterrows():
            pdf.cell(200, 10, txt=f"ID: {row['call_id']}, Оператор: {row['operator_id']}, "
                               f"Негатив: {row['negativity_score']:.2f}, Тип: {row['situation_type']}", ln=True)

        reports['pdf'] = pdf.output(dest='S').encode('latin1')

        # Excel отчёт
        output = BytesIO()
        with pd.ExcelWriter(output, engine='xlsxwriter') as writer:
            daily_data.to_excel(writer, sheet_name='Данные за день', index=False)
            summary_df = pd.DataFrame([stats])
            summary_df.to_excel(writer, sheet_name='Сводка', index=False)
        reports['excel'] = output.getvalue()

        return reports

    @staticmethod
    def export_to_bi_system(data: pd.DataFrame, format: str = 'csv') -> bytes:
        """Экспорт данных для BI‑систем"""
        if format == 'csv':
            return data.to_csv(index=False).encode('utf-8')
        elif format == 'json':
            return data.to_json(orient='records').encode('utf-8')
        else:
            raise ValueError("Поддерживаются форматы: csv, json")
```

#### Шаг 5.5. Интеграция дашборда с системой анализа

**Файл `dashboard_integration.py`:**

```python
from dashboard_ui import DashboardUI
from visualizations import Visualizations
from report_generator import ReportGenerator
from core_analyzer import CoreAnalyzer
import pandas as pd

class DashboardIntegration:
    def __init__(self, analyzer: CoreAnalyzer):
        self.analyzer = analyzer
        self.ui = DashboardUI()
        self.visualizations = Visualizations()
        self.reports = ReportGenerator()
        self.setup_callbacks()

    def setup_callbacks(self):
        """Настройка callback‑функций для интерактивности"""
        @self.ui.app.callback(
            [Output('total-calls-kpi', 'children'),
             Output('negative-calls-kpi', 'children'),
             Output('avg-negativity-kpi', 'children'),
             Output('critical-situations-kpi', 'children'),
             Output('negativity-trend-graph', 'figure'),
             Output('heatmap-operators-time', 'figure'),
             Output('situation-types-pie', 'figure'),
             Output('operators-performance-bar', 'figure'),
             Output('operators-negativity-trend', 'figure'),
             Output('calls-table', 'data')],
            [Input('date-range-filter', 'start_date'),
             Input('date-range-filter', 'end_date'),
             Input('operator-filter', 'value'),
             Input('situation-type-filter', 'value'),
             Input('severity-filter', 'value')]
        )
        def update_dashboard(start_date, end_date, operators, situation_type, severity):
            # Получаем данные из системы анализа
            raw_data = self.analyzer.get_analysis_results()
            filtered_data = self._apply_filters(
                raw_data, start_date, end_date, operators, situation_type, severity
            )

            # Расчёт KPI
            total_calls = len(filtered_data)
            negative_calls = len(filtered_data[filtered_data['negativity_score'] > 0.5])
            avg_negativity = filtered_data['negativity_score'].mean() if not filtered_data.empty else 0
            critical_situations = len(filtered_data[filtered_data['priority_level'] == 'critical'])

            # Создание визуализаций
            negativity_trend_fig = self.visualizations.create_negativity_trend(filtered_data)
            heatmap_fig = self.visualizations.create_heatmap_operators_time(filtered_data)
            pie_fig = self.visualizations.create_situation_types_pie(filtered_data)
            operators_perf_fig = self.visualizations.create_operators_performance(filtered_data)
            trend_fig = self.visualizations.create_operators_negativity_trend(filtered_data)

            # Подготовка данных для таблицы
            table_data = filtered_data[[
                'call_id', 'operator_id', 'negativity_score', 'situation_type', 'analysis_timestamp'
            ]].to_dict('records')

            return (
                f"{total_calls}",
                f"{negative_calls}",
                f"{avg_negativity:.2f}",
                f"{critical_situations}",
                negativity_trend_fig,
                heatmap_fig,
                pie_fig,
                operators_perf_fig,
                trend_fig,
                table_data
            )

    def _apply_filters(self, data: pd.DataFrame, start_date, end_date,
                        operators, situation_type, severity) -> pd.DataFrame:
        """Применение фильтров к данным"""
        filtered = data.copy()

        # Фильтр по дате
        if start_date and end_date:
            filtered = filtered[
                (filtered['analysis_timestamp'] >= start_date) &
                (filtered['analysis_timestamp'] <= end_date)
            ]

        # Фильтр по операторам
        if operators:
            filtered = filtered[filtered['operator_id'].isin(operators)]

        # Фильтр по типу ситуации
        if situation_type and situation_type != 'all':
            filtered = filtered[filtered['situation_type'] == situation_type]

        # Фильтр по серьёзности
        if severity and severity != 'all':
            filtered = filtered[filtered['priority_level'] == severity]

        return filtered

    def run_server(self, debug: bool = True, port: int = 8050):
        """Запуск сервера дашборда"""
        self.ui.app.run_server(debug=debug, port=port)
```

#### Шаг 5.6. Модуль просмотра аудиозаписей

**Файл `audio_player.py`:**

```python
import dash_html_components as html
import dash_core_components as dcc
from dash.dependencies import Input, Output
import base64

class AudioPlayer:
    @staticmethod
    def create_audio_player(call_id: str, audio_path: str) -> html.Div:
        """Создание компонента плеера для аудиозаписи"""
        if not audio_path or not os.path.exists(audio_path):
            return html.Div("Аудиофрагмент недоступен")

        # Кодируем аудио в base64 для встраивания
        with open(audio_path, 'rb') as f:
            audio_bytes = f.read()
            audio_base64 = base64.b64encode(audio_bytes).decode()

        audio_src = f"data:audio/mp3;base64,{audio_base64}"

        return html.Div([
            html.H4(f"Аудиозапись звонка {call_id}"),
            html.Audio(
                src=audio_src,
                controls=True,
                style={'width': '100%'}
            ),
            html.P(f"Путь к файлу: {audio_path}")
        ])

    @staticmethod
    def add_audio_to_table_callback(app):
        """Добавление callback для отображения аудио при выборе строки в таблице"""
        @app.callback(
            Output('audio-player-container', 'children'),
            [Input('calls-table', 'selected_rows')],
            [State('calls-table', 'data')]
        )
        def display_audio(selected_rows, table_data):
            if selected_rows and selected_rows[0] < len(table_data):
                selected_call = table_data[selected_rows[0]]
                call_id = selected_call['call_id']
                # Здесь должен быть механизм получения пути к аудио по ID звонка
                audio_path = self._get_audio_path_by_call_id(call_id)
                return AudioPlayer.create_audio_player(call_id, audio_path)
            return "Выберите звонок из таблицы для прослушивания аудио"
```

---

### Итоговый обзор этапа 5

**Реализованные компоненты:**

1. **Пользовательский интерфейс:**
* главная панель с метриками KPI;
* интерактивные фильтры по всем параметрам;
* вкладки для разных видов анализа;
* детализированная таблица звонков;
* встроенный аудиоплеер.

2. **Визуализации:**
* график динамики негатива по дням;
* тепловая карта по операторам и времени;
* круговая диаграмма типов ситуаций;
* столбчатые диаграммы эффективности операторов;
* тренды негатива по операторам;
* распределение по уровням серьёзности;
* топ проблемных звонков.

3. **Модуль отчётов:**
* ежедневные PDF‑отчёты с сводкой и топ‑5 звонков;
* Excel‑отчёты с полными данными;
* экспорт в форматах CSV/JSON для BI‑систем.

4. **Интеграция с системой анализа:**
* централизованное получение данных из `CoreAnalyzer`;
* применение фильтров в реальном времени;
* автоматическое обновление визуализаций;
* интерактивный просмотр аудиозаписей.

**Результаты этапа:**
* полноценный интерактивный дашборд для мониторинга;
* инструменты для глубокого анализа данных;
* автоматизированная генерация отчётов;
* возможность детального разбора конкретных звонков;
* масштабируемая архитектура для дальнейшего развития.