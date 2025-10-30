import streamlit as st
import pandas as pd
import json
import time
import os
from datetime import datetime
import plotly.express as px
import plotly.graph_objects as go
import uuid
import io

# Конфигурация страницы
st.set_page_config(
    page_title="Приоритетные задачи релиза",
    page_icon="🚀",
    layout="wide",
    initial_sidebar_state="expanded"
)

# Константы
DATA_FILE = "release_dashboard_data.json"

# CSS стили с анимациями
st.markdown("""
<style>
    @import url('https://fonts.googleapis.com/css2?family=Inter:wght@300;400;500;600;700&display=swap');

    * {
        font-family: 'Inter', sans-serif;
    }

    .main-header {
        background: linear-gradient(135deg, #000000 0%, #333333 100%);
        padding: 2rem;
        border-radius: 15px;
        margin-bottom: 2rem;
        border-left: 5px solid #FF6B35;
        animation: slideIn 0.8s ease-out;
    }

    .task-card {
        background: #1a1a1a;
        padding: 1.5rem;
        border-radius: 12px;
        margin-bottom: 1rem;
        border-left: 4px solid #FF6B35;
        transition: all 0.3s ease;
        animation: fadeIn 0.6s ease-out;
    }

    .task-card:hover {
        transform: translateY(-5px);
        box-shadow: 0 8px 25px rgba(255, 107, 53, 0.2);
        border-left: 4px solid #FF8C00;
    }

    .status-open { border-left-color: #FF6B35 !important; }
    .status-progress { border-left-color: #FFA500 !important; }
    .status-testing { border-left-color: #FFD700 !important; }
    .status-ready { border-left-color: #32CD32 !important; }
    .status-done { border-left-color: #008000 !important; }
    .status-closed { border-left-color: #696969 !important; }

    .priority-high { background: linear-gradient(135deg, #1a1a1a 0%, #331100 100%) !important; }
    .priority-medium { background: linear-gradient(135deg, #1a1a1a 0%, #332200 100%) !important; }
    .priority-low { background: linear-gradient(135deg, #1a1a1a 0%, #333300 100%) !important; }

    .danger-button {
        background: linear-gradient(135deg, #FF3333 0%, #CC0000 100%) !important;
        color: white;
        border: none;
        padding: 0.5rem 1.5rem;
        border-radius: 8px;
        font-weight: 600;
        transition: all 0.3s ease;
    }

    .danger-button:hover {
        transform: scale(1.05);
        box-shadow: 0 5px 15px rgba(255, 51, 51, 0.4);
    }

    .export-button {
        background: linear-gradient(135deg, #32CD32 0%, #228B22 100%) !important;
        color: white;
        border: none;
        padding: 0.5rem 1.5rem;
        border-radius: 8px;
        font-weight: 600;
        transition: all 0.3s ease;
    }

    .export-button:hover {
        transform: scale(1.05);
        box-shadow: 0 5px 15px rgba(50, 205, 50, 0.4);
    }

    @keyframes slideIn {
        from { transform: translateY(-30px); opacity: 0; }
        to { transform: translateY(0); opacity: 1; }
    }

    @keyframes fadeIn {
        from { opacity: 0; transform: scale(0.95); }
        to { opacity: 1; transform: scale(1); }
    }

    .stButton button {
        background: linear-gradient(135deg, #FF6B35 0%, #FF8C00 100%);
        color: white;
        border: none;
        padding: 0.5rem 1.5rem;
        border-radius: 8px;
        font-weight: 600;
        transition: all 0.3s ease;
    }

    .stButton button:hover {
        transform: scale(1.05);
        box-shadow: 0 5px 15px rgba(255, 107, 53, 0.4);
    }

    .metric-card {
        background: #1a1a1a;
        padding: 1.5rem;
        border-radius: 12px;
        text-align: center;
        border: 1px solid #333;
        animation: fadeIn 0.8s ease-out;
    }
</style>
""", unsafe_allow_html=True)


