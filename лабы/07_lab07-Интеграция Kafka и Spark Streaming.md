# Лабораторная работа №7: Интеграция Kafka и Spark Streaming

**Цель работы:** Освоить интеграцию Apache Kafka с Apache Spark Streaming для обработки потоковых данных в реальном времени. Получить практические навыки настройки прямого чтения данных из топиков Kafka и их обработки с помощью Structured Streaming.

**Стек технологий:**
*   **Платформы:** Apache Kafka, Apache Spark
*   **Язык программирования:** Python (PySpark)
*   **Библиотеки:** kafka-python, pyspark
*   **ОС:** Linux (Ubuntu) или Windows с WSL2
*   **Инструменты:** Jupyter Notebook, Kafka CLI tools

### Теоретическая часть (краткое содержание):

**Интеграция Kafka-Spark** позволяет обрабатывать потоковые данные непосредственно из топиков Kafka с использованием мощностей Spark Streaming.

**Ключевые компоненты интеграции:**
*   **Kafka Direct API:** Прямое подключение к партициям топика
*   **Offset management:** Управление позицией чтения сообщений
*   **Exactly-once semantics:** Гарантия однократной обработки сообщений
*   **Structured Streaming integration:** Нативная поддержка в Spark Structured Streaming

**Преимущества подхода:**
*   Высокая пропускная способность
*   Отказоустойчивость
*   Масштабируемость
*   Поддержка различных форматов данных

### Задание на практическую реализацию:

#### Часть 1: Подготовка Kafka-инфраструктуры

1.  **Создайте топик для обработки:**
    ```bash
    kafka-topics.sh --create \
        --topic real-time-transactions \
        --bootstrap-server localhost:9092 \
        --partitions 3 \
        --replication-factor 1
    ```

2.  **Запустите Producer для генерации тестовых данных:**
    ```python
    from kafka import KafkaProducer
    import json
    import time
    import random
    
    producer = KafkaProducer(
        bootstrap_servers='localhost:9092',
        value_serializer=lambda v: json.dumps(v).encode('utf-8')
    )
    
    transaction_types = ['purchase', 'refund', 'transfer', 'withdrawal']
    
    for i in range(1000):
        transaction = {
            'transaction_id': f'txn_{i:04d}',
            'user_id': f'user_{random.randint(1, 100)}',
            'amount': round(random.uniform(10.0, 1000.0), 2),
            'type': random.choice(transaction_types),
            'timestamp': time.strftime('%Y-%m-%d %H:%M:%S'),
            'location': random.choice(['online', 'store'])
        }
        producer.send('real-time-transactions', value=transaction)
        time.sleep(0.5)
    ```

#### Часть 2: Настройка Spark Streaming для чтения из Kafka

1.  **Инициализация Spark Session с поддержкой Kafka:**
    ```python
    from pyspark.sql import SparkSession
    from pyspark.sql.functions import *
    from pyspark.sql.types import *
    
    spark = SparkSession.builder \
        .appName("KafkaSparkIntegration") \
        .config("spark.jars.packages", "org.apache.spark:spark-sql-kafka-0-10_2.12:3.3.0") \
        .getOrCreate()
    ```

2.  **Чтение потоковых данных из Kafka:**
    ```python
    kafka_df = spark.readStream \
        .format("kafka") \
        .option("kafka.bootstrap.servers", "localhost:9092") \
        .option("subscribe", "real-time-transactions") \
        .option("startingOffsets", "earliest") \
        .load()
    ```

#### Часть 3: Обработка потоковых данных

1.  **Парсинг JSON-сообщений:**
    ```python
    schema = StructType([
        StructField("transaction_id", StringType()),
        StructField("user_id", StringType()),
        StructField("amount", DoubleType()),
        StructField("type", StringType()),
        StructField("timestamp", StringType()),
        StructField("location", StringType())
    ])
    
    parsed_df = kafka_df.select(
        col("key").cast("string"),
        from_json(col("value").cast("string"), schema).alias("data"),
        col("timestamp").alias("processing_time")
    ).select("data.*", "processing_time")
    ```

2.  **Реализация агрегаций в реальном времени:**
    ```python
    # Агрегация по типам транзакций
    windowed_agg = parsed_df \
        .withWatermark("processing_time", "1 minute") \
        .groupBy(
            window(col("processing_time"), "5 minutes"),
            col("type")
        ) \
        .agg(
            count("*").alias("transaction_count"),
            sum("amount").alias("total_amount"),
            avg("amount").alias("avg_amount")
        )
    ```

#### Часть 4: Фильтрация и обогащение данных

1.  **Фильтрация подозрительных операций:**
    ```python
    suspicious_transactions = parsed_df.filter(
        (col("amount") > 500) & 
        (col("type") == "withdrawal") &
        (col("location") == "online")
    )
    ```

2.  **Обогащение данных дополнительной информацией:**
    ```python
    # Добавление категории транзакции по сумме
    enriched_df = parsed_df.withColumn(
        "amount_category",
        when(col("amount") < 100, "small")
        .when(col("amount") < 500, "medium")
        .otherwise("large")
    )
    ```

#### Часть 5: Вывод результатов

1.  **Настройка вывода в консоль:**
    ```python
    query = windowed_agg.writeStream \
        .outputMode("update") \
        .format("console") \
        .option("truncate", "false") \
        .start()
    
    query.awaitTermination()
    ```

2.  **Сохранение результатов в хранилище:**
    ```python
    output_query = suspicious_transactions.writeStream \
        .outputMode("append") \
        .format("parquet") \
        .option("path", "output/suspicious_transactions") \
        .option("checkpointLocation", "checkpoints/suspicious") \
        .start()
    ```

### Требования к оформлению и отчёту:

1.  **Код:** Полные версии всех скриптов (producer, spark processing)
2.  **Скриншоты:**
    *   Работа Kafka producer
    *   Результаты агрегации в консоли Spark
    *   Структура выходных данных
3.  **Отчёт:** Включите:
    *   Описание архитектуры решения
    *   Объяснение выбора параметров окон и watermark
    *   Анализ производительности системы
    *   Проблемы интеграции и их решение

### Критерии оценки:

*   **Удовлетворительно (3):** Настроено чтение из Kafka, базовая обработка данных
*   **Хорошо (4):** Реализованы агрегации и фильтрация, вывод в консоль
*   **Отлично (5):** Реализовано сохранение результатов, продвинутая обработка, анализ работы системы

---

**Пример вывода агрегации:**
```
Batch: 1
+------------------------------------------+----------+-----------------+-------------+------------------+
|window                                    |type      |transaction_count|total_amount |avg_amount        |
+------------------------------------------+----------+-----------------+-------------+------------------+
|{2024-01-15 10:00:00, 2024-01-15 10:05:00}|purchase  |45               |12567.89     |279.29            |
|{2024-01-15 10:00:00, 2024-01-15 10:05:00}|withdrawal|12               |6543.21      |545.27            |
+------------------------------------------+----------+-----------------+-------------+------------------+
```