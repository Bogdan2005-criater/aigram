Модернизация задач с добавлением работы с API с использованием **Flask** предполагает создание серверной части для обработки данных, получаемых от бота, а также добавление API для обработки заказов и отчетов.

Предлагаю следующие шаги для улучшения задачи:

1. **Создание API на Flask** для обработки запросов от бота.
2. **Интеграция с API**: Бот будет отправлять данные (например, заказ пиццы или отчет о проекте) на сервер Flask через HTTP-запросы.
3. **Использование базы данных** (например, SQLite или PostgreSQL) для хранения информации о заказах или отчетах.(**НЕЛБЯЗАТЕЛЬНО, можно зранить в файле или в памяти**)

### Задача 1: Бот для заказа доставки пиццы с использованием Flask API

#### Описание:
Пользователь заказывает пиццу через бота, и данные о заказе отправляются на сервер Flask через API для дальнейшей обработки и подтверждения.

#### Шаги реализации:

1. **API для обработки заказов пиццы (Flask)**: Flask-сервер будет получать информацию о заказе (размер пиццы, количество и адрес) через POST-запросы.
2. **Telegram-бот**: Бот будет отправлять запросы на API Flask, чтобы создать заказ.

---

#### 1. Создание API с использованием Flask

Пример кода для создания API, который будет принимать данные о заказе пиццы.

```python
from flask import Flask, request, jsonify
from datetime import datetime

app = Flask(__name__)

# Временное хранилище для заказов (можно заменить на базу данных)
orders = []

@app.route('/api/create_order', methods=['POST'])
def create_order():
    data = request.json
    if not all(key in data for key in ('size', 'quantity', 'address')):
        return jsonify({"error": "Missing required fields"}), 400
    
    order = {
        'size': data['size'],
        'quantity': data['quantity'],
        'address': data['address'],
        'timestamp': datetime.now().strftime('%Y-%m-%d %H:%M:%S')
    }
    
    # Добавление заказа в хранилище
    orders.append(order)
    return jsonify({"message": "Order created successfully", "order": order}), 201

@app.route('/api/orders', methods=['GET'])
def get_orders():
    return jsonify(orders)

if __name__ == '__main__':
    app.run(debug=True)
```

**Объяснение:**
- `/api/create_order`: Принимает POST-запрос с данными о заказе в формате JSON.
- `/api/orders`: Возвращает все заказы, сделанные через API.

---

#### 2. Интеграция Telegram-бота с API Flask

Теперь, давайте изменим код Telegram-бота для того, чтобы он отправлял данные на сервер Flask.

Пример кода для бота:

```python
import requests
from aiogram import Bot, Dispatcher, types
from aiogram.types import MediaGroup
from aiogram.contrib.fsm_storage.memory import MemoryStorage
from aiogram.dispatcher import FSMContext
from aiogram.dispatcher.filters.state import State, StatesGroup
from aiogram.utils import executor

API_TOKEN = 'ВАШ_ТОКЕН'
bot = Bot(token=API_TOKEN)
dp = Dispatcher(bot, storage=MemoryStorage())

class PizzaOrder(StatesGroup):
    waiting_for_size = State()
    waiting_for_quantity = State()
    waiting_for_address = State()

API_URL = "http://localhost:5000/api/create_order"  # URL Flask API

@dp.message_handler(commands='start')
async def start(message: types.Message):
    await message.answer("Привет! Выберите размер пиццы: Маленькая, Средняя, Большая.")
    await PizzaOrder.waiting_for_size.set()

@dp.message_handler(state=PizzaOrder.waiting_for_size)
async def get_size(message: types.Message, state: FSMContext):
    await state.update_data(size=message.text)
    await message.answer("Введите количество пицц.")
    await PizzaOrder.waiting_for_quantity.set()

@dp.message_handler(state=PizzaOrder.waiting_for_quantity)
async def get_quantity(message: types.Message, state: FSMContext):
    await state.update_data(quantity=message.text)
    await message.answer("Введите адрес доставки.")
    await PizzaOrder.waiting_for_address.set()

@dp.message_handler(state=PizzaOrder.waiting_for_address)
async def get_address(message: types.Message, state: FSMContext):
    await state.update_data(address=message.text)
    data = await state.get_data()

    # Отправка данных на API Flask
    response = requests.post(API_URL, json=data)

    if response.status_code == 201:
        await message.answer(f"Ваш заказ принят!\nРазмер: {data['size']}\nКоличество: {data['quantity']}\nАдрес: {data['address']}")
    else:
        await message.answer("Произошла ошибка при обработке вашего заказа.")

    await state.finish()

if __name__ == '__main__':
    executor.start_polling(dp, skip_updates=True)
```

**Объяснение кода:**
- В этом коде бот поочередно запрашивает у пользователя размер пиццы, количество и адрес доставки.
- После того как все данные собраны, бот отправляет POST-запрос на API Flask с данными о заказе.
- В случае успешной отправки запроса (статус 201) бот информирует пользователя о принятии заказа.

---

### Задача 2: Бот для создания отчёта с использованием Flask API

#### Описание:
Пользователь создает отчет, загружая документ и вводя название и описание. Эти данные отправляются на сервер Flask через API для сохранения.

#### Шаги реализации:

1. **API для обработки отчетов (Flask)**: Flask-сервер будет получать данные о проекте и загрузки файлов через POST-запросы.
2. **Telegram-бот**: Бот будет отправлять запросы на сервер Flask с данными о проекте и документом.

---

#### 1. Создание API для обработки отчетов

```python
from flask import Flask, request, jsonify
import os
from werkzeug.utils import secure_filename

app = Flask(__name__)

UPLOAD_FOLDER = 'uploads/'
app.config['UPLOAD_FOLDER'] = UPLOAD_FOLDER

# Временное хранилище для отчетов
reports = []

# Убедимся, что директория существует
os.makedirs(UPLOAD_FOLDER, exist_ok=True)

@app.route('/api/create_report', methods=['POST'])
def create_report():
    data = request.form
    file = request.files.get('document')

    if not data.get('name') or not data.get('description') or not file:
        return jsonify({"error": "Missing required fields"}), 400

    # Сохраняем файл
    filename = secure_filename(file.filename)
    filepath = os.path.join(app.config['UPLOAD_FOLDER'], filename)
    file.save(filepath)

    report = {
        'name': data['name'],
        'description': data['description'],
        'document': filepath
    }

    reports.append(report)

    return jsonify({"message": "Report created successfully", "report": report}), 201

if __name__ == '__main__':
    app.run(debug=True)
```

**Объяснение:**
- `/api/create_report`: Принимает POST-запрос с полями `name`, `description` и документом.
- Сохраняет загруженный файл на сервере и добавляет данные в отчет.

---

#### 2. Интеграция Telegram-бота с API Flask


**Объяснение кода:**
- Бот собирает название, описание и документ.
- После получения документа, бот отправляет его вместе с другими данными на сервер Flask через POST-запрос.

---

### Заключение

В этой модернизированной версии задания мы добавили серверную часть с использованием **Flask API**. Бот теперь отправляет данные о заказах пиццы или отчетах на сервер Flask для обработки. Это позволяет разделить логику взаимодействия с пользователем и хранение/обработку данных, улучшая масштабируемость приложения.
