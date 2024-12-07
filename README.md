### Варианты задач на основе изученного материала:

#### **Задача 1: Бот для заказа доставки пиццы**
Пользователь должен:
1. Выбрать размер пиццы.
2. Указать количество.
3. Ввести адрес доставки.
4. После ввода адреса бот отправляет меню (в виде медиа-группы с картинками и описанием), чтобы пользователь мог выбрать дополнительные блюда.

---

#### **Задача 2: Бот для создания отчёта**
Бот собирает от пользователя:
1. Название проекта.
2. Краткое описание проекта.
3. Документ с подробным описанием (PDF или Word файл).  
После загрузки файла бот отправляет всё это одним сообщением, включая сам файл и собранные данные, используя `MediaGroup`.

---

### Решение задачи №2 (пример):

#### **Постановка задачи:**
Создать бота, который:
1. Спрашивает название проекта.
2. Запрашивает краткое описание проекта.
3. Просит загрузить документ с подробным описанием.  
После этого бот отправляет ответное сообщение в формате `MediaGroup`, состоящее из текстового описания и загруженного файла.

---

#### **Реализация:**

```python
from aiogram import Bot, Dispatcher, types
from aiogram.types import InputFile, MediaGroup
from aiogram.contrib.fsm_storage.memory import MemoryStorage
from aiogram.dispatcher.filters.state import State, StatesGroup
from aiogram.dispatcher import FSMContext
from aiogram.utils import executor

# Инициализация бота
API_TOKEN = 'ВАШ_ТОКЕН'
bot = Bot(token=API_TOKEN)
storage = MemoryStorage()
dp = Dispatcher(bot, storage=storage)

# Состояния бота
class ProjectReport(StatesGroup):
    waiting_for_name = State()
    waiting_for_description = State()
    waiting_for_document = State()

# Хэндлер команды /start
@dp.message_handler(commands='start')
async def start(message: types.Message):
    await message.answer("Привет! Давайте начнем с названия проекта.")
    await ProjectReport.waiting_for_name.set()

# Хэндлер получения названия проекта
@dp.message_handler(state=ProjectReport.waiting_for_name)
async def get_name(message: types.Message, state: FSMContext):
    await state.update_data(name=message.text)
    await message.answer("Отлично! Теперь напишите краткое описание проекта.")
    await ProjectReport.waiting_for_description.set()

# Хэндлер получения описания проекта
@dp.message_handler(state=ProjectReport.waiting_for_description)
async def get_description(message: types.Message, state: FSMContext):
    await state.update_data(description=message.text)
    await message.answer("Загрузите документ с подробным описанием проекта.")
    await ProjectReport.waiting_for_document.set()

# Хэндлер получения документа
@dp.message_handler(content_types=types.ContentType.DOCUMENT, state=ProjectReport.waiting_for_document)
async def get_document(message: types.Message, state: FSMContext):
    document = message.document
    await state.update_data(document=document.file_id)

    # Получение всех данных из состояния
    data = await state.get_data()

    # Формирование MediaGroup
    media = MediaGroup()
    media.attach_text(f"Название проекта: {data['name']}\n"
                      f"Описание: {data['description']}")
    media.attach_document(document.file_id, caption="Подробный документ")

    # Отправка MediaGroup
    await bot.send_media_group(chat_id=message.chat.id, media=media)

    await state.finish()

# Запуск бота
if __name__ == '__main__':
    executor.start_polling(dp, skip_updates=True)
```

---

#### **Пояснение к решению:**
1. **Состояния:**
   - Созданы три состояния для последовательного запроса данных.
   - В каждом состоянии бот ожидает определённый тип информации от пользователя.

2. **MediaGroup:**
   - После сбора данных бот формирует `MediaGroup`, включающий текстовую часть (название и описание проекта) и документ, загруженный пользователем.

3. **Безопасность:**
   - `FSMContext` используется для хранения промежуточных данных, что позволяет обрабатывать несколько пользователей одновременно.

4. **Отправка документа:**
   - Документ отправляется через `attach_document` в `MediaGroup`.
