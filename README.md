Настоящий проект реализует ETL pipeline для обработки веб-логов с использованием Apache Spark 3.4.4, PostgreSQL, Python и FastAPI.

Непосредственно pipeline выполняет:
- загрузку логов в PostgreSQL (RAW слой)
- full load через Spark
- incremental load через Spark с использованием High Watermark (HWM)
- запись трансформированных данных в витрину dm_web_logs
- запуск ETL через REST API

Архитектура такова:

access.log -> Python Loader -> PostgreSQL (raw_web_logs) -> Spark -> transform -> PostgreSQL (dm_web_logs)

Стек технологий:
- Python
- PostgreSQL
- Docker
- Apache Spark/PySpark 3.4.4
- FastAPI
- DBeaver

etl-web-log/
|—-app/
|      |--api/
|           |-main.py
|——--etl/
|       |—-transforms.py
|       |—-full_load.py
|       |—-incremental_load.py
|—-scripts/
|      |—load_kaggle_data.py
|—-data/
|    |--sample.log.
|    |--sample2.log
|—-jars/
|    |—postgresql.jar
|—-venv/
|—-README.md

Исходные данные загружаются в таблицу raw_web_logs.
Основные поля:
- ip
- timestamp
- method
- url
- protocol
- status_code
- response_size
- referer
- user_agent
- raw_line

В Spark реализованы следующие трансформации:
1) классификация HTTP-статусов в status_class
2) выделение event_date
3) выделение event_hour
4) определение is_bot по user_agent
5) фильтрация статических ресурсов (jpg, png, css, js, ico)

full_load.py читает все данные из raw_web_logs, применяет трансформации, записывает результат в dm_web_logs
Запуск файла:
- python app\etl\full_load.py

incremental_load.py читает last_hwm из таблицы etl_metadata, загружает только новые строки по условию timestamp > last_hwm, а также применяет трансформации, дописывает данные в dm_web_logs и обновляет last_hwm
Запуск файла:
- python app\etl\incremental_load.py

API
Запуск:
- uvicorn app.api.main:app --reload

Swagger UI:
http://127.0.0.1:8000/docs

Доступные эндпоинты:
POST /etl/full
POST /etl/incremental
GET /etl/status/{task_id}
GET /etl/history

Как запустить проект
1. Запустить PostgreSQL
- docker start pg-etl
2. Перейти в проект
- cd C:\etlproj-34\etl-web-logs (описываю через папку у себя на диске C)
3. Активировать виртуальное окружение
- venv\Scripts\activate
4. Установить зависимости
- python -m pip install pyspark==3.4.4 fastapi uvicorn psycopg2-binary
5. Загрузить raw-данные
- python scripts\load_kaggle_data.py
6. Запустить full load
- python app\etl\full_load.py
7. Запустить incremental load
- python app\etl\incremental_load.py
8. Запустить API
- uvicorn app.api.main:app --reload

Результаты
1) загружено около 10000 строк raw логов
2) после трансформаций получено около 9058 строк
3) реализован full load
4) реализован incremental load с HWM
5) реализован API
6) реализован сценарий повторного incremental без новых данных
7) реализован сценарий с подгрузкой новых данных через sample2.log
