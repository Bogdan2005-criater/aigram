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


Вот подробное объяснение всех элементов кода:

### **Импорты**
```python
from aiogram import Bot, Dispatcher, types
from aiogram.types import InputFile, MediaGroup
from aiogram.contrib.fsm_storage.memory import MemoryStorage
from aiogram.dispatcher.filters.state import State, StatesGroup
from aiogram.dispatcher import FSMContext
from aiogram.utils import executor
```
- **aiogram** — библиотека для создания Telegram-ботов.
- **Bot** — объект бота, через который осуществляется взаимодействие с Telegram API.
- **Dispatcher** — объект для управления хэндлерами (обработчиками событий).
- **types** — модуль, содержащий различные Telegram-объекты, например `Message`, `InputFile`, и др.
- **InputFile** — используется для отправки файлов в Telegram.
- **MediaGroup** — объект для группировки сообщений с мультимедиа (тексты, документы, фото и т.д.).
- **MemoryStorage** — хранилище данных на основе памяти, используется для хранения состояний пользователя.
- **State, StatesGroup** — используются для реализации конечных автоматов.
- **FSMContext** — объект, обеспечивающий работу конечного автомата и доступ к данным состояния.
- **executor** — модуль для запуска бота.

---

### **Инициализация**
```python
API_TOKEN = 'ВАШ_ТОКЕН'
bot = Bot(token=API_TOKEN)
storage = MemoryStorage()
dp = Dispatcher(bot, storage=storage)
```
- **API_TOKEN** — токен вашего бота, который можно получить у [@BotFather](https://t.me/botfather).
- **Bot** — создаёт объект бота с указанным токеном.
- **MemoryStorage** — создаёт временное хранилище для хранения данных о пользователях.
- **Dispatcher** — объект для управления взаимодействием между ботом и пользователем; принимает бот и хранилище.

---

### **Создание состояний**
```python
class ProjectReport(StatesGroup):
    waiting_for_name = State()
    waiting_for_description = State()
    waiting_for_document = State()
```
- **StatesGroup** — класс, который объединяет состояния.
- **waiting_for_name, waiting_for_description, waiting_for_document** — состояния, которые последовательно используются ботом для выполнения задачи. Например:
  - `waiting_for_name` — бот ожидает от пользователя ввода названия проекта.
  - `waiting_for_description` — бот ожидает краткое описание проекта.
  - `waiting_for_document` — бот ожидает загрузки документа.

---

### **Хэндлер команды /start**
```python
@dp.message_handler(commands='start')
async def start(message: types.Message):
    await message.answer("Привет! Давайте начнем с названия проекта.")
    await ProjectReport.waiting_for_name.set()
```
- **@dp.message_handler(commands='start')** — хэндлер для обработки команды `/start`.
- **message: types.Message** — объект, содержащий сообщение пользователя.
- **message.answer()** — отправка текстового сообщения в ответ.
- **await ProjectReport.waiting_for_name.set()** — перевод бота в состояние `waiting_for_name`.

---

### **Хэндлер получения названия проекта**
```python
@dp.message_handler(state=ProjectReport.waiting_for_name)
async def get_name(message: types.Message, state: FSMContext):
    await state.update_data(name=message.text)
    await message.answer("Отлично! Теперь напишите краткое описание проекта.")
    await ProjectReport.waiting_for_description.set()
```
- **@dp.message_handler(state=ProjectReport.waiting_for_name)** — хэндлер срабатывает только в состоянии `waiting_for_name`.
- **state.update_data(name=message.text)** — сохраняет введённое пользователем название проекта в хранилище.
- **message.answer()** — отправляет сообщение с просьбой ввести описание.
- **await ProjectReport.waiting_for_description.set()** — переключает состояние на `waiting_for_description`.

---

### **Хэндлер получения описания проекта**
```python
@dp.message_handler(state=ProjectReport.waiting_for_description)
async def get_description(message: types.Message, state: FSMContext):
    await state.update_data(description=message.text)
    await message.answer("Загрузите документ с подробным описанием проекта.")
    await ProjectReport.waiting_for_document.set()
```
- Аналогично предыдущему хэндлеру:
  - Сохраняет описание проекта.
  - Переключает состояние на `waiting_for_document`.

---

### **Хэндлер получения документа**
```python
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
```
- **content_types=types.ContentType.DOCUMENT** — хэндлер активируется только при получении документа.
- **document = message.document** — сохраняет документ, который отправил пользователь.
- **state.update_data(document=document.file_id)** — сохраняет ID документа в хранилище.
- **data = await state.get_data()** — извлекает все данные, которые были сохранены в текущей сессии.
- **MediaGroup()** — создаёт группу мультимедиа, чтобы отправить несколько объектов одним сообщением.
  - `attach_text()` — добавляет текст в группу.
  - `attach_document()` — добавляет документ в группу.
- **bot.send_media_group()** — отправляет сообщение с группой мультимедиа.
- **state.finish()** — завершает работу конечного автомата, очищает данные.

---

### **Запуск бота**
```python
if __name__ == '__main__':
    executor.start_polling(dp, skip_updates=True)
```
- **executor.start_polling()** — запускает бота в режиме получения обновлений от Telegram.
- **skip_updates=True** — пропускает необработанные сообщения, если бот был отключён.

---

Этот код организует работу бота так, чтобы пользователи последовательно вводили данные, а бот их обрабатывал и сохранял, избегая конфликтов между разными пользователями.
