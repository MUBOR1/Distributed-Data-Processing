# Лабораторная работа №8: Знакомство с NoSQL БД (на примере Redis)

**Цель работы:** Освоить основы работы с NoSQL базой данных Redis. Получить практические навыки выполнения основных операций CRUD, работы с различными типами данных и сравнения производительности с реляционной СУБД.

**Стек технологий:**
*   **База данных:** Redis
*   **Язык программирования:** Python 3
*   **Библиотека:** redis-py
*   **Реляционная СУБД:** SQLite (для сравнения)
*   **ОС:** Linux (Ubuntu) или Windows
*   **Инструменты:** Redis CLI, Python IDE

### Теоретическая часть (краткое содержание):

**Redis** — высокопроизводительная NoSQL база данных типа «ключ-значение» с возможностью хранения различных структур данных.

**Ключевые особенности Redis:**
*   **In-memory data storage:** Данные хранятся в оперативной памяти
*   **Поддержка структур данных:** строки, списки, множества, хеши, сортированные множества
*   **Персистентность:** Возможность сохранения данных на диск
*   **Репликация:** Поддержка мастер-слейв репликации
*   **Высокая производительность:** До сотен тысяч операций в секунду

**Сравнение с реляционными БД:**
*   **Гибкость схемы данных:** Отсутствие жесткой схемы
*   **Производительность:** Значительное преимущество для определенных сценариев
*   **Масштабируемость:** Простое горизонтальное масштабирование

### Задание на практическую реализацию:

#### Часть 1: Установка и настройка Redis

1.  **Установите Redis:**
    ```bash
    # Для Ubuntu
    sudo apt update
    sudo apt install redis-server
    sudo systemctl start redis-server
    sudo systemctl enable redis-server
    ```

2.  **Проверьте работу Redis:**
    ```bash
    redis-cli ping
    # Должен ответить: PONG
    ```

#### Часть 2: Базовые операции CRUD с помощью redis-cli

1.  **Работа со строками:**
    ```bash
    # Создание
    SET user:1000 "John Doe"
    SET user:1001 "Jane Smith"
    
    # Чтение
    GET user:1000
    
    # Обновление
    SET user:1000 "John Updated"
    
    # Удаление
    DEL user:1001
    ```

2.  **Работа с хешами:**
    ```bash
    # Создание хеша
    HSET user:profile:1000 name "John" age 30 email "john@example.com"
    
    # Чтение всех полей
    HGETALL user:profile:1000
    
    # Чтение отдельного поля
    HGET user:profile:1000 name
    ```

#### Часть 3: Работа с Redis через Python

1.  **Установите библиотеку redis-py:**
    ```bash
    pip install redis
    ```

2.  **Подключение к Redis:**
    ```python
    import redis
    import time
    
    # Создание подключения
    r = redis.Redis(
        host='localhost',
        port=6379,
        db=0,
        decode_responses=True  # для автоматического декодирования строк
    )
    ```

3.  **Реализация CRUD операций:**
    ```python
    # Create
    r.set('product:1000', 'Laptop')
    r.hset('product:details:1000', mapping={
        'name': 'Laptop', 
        'price': '999.99',
        'category': 'electronics'
    })
    
    # Read
    product = r.get('product:1000')
    details = r.hgetall('product:details:1000')
    
    # Update
    r.hset('product:details:1000', 'price', '899.99')
    
    # Delete
    r.delete('product:1000')
    r.delete('product:details:1000')
    ```

#### Часть 4: Работа со списками и множествами

1.  **Операции со списками:**
    ```python
    # Добавление в список
    r.lpush('recent:users', 'user1000')
    r.lpush('recent:users', 'user1001')
    
    # Получение элементов
    recent_users = r.lrange('recent:users', 0, 10)
    ```

2.  **Операции с множествами:**
    ```python
    # Добавление во множество
    r.sadd('product:categories', 'electronics', 'books', 'clothing')
    
    # Проверка принадлежности
    is_member = r.sismember('product:categories', 'electronics')
    
    # Получение всех элементов
    categories = r.smembers('product:categories')
    ```

#### Часть 5: Сравнение производительности с SQLite

1.  **Тест вставки 1000 записей:**
    ```python
    def test_redis_insertion(n=1000):
        start_time = time.time()
        for i in range(n):
            r.set(f'test:{i}', f'value{i}')
        return time.time() - start_time
    
    def test_sqlite_insertion(n=1000):
        # Реализация вставки в SQLite
        pass
    
    redis_time = test_redis_insertion(1000)
    sqlite_time = test_sqlite_insertion(1000)
    
    print(f"Redis: {redis_time:.4f} сек")
    print(f"SQLite: {sqlite_time:.4f} сек")
    print(f"Разница: {sqlite_time/redis_time:.2f}x")
    ```

2.  **Тест чтения 1000 записей:**
    ```python
    def test_redis_read(n=1000):
        start_time = time.time()
        for i in range(n):
            value = r.get(f'test:{i}')
        return time.time() - start_time
    ```

### Требования к оформлению и отчёту:

1.  **Код:** Полные версии Python-скриптов для работы с Redis и SQLite
2.  **Скриншоты:**
    *   Результаты выполнения команд в redis-cli
    *   Результаты тестов производительности
    *   Примеры работы с различными типами данных
3.  **Отчёт:** Включите:
    *   Сравнительную таблицу производительности
    *   Анализ преимуществ и недостатков Redis
    *   Рекомендации по использованию Redis vs SQLite
    *   Примеры использования различных структур данных

### Критерии оценки:

*   **Удовлетворительно (3):** Установлен Redis, выполнены базовые операции CRUD
*   **Хорошо (4):** Реализована работа с различными типами данных, проведено сравнение производительности
*   **Отлично (5):** Проведен комплексный анализ, даны обоснованные рекомендации по использованию

---

**Пример тестовых данных для сравнения:**
```python
# Генерация тестовых данных
test_data = [
    {'id': i, 'name': f'Product {i}', 'price': round(i * 1.5, 2)}
    for i in range(1000)
]
```

**Рекомендуемые метрики для сравнения:**
*   Время вставки 1000 записей
*   Время чтения 1000 записей
*   Потребление памяти
*   Простота реализации сложных запросов