class DataManager:
    """Менеджер данных для сохранения в JSON файл"""

    @staticmethod
    def load_data():
        """Загрузка данных из файла"""
        try:
            if os.path.exists(DATA_FILE):
                with open(DATA_FILE, 'r', encoding='utf-8') as f:
                    return json.load(f)
        except Exception as e:
            st.error(f"Ошибка загрузки данных: {e}")

        # Возвращаем данные по умолчанию если файла нет или ошибка
        return {
            'releases': {
                'REL-2024.1': {
                    'name': 'REL-2024.1',
                    'tasks': [],
                    'created_date': '2024-01-15'
                }
            },
            'current_release': 'REL-2024.1',
            'task_id_counter': 1
        }

    @staticmethod
    def save_data(data):
        """Сохранение данных в файл"""
        try:
            with open(DATA_FILE, 'w', encoding='utf-8') as f:
                json.dump(data, f, ensure_ascii=False, indent=2)
            return True
        except Exception as e:
            st.error(f"Ошибка сохранения данных: {e}")
            return False


class ReleaseDashboard:
    def __init__(self):
        self.data_manager = DataManager()
        self.init_data()

    def init_data(self):
        """Инициализация данных с загрузкой из файла"""
        data = self.data_manager.load_data()

        # Инициализируем session_state из загруженных данных
        st.session_state.releases = data.get('releases', {
            'REL-2024.1': {
                'name': 'REL-2024.1',
                'tasks': [],
                'created_date': '2024-01-15'
            }
        })

        st.session_state.current_release = data.get('current_release', 'REL-2024.1')
        st.session_state.task_id_counter = data.get('task_id_counter', 1)

    def save_state(self):
        """Сохранение текущего состояния в файл"""
        data = {
            'releases': st.session_state.releases,
            'current_release': st.session_state.current_release,
            'task_id_counter': st.session_state.task_id_counter,
            'last_saved': datetime.now().isoformat()
        }
        return self.data_manager.save_data(data)

    def is_task_id_unique(self, task_id, current_task_id=None):
        """Проверка уникальности ID задачи"""
        current_release = st.session_state.current_release
        for task in st.session_state.releases[current_release]['tasks']:
            # Пропускаем текущую задачу при редактировании
            if current_task_id and task['id'] == current_task_id:
                continue
            if task['id'] == task_id:
                return False
        return True

    def generate_task_id(self):
        """Генерация автоматического ID задачи"""
        task_id = f"TASK-{st.session_state.task_id_counter:04d}"
        st.session_state.task_id_counter += 1
        self.save_state()  # Сохраняем увеличенный счетчик
        return task_id

    def add_task(self, task_id, title, description, priority, status="open"):
        """Добавление новой задачи с ручным ID"""
        # Проверяем уникальность ID
        if not self.is_task_id_unique(task_id):
            st.error(f"❌ ID задачи '{task_id}' уже существует! Используйте другой ID.")
            return False

        task = {
            'id': task_id,
            'title': title,
            'description': description,
            'priority': priority,
            'status': status,
            'created_date': datetime.now().strftime("%Y-%m-%d %H:%M"),
            'linked_tasks': []
        }

        current_release = st.session_state.current_release
        st.session_state.releases[current_release]['tasks'].append(task)

        # Сохраняем состояние после добавления
        if self.save_state():
            st.success("✅ Задача добавлена!")
        else:
            st.error("⚠️ Задача добавлена, но возникла ошибка сохранения!")

        time.sleep(1)
        st.rerun()
        return True

    def update_task_status(self, task_id, new_status):
        """Обновление статуса задачи"""
        current_release = st.session_state.current_release
        for task in st.session_state.releases[current_release]['tasks']:
            if task['id'] == task_id:
                task['status'] = new_status
                break
        self.save_state()

    def delete_task(self, task_id):
        """Удаление задачи"""
        current_release = st.session_state.current_release

        # Сначала удаляем все ссылки на эту задачу в других задачах
        for task in st.session_state.releases[current_release]['tasks']:
            task['linked_tasks'] = [link for link in task['linked_tasks'] if link['task_id'] != task_id]

        # Затем удаляем саму задачу
        st.session_state.releases[current_release]['tasks'] = [
            task for task in st.session_state.releases[current_release]['tasks']
            if task['id'] != task_id
        ]

        if self.save_state():
            st.success("🗑️ Задача удалена!")
        else:
            st.error("⚠️ Задача удалена, но возникла ошибка сохранения!")

        time.sleep(1)
        st.rerun()

    def delete_release(self, release_name):
        """Удаление релиза"""
        if len(st.session_state.releases) <= 1:
            st.error("❌ Нельзя удалить последний релиз!")
            return

        if release_name in st.session_state.releases:
            del st.session_state.releases[release_name]

            # Если удаляем текущий релиз, переключаемся на первый доступный
            if st.session_state.current_release == release_name:
                st.session_state.current_release = list(st.session_state.releases.keys())[0]

            if self.save_state():
                st.success(f"✅ Релиз {release_name} удален!")
            else:
                st.error(f"⚠️ Релиз {release_name} удален, но возникла ошибка сохранения!")

            time.sleep(1)
            st.rerun()

    def link_tasks(self, task_id, linked_task_id, link_type):
        """Связывание задач"""
        current_release = st.session_state.current_release

        # Проверяем существование обеих задач
        task_ids = [task['id'] for task in st.session_state.releases[current_release]['tasks']]
        if task_id not in task_ids:
            st.error(f"❌ Задача {task_id} не найдена!")
            return
        if linked_task_id not in task_ids:
            st.error(f"❌ Задача {linked_task_id} не найдена!")
            return

        # Добавляем связь
        for task in st.session_state.releases[current_release]['tasks']:
            if task['id'] == task_id:
                # Проверяем, нет ли уже такой связи
                existing_link = next((link for link in task['linked_tasks'] if link['task_id'] == linked_task_id), None)
                if not existing_link:
                    task['linked_tasks'].append({
                        'task_id': linked_task_id,
                        'type': link_type,
                        'link_id': str(uuid.uuid4())[:8]  # Уникальный ID связи
                    })
                    self.save_state()
                    st.success("✅ Задачи связаны!")
                else:
                    st.info("ℹ️ Связь между этими задачами уже существует")
                break

    def remove_task_link(self, task_id, link_id):
        """Удаление связи между задачами"""
        current_release = st.session_state.current_release
        for task in st.session_state.releases[current_release]['tasks']:
            if task['id'] == task_id:
                task['linked_tasks'] = [link for link in task['linked_tasks'] if link['link_id'] != link_id]
                self.save_state()
                st.success("🔗 Связь удалена!")
                time.sleep(1)
                st.rerun()
                break

    def prepare_export_data(self):
        """Подготовка данных для экспорта"""
        current_release = st.session_state.current_release
        tasks = st.session_state.releases[current_release]['tasks']

        # Создаем данные для экспорта
        export_data = []

        for task in tasks:
            # Форматируем связи в читаемый вид
            links_info = ""
            if task.get('linked_tasks'):
                links_list = []
                for link in task['linked_tasks']:
                    # Находим название связанной задачи
                    linked_task = next((t for t in tasks if t['id'] == link['task_id']), None)
                    task_title = linked_task['title'] if linked_task else link['task_id']
                    links_list.append(f"{link['type']} -> {link['task_id']}: {task_title}")
                links_info = "; ".join(links_list)

            # Форматируем статус и приоритет для читаемости
            status_display = {
                'open': 'Открыта',
                'in progress': 'В работе',
                'ready for testing': 'Готова к тестированию',
                'testing': 'Тестируется',
                'done': 'Выполнена',
                'closed': 'Закрыта'
            }.get(task['status'], task['status'])

            priority_display = {
                'low': 'Низкий',
                'medium': 'Средний',
                'high': 'Высокий'
            }.get(task['priority'], task['priority'])

            export_data.append({
                'ID задачи': task['id'],
                'Ссылка на задачу': task['title'],
                'Ссылка на деплой': task['description'],
                'Статус': status_display,
                'Приоритет': priority_display,
                'Дата создания': task['created_date'],
                'Связи': links_info
            })

        return pd.DataFrame(export_data)

    def export_tasks_to_csv(self):
        """Экспорт задач в CSV с правильной кодировкой для Excel"""
        df = self.prepare_export_data()

        # Создаем CSV с кодировкой windows-1251 для корректного отображения в Excel
        output = io.BytesIO()

        # Пишем BOM для UTF-8 (это поможет Excel правильно определить кодировку)
        output.write(b'\xef\xbb\xbf')

        # Конвертируем DataFrame в CSV с UTF-8 кодировкой
        csv_content = df.to_csv(index=False, encoding='utf-8')
        output.write(csv_content.encode('utf-8'))

        return output.getvalue()

    def get_status_color(self, status):
        """Получение цвета для статуса"""
        colors = {
            'open': '#FF6B35',
            'in progress': '#FFA500',
            'ready for testing': '#FFD700',
            'testing': '#32CD32',
            'done': '#008000',
            'closed': '#696969'
        }
        return colors.get(status, '#FF6B35')

    def create_metrics(self):
        """Создание метрик"""
        current_release = st.session_state.current_release
        tasks = st.session_state.releases[current_release]['tasks']

        if not tasks:
            return

        status_counts = {}
        priority_counts = {}

        for task in tasks:
            status_counts[task['status']] = status_counts.get(task['status'], 0) + 1
            priority_counts[task['priority']] = priority_counts.get(task['priority'], 0) + 1

        # Метрики
        col1, col2, col3, col4 = st.columns(4)

        with col1:
            total_tasks = len(tasks)
            st.markdown(f"""
            <div class="metric-card">
                <h3 style="color: #FF6B35; margin: 0;">📊 Всего задач</h3>
                <h1 style="color: white; margin: 0.5rem 0;">{total_tasks}</h1>
            </div>
            """, unsafe_allow_html=True)

        with col2:
            completed = status_counts.get('done', 0) + status_counts.get('closed', 0)
            progress = (completed / total_tasks * 100) if total_tasks > 0 else 0
            st.markdown(f"""
            <div class="metric-card">
                <h3 style="color: #FF6B35; margin: 0;">✅ Выполнено</h3>
                <h1 style="color: white; margin: 0.5rem 0;">{completed}</h1>
                <p style="color: #FF8C00; margin: 0;">{progress:.1f}%</p>
            </div>
            """, unsafe_allow_html=True)

        with col3:
            in_progress = status_counts.get('in progress', 0) + status_counts.get('testing', 0)
            st.markdown(f"""
            <div class="metric-card">
                <h3 style="color: #FF6B35; margin: 0;">🔄 В работе</h3>
                <h1 style="color: white; margin: 0.5rem 0;">{in_progress}</h1>
            </div>
            """, unsafe_allow_html=True)

        with col4:
            high_priority = priority_counts.get('high', 0)
            st.markdown(f"""
            <div class="metric-card">
                <h3 style="color: #FF6B35; margin: 0;">🚨 Высокий приоритет</h3>
                <h1 style="color: white; margin: 0.5rem 0;">{high_priority}</h1>
            </div>
            """, unsafe_allow_html=True)

    def create_status_chart(self):
        """Создание графика статусов"""
        current_release = st.session_state.current_release
        tasks = st.session_state.releases[current_release]['tasks']

        if not tasks:
            return

        status_counts = {}
        for task in tasks:
            status_counts[task['status']] = status_counts.get(task['status'], 0) + 1

        # Создание графика
        fig = px.pie(
            values=list(status_counts.values()),
            names=list(status_counts.keys()),
            title="Распределение задач по статусам",
            color_discrete_sequence=['#FF6B35', '#FFA500', '#FFD700', '#32CD32', '#008000', '#696969']
        )

        fig.update_traces(textposition='inside', textinfo='percent+label')
        fig.update_layout(
            paper_bgcolor='rgba(0,0,0,0)',
            plot_bgcolor='rgba(0,0,0,0)',
            font_color='white',
            title_font_color='#FF6B35'
        )

        st.plotly_chart(fig, use_container_width=True)

    def render(self):
        """Основной метод рендеринга"""

        # Индикатор состояния данных
        if st.sidebar.button("🔄 Обновить данные"):
            self.init_data()
            st.rerun()

        # Заголовок
        st.markdown("""
        <div class="main-header">
            <h1 style="color: #FF6B35; margin: 0; font-size: 2.5rem;">🚀 Приоритетные задачи релиза</h1>
        </div>
        """, unsafe_allow_html=True)

        # Боковая панель - управление релизом
        with st.sidebar:
            st.markdown("### 🎯 Управление релизом")

            # Выбор релиза
            release_options = list(st.session_state.releases.keys())
            current_release = st.selectbox(
                "Текущий релиз:",
                release_options,
                key="release_selector"
            )

            if current_release != st.session_state.current_release:
                st.session_state.current_release = current_release
                self.save_state()
                st.rerun()

            # Удаление текущего релиза
            st.markdown("---")
            st.markdown("### 🗑️ Управление релизами")
            if st.button("❌ Удалить текущий релиз", use_container_width=True, type="secondary"):
                self.delete_release(st.session_state.current_release)

            # Экспорт
            st.markdown("---")
            st.markdown("### 📤 Экспорт задач")

            if st.button("📊 Выгрузить в CSV", use_container_width=True, key="export_csv"):
                csv_data = self.export_tasks_to_csv()
                timestamp = datetime.now().strftime('%Y%m%d_%H%M')

                st.download_button(
                    label="💾 Скачать CSV файл",
                    data=csv_data,
                    file_name=f"tasks_{st.session_state.current_release}_{timestamp}.csv",
                    mime="text/csv",
                    use_container_width=True,
                    key="download_csv"
                )

            # Создание нового релиза
            st.markdown("---")
            st.markdown("### 📦 Новый релиз")
            new_release_name = st.text_input("Название релиза:", placeholder="REL-2024.2")
            if st.button("➕ Создать релиз", use_container_width=True):
                if new_release_name and new_release_name not in st.session_state.releases:
                    st.session_state.releases[new_release_name] = {
                        'name': new_release_name,
                        'tasks': [],
                        'created_date': datetime.now().strftime("%Y-%m-%d")
                    }
                    st.session_state.current_release = new_release_name
                    if self.save_state():
                        st.success(f"✅ Релиз {new_release_name} создан!")
                    time.sleep(1)
                    st.rerun()
                elif new_release_name in st.session_state.releases:
                    st.error("❌ Релиз с таким именем уже существует!")

            # Статистика
            st.markdown("---")
            st.markdown("### 📈 Статистика")
            current_tasks = st.session_state.releases[st.session_state.current_release]['tasks']
            if current_tasks:
                completed = len([t for t in current_tasks if t['status'] in ['done', 'closed']])
                progress = (completed / len(current_tasks)) * 100
                st.metric("Прогресс", f"{progress:.1f}%")
                st.metric("Всего задач", len(current_tasks))

                # Количество связей
                total_links = sum(len(task.get('linked_tasks', [])) for task in current_tasks)
                st.metric("🔗 Связей между задачами", total_links)
            else:
                st.metric("Всего задач", 0)
                st.metric("Прогресс", "0%")
                st.metric("🔗 Связей между задачами", 0)

        # Основная область - добавление задачи
        col1, col2 = st.columns([2, 1])

        with col1:
            st.markdown("### 📝 Добавить новую задачу")
            with st.form("add_task_form"):
                col_id, col_title = st.columns([1, 2])

                with col_id:
                    manual_id = st.text_input(
                        "ID задачи:",
                        placeholder="TASK-001",
                        help="Введите уникальный ID задачи или оставьте пустым для автогенерации"
                    )

                with col_title:
                    task_title = st.text_input("Ссылка на задачу:", placeholder="Вставьте ссылку...")

                task_description = st.text_area("Ссылка на деплой:", placeholder="Вставьте ссылку...")

                col11, col12 = st.columns(2)
                with col11:
                    task_priority = st.selectbox(
                        "Приоритет:",
                        ["low", "medium", "high"],
                        format_func=lambda x: {"low": "🔵 Низкий", "medium": "🟡 Средний", "high": "🔴 Высокий"}[x]
                    )
                with col12:
                    task_status = st.selectbox(
                        "Статус:",
                        ["open", "in progress", "ready for testing", "testing", "done", "closed"]
                    )

                if st.form_submit_button("➕ Добавить задачу", use_container_width=True):
                    if task_title:
                        # Генерируем ID если не введен вручную
                        if not manual_id:
                            manual_id = self.generate_task_id()

                        self.add_task(manual_id, task_title, task_description, task_priority, task_status)
                    else:
                        st.error("❌ Введите название задачи!")

        with col2:
            st.markdown("### 🔗 Связать задачи")
            current_tasks = st.session_state.releases[st.session_state.current_release]['tasks']
            if len(current_tasks) >= 2:
                task_options = {task['id']: f"{task['id']}: {task['title']}" for task in current_tasks}

                task1 = st.selectbox("Задача 1:", list(task_options.keys()), format_func=lambda x: task_options[x])
                task2 = st.selectbox("Задача 2:", list(task_options.keys()), format_func=lambda x: task_options[x])
                link_type = st.selectbox("Тип связи:", ["blocks", "related to", "duplicate", "depends on"])

                if st.button("🔗 Связать задачи", use_container_width=True):
                    if task1 != task2:
                        self.link_tasks(task1, task2, link_type)
                    else:
                        st.error("❌ Выберите разные задачи!")
            else:
                st.info("ℹ️ Нужно как минимум 2 задачи для создания связей")

        # Метрики
        self.create_metrics()

        st.markdown("---")

        # Предпросмотр данных для экспорта
        st.markdown("### 👁️ Предпросмотр данных для экспорта")
        current_tasks = st.session_state.releases[st.session_state.current_release]['tasks']
        if current_tasks:
            df_preview = self.prepare_export_data()
            st.dataframe(df_preview, use_container_width=True)
        else:
            st.info("Нет задач для отображения")

        st.markdown("---")

        # Список задач
        st.markdown("### 📋 Задачи релиза")

        current_release = st.session_state.current_release
        tasks = st.session_state.releases[current_release]['tasks']

        if not tasks:
            st.info("🎉 Пока нет задач в этом релизе! Добавьте первую задачу выше.")
            return

        # Фильтры - ПО УМОЛЧАНИЮ ВСЕ ЗАДАЧИ
        col1, col2, col3 = st.columns(3)
        with col1:
            filter_status = st.multiselect(
                "Фильтр по статусу:",
                ["open", "in progress", "ready for testing", "testing", "done", "closed"],
                default=["open", "in progress", "ready for testing", "testing", "done", "closed"]
            )
        with col2:
            filter_priority = st.multiselect(
                "Фильтр по приоритету:",
                ["low", "medium", "high"],
                default=["low", "medium", "high"]
            )

        # Применение фильтров
        filtered_tasks = [
            task for task in tasks
            if task['status'] in filter_status and task['priority'] in filter_priority
        ]

        if not filtered_tasks:
            st.info("🔍 Нет задач, соответствующих выбранным фильтрам")
            return

        # Отображение задач
        for task in filtered_tasks:
            status_class = f"status-{task['status'].replace(' ', '-')}"
            priority_class = f"priority-{task['priority']}"

            col1, col2 = st.columns([3, 1])

            with col1:
                # Отображаем связанные задачи
                linked_tasks_html = ""
                if task.get('linked_tasks'):
                    linked_tasks_html = "<div style='margin-top: 0.5rem;'><strong style='color: #FF8C00;'>🔗 Связанные задачи:</strong><br>"
                    for link in task['linked_tasks']:
                        # Находим связанную задачу для отображения названия
                        linked_task = next((t for t in tasks if t['id'] == link['task_id']), None)
                        task_title = linked_task['title'] if linked_task else link['task_id']
                        linked_tasks_html += f"""
                        <div style='display: flex; justify-content: space-between; align-items: center; background: #2a2a2a; padding: 0.3rem 0.5rem; margin: 0.2rem 0; border-radius: 4px;'>
                            <span>{link['type']} → {link['task_id']}: {task_title}</span>
                        </div>
                        """
                    linked_tasks_html += "</div>"

                st.markdown(f"""
                <div class="task-card {status_class} {priority_class}">
                    <div style="display: flex; justify-content: between; align-items: start;">
                        <div style="flex: 1;">
                            <h4 style="color: white; margin: 0 0 0.5rem 0;">{task['title']}</h4>
                            <p style="color: #CCCCCC; margin: 0 0 0.5rem 0;">{task['description']}</p>
                            <div style="display: flex; gap: 1rem; font-size: 0.9rem;">
                                <span style="color: #FF6B35;">🆔 {task['id']}</span>
                                <span style="color: #FFA500;">📅 {task['created_date']}</span>
                                <span style="color: {self.get_status_color(task['status'])};">● {task['status'].upper()}</span>
                                <span style="color: #32CD32;">🎯 {task['priority'].upper()}</span>
                            </div>
                            {linked_tasks_html}
                        </div>
                    </div>
                </div>
                """, unsafe_allow_html=True)

            with col2:
                # Управление статусом
                new_status = st.selectbox(
                    "Изменить статус:",
                    ["open", "in progress", "ready for testing", "testing", "done", "closed"],
                    index=["open", "in progress", "ready for testing", "testing", "done", "closed"].index(
                        task['status']),
                    key=f"status_{task['id']}"
                )

                if new_status != task['status']:
                    self.update_task_status(task['id'], new_status)
                    st.rerun()

                # Кнопка удаления
                if st.button("🗑️ Удалить", key=f"delete_{task['id']}", use_container_width=True):
                    self.delete_task(task['id'])

        # Графики
        st.markdown("---")
        col1, col2 = st.columns(2)

        with col1:
            self.create_status_chart()

        with col2:
            st.markdown("### 📊 Приоритеты задач")
            current_tasks = st.session_state.releases[st.session_state.current_release]['tasks']
            if current_tasks:
                priority_data = {
                    'low': len([t for t in current_tasks if t['priority'] == 'low']),
                    'medium': len([t for t in current_tasks if t['priority'] == 'medium']),
                    'high': len([t for t in current_tasks if t['priority'] == 'high'])
                }

                fig = go.Figure(data=[go.Bar(
                    x=list(priority_data.keys()),
                    y=list(priority_data.values()),
                    marker_color=['#32CD32', '#FFA500', '#FF6B35']
                )])

                fig.update_layout(
                    title="Распределение по приоритетам",
                    paper_bgcolor='rgba(0,0,0,0)',
                    plot_bgcolor='rgba(0,0,0,0)',
                    font_color='white',
                    title_font_color='#FF6B35'
                )

                st.plotly_chart(fig, use_container_width=True)


# Запуск приложения
if __name__ == "__main__":
    dashboard = ReleaseDashboard()
    dashboard.render()
