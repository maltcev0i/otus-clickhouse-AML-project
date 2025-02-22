# otus-clickhouse-AML-project


## 

Просто выполните команду docker compose up в этой папке. В стеке используются следующие Docker-контейнеры:

d.timeplus.com/timeplus-io/proton:latest — потоковый SQL-движок.
clickhouse/clickhouse-server:latest
quay.io/cloudhut/owl-shop:latest — генератор данных. Owl Shop — это воображаемый интернет-магазин, который симулирует обмен данными между микросервисами через Apache Kafka.
docker.redpanda.com/redpandadata/redpanda — совместимая с Kafka шина потоковых сообщений.
docker.redpanda.com/redpandadata/console — веб-интерфейс для исследования данных в Kafka/Redpanda.
Когда все контейнеры будут запущены, в Redpanda автоматически создадутся несколько топиков с данными в реальном времени.

## Чтение данных из Redpanda, применение ETL и запись в ClickHouse
Откройте `proton client` в контейнере proton. Выполните следующий SQL-запрос для создания внешнего потока, который будет читать данные в реальном времени из Redpanda.

```sql
CREATE EXTERNAL STREAM frontend_events(raw string)
SETTINGS type='kafka',
         brokers='redpanda:9092',
         topic='owlshop-frontend-events';
```

Откройте `clickhouse client` в контейнере ClickHouse. Выполните следующий SQL-запрос для создания обычной таблицы MergeTree.

```sql
CREATE TABLE events
(
    _tp_time DateTime64(3),
    url String,
    method String,
    ip String
)
ENGINE=MergeTree()
PRIMARY KEY (_tp_time, url);
```

Вернитесь к `proton client` и выполните следующий SQL-запрос для создания внешней таблицы, которая будет подключаться к ClickHouse:
```sql
CREATE EXTERNAL TABLE ch_local
SETTINGS type='clickhouse',
         address='clickhouse:9000',
         table='events';
```

Затем создайте материализованное представление, которое будет читать данные из Redpanda, извлекать необходимые значения, преобразовывать IP-адрес в замаскированный md5 и отправлять данные во внешнюю таблицу. Таким образом, преобразованные данные будут непрерывно записываться в ClickHouse.

```sql
CREATE MATERIALIZED VIEW mv INTO ch_local AS
    SELECT now64() AS _tp_time,
           raw:requestedUrl AS url,
           raw:method AS method,
           lower(hex(md5(raw:ipAddress))) AS ip
    FROM frontend_events;
```

## Чтение данных из clickhouse

Вы можете выполнить следующий SQL-запрос для выборки данных из ClickHouse:

```sql
SELECT * FROM ch_local;
```

Или применить SQL-функции и группировку, например:

```sql
SELECT method, count() AS cnt FROM ch_local GROUP BY method
```

Обратите внимание, что Proton будет считывать все строки с выбранными столбцами из ClickHouse и выполнять агрегацию локально.