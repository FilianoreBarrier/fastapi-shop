📬 Руководство по тестированию API в Postman
FastAPI Shop — Backend Testing Guide

⚙️ Начальная настройка
1. Создайте Environment
В Postman: Environments → Add
VariableValuebase_urlhttp://localhost:8000
Используйте переменную {{base_url}} во всех запросах.
2. Запустите сервер
bashcd backend
pip install -r requirements.txt
python seed_data.py     # Заполнить БД тестовыми данными
python run.py           # Запустить сервер
Документация Swagger: http://localhost:8000/api/docs

🏥 Health Check
GET / — Главная страница
GET {{base_url}}/
Ожидаемый ответ 200 OK:
json{
  "message": "Welcome to fastapi-shop API",
  "docs": "api/docs"
}

GET /health — Проверка состояния
GET {{base_url}}/health
Ожидаемый ответ 200 OK:
json{
  "status": "healthy"
}

📁 Categories — Категории
GET /api/categories/ — Получить все категории
GET {{base_url}}/api/categories/
Ожидаемый ответ 200 OK:
json[
  { "id": 1, "name": "Electronics", "slug": "electronics" },
  { "id": 2, "name": "Clothing",    "slug": "clothing"    },
  { "id": 3, "name": "Books",       "slug": "books"       },
  { "id": 4, "name": "Home & Garden","slug": "home-garden" }
]

⚠️ Если БД пустая — вернётся пустой массив []. Запустите python seed_data.py.


GET /api/categories/{id} — Получить категорию по ID
GET {{base_url}}/api/categories/1
Ожидаемый ответ 200 OK:
json{
  "id": 1,
  "name": "Electronics",
  "slug": "electronics"
}
Тест — несуществующий ID:
GET {{base_url}}/api/categories/999
Ожидаемый ответ 404 Not Found:
json{
  "detail": "Category with id 999 not found "
}

📦 Products — Товары
GET /api/products/ — Получить все товары
GET {{base_url}}/api/products/
Ожидаемый ответ 200 OK:
json{
  "products": [
    {
      "id": 1,
      "name": "Wireless Headphones",
      "description": "High-quality wireless headphones...",
      "price": 299.99,
      "category_id": 1,
      "image_url": "https://images.unsplash.com/...",
      "created_at": "2024-01-01T00:00:00",
      "category": {
        "id": 1,
        "name": "Electronics",
        "slug": "electronics"
      }
    }
  ],
  "total": 13
}

GET /api/products/{id} — Получить товар по ID
GET {{base_url}}/api/products/1
Ожидаемый ответ 200 OK: — один объект товара с вложенной категорией.
Тест — несуществующий товар:
GET {{base_url}}/api/products/9999
Ожидаемый ответ 404 Not Found:
json{
  "detail": "Product with id 9999 not found"
}

GET /api/products/category/{id} — Товары по категории
GET {{base_url}}/api/products/category/1
Ожидаемый ответ 200 OK: — список товаров только категории Electronics.
Тест — несуществующая категория:
GET {{base_url}}/api/products/category/999
Ожидаемый ответ 404 Not Found:
json{
  "detail": "Category with id 999 not found"
}

⚠️ Известный баг маршрутизации: Запрос GET /api/products/category/1
может конфликтовать с GET /api/products/{product_id}, т.к. FastAPI
парсит "category" как product_id. Убедитесь, что маршрут /category/{id}
объявлен до /{product_id} в routes/products.py.


🛒 Cart — Корзина

Корзина не хранится на сервере — состояние передаётся в каждом запросе в теле.


POST /api/cart/add — Добавить товар в корзину
POST {{base_url}}/api/cart/add
Content-Type: application/json
Тело запроса:
json{
  "product_id": 1,
  "quantity": 2,
  "cart": {}
}
Ожидаемый ответ 200 OK:
json{
  "cart": { "1": 2 }
}
Добавить ещё один товар в существующую корзину:
json{
  "product_id": 2,
  "quantity": 1,
  "cart": { "1": 2 }
}
Ожидаемый ответ:
json{
  "cart": { "1": 2, "2": 1 }
}
Тест — несуществующий товар:
json{
  "product_id": 9999,
  "quantity": 1,
  "cart": {}
}
Ожидаемый ответ 404 Not Found:
json{
  "detail": "Product with id 9999 not found"
}
Тест — нулевое количество (валидация):
json{
  "product_id": 1,
  "quantity": 0,
  "cart": {}
}
Ожидаемый ответ 422 Unprocessable Entity — quantity должен быть > 0.

POST /api/cart/ — Получить детали корзины
POST {{base_url}}/api/cart/
Content-Type: application/json
Тело запроса (текущее состояние корзины):
json{ "1": 2, "2": 1 }
Ожидаемый ответ 200 OK:
json{
  "items": [
    {
      "product_id": 1,
      "name": "Wireless Headphones",
      "price": 299.99,
      "quantity": 2,
      "subtotal": 599.98,
      "image_url": "https://..."
    },
    {
      "product_id": 2,
      "name": "Smart Watch Pro",
      "price": 399.99,
      "quantity": 1,
      "subtotal": 399.99,
      "image_url": "https://..."
    }
  ],
  "total": 1000,
  "items_count": 3
}
Тест — пустая корзина:
json{}
Ожидаемый ответ:
json{
  "items": [],
  "total": 0.0,
  "items_count": 0
}

PUT /api/cart/update — Обновить количество товара
PUT {{base_url}}/api/cart/update
Content-Type: application/json
Тело запроса:
json{
  "product_id": 1,
  "quantity": 5,
  "cart": { "1": 2, "2": 1 }
}
Ожидаемый ответ 200 OK:
json{
  "cart": { "1": 5, "2": 1 }
}
Тест — товар отсутствует в корзине:
json{
  "product_id": 3,
  "quantity": 1,
  "cart": { "1": 2 }
}
Ожидаемый ответ 404 Not Found:
json{
  "detail": "Product with id 3 not found in cart"
}

DELETE /api/cart/remove/{product_id} — Удалить товар из корзины
DELETE {{base_url}}/api/cart/remove/1
Content-Type: application/json
Тело запроса (текущее состояние корзины):
json{
  "cart": { "1": 2, "2": 1 }
}
Ожидаемый ответ 200 OK:
json{
  "cart": { "2": 1 }
}
Тест — товара нет в корзине:
DELETE {{base_url}}/api/cart/remove/99
Тело:
json{
  "cart": { "1": 2 }
}
Ожидаемый ответ 404 Not Found:
json{
  "detail": "Product with id 99 not found in cart"
}

🔁 Типичный сценарий E2E
Следуйте этому порядку для полного тест-прогона:
ШагМетодURL1GET/health2GET/api/categories/3GET/api/products/4GET/api/products/15GET/api/products/category/16POST/api/cart/add (добавить товар 1)7POST/api/cart/add (добавить товар 2)8POST/api/cart/ (просмотр корзины)9PUT/api/cart/update (изменить кол-во)10DELETE/api/cart/remove/1 (удалить товар)11POST/api/cart/ (финальная корзина)

⚠️ Известные особенности
ПроблемаОписаниеtotal в корзинеОкругляется через round() — дробная часть теряетсяКорзина statelessСостояние не хранится на сервере, передаётся клиентом@app.on_event("startup")Устаревший API FastAPI, но работаетМаршруты /productsПорядок объявления маршрутов важен — /category/{id} должен быть выше /{id}name в CategoryBaseМинимум 5 символов — "Electronics" ✅, "Art" ❌

Версия API: FastAPI 0.136.1 · Python 3.x · SQLite