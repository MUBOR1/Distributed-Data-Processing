# Лабораторная работа №9 (Часть 2): Миграция и анализ большого набора данных с помощью Spark SQL

**Тема:** Миграция и анализ большого набора данных с помощью Spark SQL

**Цель работы:** Освоить методы миграции больших объемов данных из реляционных баз данных в распределенные системы хранения и выполнения сложного анализа с использованием Spark SQL. Разработать ETL-пайплайн для преобразования и обогащения данных.

**Стек технологий:**
- **Платформы:** Apache Spark, PostgreSQL, HDFS
- **Языки программирования:** Python (PySpark), SQL
- **Библиотеки:** pyspark, psycopg2, pandas
- **ОС:** Linux (Ubuntu) или Windows с WSL2
- **Инструменты:** Jupyter Notebook, pgAdmin, Spark Web UI

---

### Теоретическая часть

**Архитектура решения:**
1. **Extract:** Извлечение данных из PostgreSQL с использованием JDBC
2. **Transform:** Преобразование и обогащение данных с помощью Spark SQL
3. **Load:** Загрузка результатов в HDFS и аналитические таблицы
4. **Analysis:** Выполнение сложных аналитических запросов

**Ключевые аспекты:**
- Обработка datasets объемом 10+ GB
- Оптимизация производительности Spark Jobs
- Использование партиционирования и кэширования
- Анализ временных рядов и оконные функции

---

### Задание на практическую реализацию

#### Этап 1: Подготовка данных и инфраструктуры (3 часа)

1. **Настройка PostgreSQL и генерация тестовых данных:**
```sql
-- Создание базы данных
CREATE DATABASE sales_db;

-- Таблица продаж
CREATE TABLE sales (
    sale_id SERIAL PRIMARY KEY,
    product_id INTEGER,
    customer_id INTEGER,
    sale_date DATE,
    amount DECIMAL(10,2),
    quantity INTEGER,
    region VARCHAR(50),
    category VARCHAR(50)
);

-- Генерация 10 млн записей
INSERT INTO sales (...)
SELECT 
    generate_series(1, 10000000),
    floor(random() * 1000) + 1,
    floor(random() * 100000) + 1,
    CURRENT_DATE - (random() * 365)::integer,
    random() * 1000,
    floor(random() * 10) + 1,
    CASE floor(random() * 5)
        WHEN 0 THEN 'North' WHEN 1 THEN 'South' 
        WHEN 2 THEN 'East' WHEN 3 THEN 'West' 
        ELSE 'Central' END,
    CASE floor(random() * 6)
        WHEN 0 THEN 'Electronics' WHEN 1 THEN 'Clothing'
        WHEN 2 THEN 'Food' WHEN 3 THEN 'Books'
        WHEN 4 THEN 'Sports' ELSE 'Home' END
FROM generate_series(1, 10000000);
```

#### Этап 2: Миграция данных из PostgreSQL в HDFS (4 часа)

**Реализуйте ETL-пайплайн для миграции:**
```python
from pyspark.sql import SparkSession
from pyspark.sql.functions import *

spark = SparkSession.builder \
    .appName("PostgreSQLMigration") \
    .config("spark.jars", "/path/to/postgresql-42.5.0.jar") \
    .config("spark.sql.adaptive.enabled", "true") \
    .config("spark.sql.adaptive.coalescePartitions.enabled", "true") \
    .getOrCreate()

# Чтение данных из PostgreSQL
jdbc_url = "jdbc:postgresql://localhost:5432/sales_db"
properties = {
    "user": "username",
    "password": "password",
    "driver": "org.postgresql.Driver"
}

# Чтение с партиционированием
sales_df = spark.read \
    .jdbc(url=jdbc_url,
          table="(SELECT *, MOD(sale_id, 10) as partition_key FROM sales) as partitioned_sales",
          column="partition_key",
          lowerBound=0,
          upperBound=10,
          numPartitions=10,
          properties=properties)

# Оптимизация схемы данных
optimized_df = sales_df \
    .withColumn("sale_year", year(col("sale_date"))) \
    .withColumn("sale_month", month(col("sale_date"))) \
    .withColumn("sale_quarter", quarter(col("sale_date")))

# Сохранение в HDFS в формате Parquet
optimized_df.write \
    .mode("overwrite") \
    .partitionBy("sale_year", "sale_month") \
    .format("parquet") \
    .save("hdfs://localhost:9000/data/sales_parquet")
```

#### Этап 3: Сложный анализ данных с помощью Spark SQL (5 часов)

