# Лабораторная работа №6: Работа с Apache Kafka

**Цель работы:** Освоить основы работы с Apache Kafka как платформой для обмена сообщениями. Получить практические навыки создания топиков, написания производителей (producers) и потребителей (consumers) на Python, а также понимания архитектуры Kafka.

**Стек технологий:**
*   **Платформа:** Apache Kafka
*   **Язык программирования:** Python 3
*   **Библиотека:** kafka-python
*   **ОС:** Linux (Ubuntu) или Windows с WSL2
*   **Инструменты:** Kafka CLI tools, Python IDE

### Теоретическая часть (краткое содержание):

**Apache Kafka** — распределённая платформа для обмена сообщениями и потоковой обработки данных.

**Ключевые понятия:**
*   **Топик (Topic):** Категория или имя потока сообщений
*   **Партиция (Partition):** Топики разделяются на партиции для параллельной обработки
*   **Производитель (Producer):** Приложение, отправляющее сообщения в топик
*   **Потребитель (Consumer):** Приложение, читающее сообщения из топика
*   **Consumer Group:** Группа потребителей, совместно обрабатывающих сообщения
*   **Брокер (Broker):** Сервер Kafka в кластере
*   **Zookeeper:** Сервис для координации и управления кластером

### Задание на практическую реализацию:

#### Часть 1: Установка и настройка Kafka

1.  **Установите Apache Kafka** (можно использовать Docker или standalone версию)
2.  **Запустите Zookeeper и Kafka broker:**
    ```bash
    # Запуск Zookeeper
    bin/zookeeper-server-start.sh config/zookeeper.properties
    
    # Запуск Kafka broker
    bin/kafka-server-start.sh config/server.properties
    ```

#### Часть 2: Работа с топиками через CLI

1.  **Создайте топик:** `kafka-logs`
    ```bash
    kafka-topics.sh --create --topic kafka-logs --bootstrap-server localhost:9092 --partitions 3 --replication-factor 1
    ```

2.  **Проверьте список топиков:**
    ```bash
    kafka-topics.sh --list --bootstrap-server localhost:9092
    ```

3.  **Посмотрите информацию о топике:**
    ```bash
    kafka-topics.sh --describe --topic kafka-logs --bootstrap-server localhost:9092
    ```

#### Часть 3: Написание Producer на Python

1.  **Установите библиотеку:**
    ```bash
    pip install kafka-python
    ```

2.  **Создайте producer:** Генерируйте логи веб-сервера в реальном времени
    ```python
    from kafka import KafkaProducer
    import json
    import time
    import random
    
    producer = KafkaProducer(
        bootstrap_servers='localhost:9092',
        value_serializer=lambda v: json.dumps(v).encode('utf-8')
    )
    
    log_types = ['INFO', 'WARNING', 'ERROR', 'DEBUG']
    
    for i in range(100):
        log_entry = {
            'timestamp': time.time(),
            'level': random.choice(log_types),
            'message': f'Log message number {i}',
            'service': 'web-server'
        }
        producer.send('kafka-logs', value=log_entry)
        time.sleep(0.1)
    ```

#### Часть 4: Написание Consumer на Python

1.  **Создайте consumer:** Читайте и обрабатывайте логи
    ```python
    from kafka import KafkaConsumer
    import json
    
    consumer = KafkaConsumer(
        'kafka-logs',
        bootstrap_servers='localhost:9092',
        auto_offset_reset='earliest',
        value_deserializer=lambda x: json.loads(x.decode('utf-8'))
    )
    
    for message in consumer:
        log_data = message.value
        print(f"Received log: {log_data}")
        # Добавьте здесь обработку логов
    ```

#### Часть 5: Анализ и обработка сообщений

1.  **Модифицируйте consumer** для фильтрации сообщений:
    *   Фильтруйте только ERROR логи
    *   Подсчитывайте количество логов каждого типа
    *   Сохраняйте логи в файл

2.  **Реализуйте статистику** в реальном времени:
    *   Количество сообщений в минуту
    *   Распределение по типам логов
    *   Активность по сервисам

### Требования к оформлению и отчёту:

1.  **Код:** Предоставьте полные версии producer.py и consumer.py
2.  **Скриншоты:**
    *   Создание топика через CLI
    *   Работа producer и consumer в терминале
    *   Примеры отправляемых и получаемых сообщений
3.  **Отчёт:** Включите:
    *   Описание архитектуры вашего решения
    *   Объяснение выбора количества партиций
    *   Анализ гарантий доставки сообщений
    *   Проблемы, с которыми столкнулись, и их решение

### Критерии оценки:

*   **Удовлетворительно (3):** Установлен Kafka, создан топик, написан простой producer
*   **Хорошо (4):** Реализованы producer и consumer, сообщения передаются корректно
*   **Отлично (5):** Реализована продвинутая обработка сообщений, сбор статистики, анализ работы системы

---

**Пример лога для отправки:**
```json
{
  "timestamp": 1642256789.123,
  "level": "ERROR",
  "message": "Database connection failed",
  "service": "auth-service",
  "ip": "192.168.1.1"
}
```

**Рекомендуемые настройки producer:**
```python
producer = KafkaProducer(
    bootstrap_servers='localhost:9092',
    value_serializer=lambda v: json.dumps(v).encode('utf-8'),
    acks='all',  # Гарантия доставки
    retries=3    # Попытки повтора
)
```