**Выполните комплексный анализ данных:**
```python
# Создание временного представления
optimized_df.createOrReplaceTempView("sales")

# 1. Анализ продаж по регионам и категориям
region_analysis = spark.sql("""
    SELECT 
        region,
        category,
        COUNT(*) as total_sales,
        SUM(amount) as total_revenue,
        AVG(amount) as avg_sale_amount,
        PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY amount) as median_sale_amount
    FROM sales
    WHERE sale_year = 2023
    GROUP BY region, category
    ORDER BY total_revenue DESC
""")

# 2. Анализ временных рядов с оконными функциями
time_series_analysis = spark.sql("""
    SELECT 
        sale_date,
        region,
        SUM(amount) OVER (
            PARTITION BY region 
            ORDER BY sale_date 
            ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
        ) as rolling_7d_revenue,
        AVG(amount) OVER (
            PARTITION BY region, category
            ORDER BY sale_date 
            RANGE BETWEEN INTERVAL 30 DAYS PRECEDING AND CURRENT ROW
        ) as avg_30d_category_revenue
    FROM sales
    WHERE sale_date >= '2023-01-01'
""")

# 3. Выявление трендов и аномалий
trend_analysis = spark.sql("""
    WITH daily_stats AS (
        SELECT 
            sale_date,
            SUM(amount) as daily_revenue,
            COUNT(*) as daily_transactions
        FROM sales
        GROUP BY sale_date
    ),
    moving_avg AS (
        SELECT
            sale_date,
            daily_revenue,
            AVG(daily_revenue) OVER (
                ORDER BY sale_date 
                ROWS BETWEEN 7 PRECEDING AND CURRENT ROW
            ) as revenue_7d_ma,
            STDDEV(daily_revenue) OVER (
                ORDER BY sale_date 
                ROWS BETWEEN 30 PRECEDING AND CURRENT ROW
            ) as revenue_30d_stddev
        FROM daily_stats
    )
    SELECT
        sale_date,
        daily_revenue,
        revenue_7d_ma,
        (daily_revenue - revenue_7d_ma) / revenue_30d_stddev as revenue_z_score
    FROM moving_avg
    WHERE ABS((daily_revenue - revenue_7d_ma) / revenue_30d_stddev) > 2.0
""")
```

#### Этап 4: Оптимизация производительности (3 часа)

**Реализуйте оптимизацию запросов:**
```python
# Кэширование часто используемых данных
spark.sql("CACHE TABLE sales")

# Настройка параметров Spark для оптимизации
spark.conf.set("spark.sql.adaptive.enabled", "true")
spark.conf.set("spark.sql.adaptive.coalescePartitions.enabled", "true")
spark.conf.set("spark.sql.adaptive.skewJoin.enabled", "true")

# Создание индексов через Bloom Filter
spark.sql("""
    CREATE BLOOMFILTER INDEX
    ON TABLE sales
    FOR COLUMNS(region, category)
""")

# Анализ плана выполнения
explain_plan = spark.sql("""
    EXPLAIN EXTENDED
    SELECT region, category, SUM(amount) 
    FROM sales 
    GROUP BY region, category
""")
explain_plan.show(truncate=False)
```

#### Этап 5: Визуализация результатов анализа (3 часа)

**Создайте дашборд для визуализации результатов:**
```python
import matplotlib.pyplot as plt
import seaborn as sns
import pandas as pd

# Конвертация результатов в Pandas DataFrame для визуализации
region_analysis_pd = region_analysis.toPandas()

# 1. Heatmap продаж по регионам и категориям
plt.figure(figsize=(12, 8))
pivot_table = region_analysis_pd.pivot_table(
    values='total_revenue', 
    index='region', 
    columns='category', 
    fill_value=0
)
sns.heatmap(pivot_table, annot=True, fmt='.0f', cmap='YlOrRd')
plt.title('Revenue by Region and Category')
plt.savefig('revenue_heatmap.png')

# 2. Временные ряды продаж
time_series_pd = time_series_analysis.limit(1000).toPandas()
plt.figure(figsize=(14, 6))
for region in time_series_pd['region'].unique():
    region_data = time_series_pd[time_series_pd['region'] == region]
    plt.plot(region_data['sale_date'], region_data['rolling_7d_revenue'], label=region)
plt.legend()
plt.title('7-Day Rolling Revenue by Region')
plt.savefig('time_series.png')
```

#### Этап 6: Мониторинг и оптимизация (2 часа)

**Реализуйте мониторинг производительности:**
```python
# Мониторинг ресурсов
def monitor_performance():
    storage_info = spark.sparkContext.getRDDStorageInfo()
    for info in storage_info:
        print(f"RDD {info.name}: {info.memUsed} MB memory used")
    
    # Анализ времени выполнения
    for query in spark.streams.active:
        print(f"Query {query.name}: {query.lastProgress}")

# Бенчмаркинг производительности
import time
def benchmark_query(query_str, iterations=3):
    times = []
    for i in range(iterations):
        start_time = time.time()
        spark.sql(query_str).count()
        times.append(time.time() - start_time)
    return sum(times) / iterations
```

---

### Требования к отчёту

1. **Архитектурная схема** ETL-пайплайна
2. **Графики производительности:** время выполнения запросов, использование ресурсов
3. **Анализ результатов:** сравнение производительности до и после оптимизации
4. **Визуализации:** дашборды с результатами анализа
5. **Рекомендации** по оптимизации больших datasets

---

### Критерии оценки

**Отлично (5):**
- Обработка 10+ GB данных с оптимизированной производительностью
- Реализация сложных аналитических запросов с оконными функциями
- Качественная визуализация результатов
- Глубокий анализ производительности и оптимизации

**Хорошо (4):**
- Обработка 5+ GB данных
- Реализация основных аналитических запросов
- Базовая визуализация данных

**Удовлетворительно (3):**
- Успешная миграция данных из PostgreSQL
- Выполнение базовых аналитических запросов

---

### Рекомендуемая литература (на русском языке)

1. **Захария М., Уэнделл П.** - "Изучаем Spark: молниеносный анализ данных" (2017)
2. **Сафонов В.О.** - "Обработка больших данных с использованием Apache Spark" (2021)
3. **Камкина М.В.** - "Анализ больших данных: методы и технологии" (2020)
4. **Таненбаум Э., ван Стен М.** - "Распределенные системы. Принципы и парадигмы" (2019)
5. **Джанса Г., Монк Дж.** - "PostgreSQL для профессионалов" (2022)