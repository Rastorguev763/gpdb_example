# Greenplum example

## Разворачиваем Greenplum 5OSS в Docker

Установка Docker [тут](https://github.com/Rastorguev763/docker_example)

Docker image Greenplum [тут](https://hub.docker.com/r/kochanpivotal/gpdb5oss)

После того как образ Greenplum будет скачен, запускаем контейнер командой

```bash
docker run -it --hostname=gpdbsne --name gpdb5oss --publish 5432:5432 --publish 88:22 --volume .:/code kochanpivotal/gpdb5oss bin/bash
```

Дальше нужно запустить сервис GP и установить пароль для доступа к базе, выполняем следующие команды

```bash
docker exec -it gpdb5oss bash
```

```bash
su - gpadmin
```

```bash
gpstart
```

```bash
psql -h localhost -p 5432 -U gpadmin
```

```bash
ALTER USER gpadmin WITH PASSWORD 'gpadmin';
```

Теперь можно подключится к базе GreenPlum с помощью DBeaver или другой аналогичной программой используя следующие настройки:

*login* : **gpadmin**

*Password* : **gpadmin**

*host* : **localhost**

*port* : **5432**

*DB name* : **gpadmin**

Создаем таблицу фактов продаж произвольных товаров

```sql
CREATE TABLE sales_fact (
  sale_id SERIAL,
  product_id INT NOT NULL ,
  sale_date TIMESTAMP NOT NULL,
  sale_amount DECIMAL(10,2) NOT NULL
)
DISTRIBUTED BY (product_id)
PARTITION BY RANGE (sale_date)
(start (date '2023-01-01') inclusive
end (date '2024-01-01') exclusive
every (interval '1 month'));
```

Создаем таблицу измерений товаров и соеденяем с таблицей фактов продаж по **product_id**

```sql
CREATE TABLE dimension_table (
    product_id INT PRIMARY KEY,
    product_name VARCHAR(50) NOT NULL,
    cost DECIMAL(10,2) NOT NULL
);
```

Добавляем связь межну таблицами

```sql
ALTER TABLE sales_fact
ADD CONSTRAINT sales_fact_fk
FOREIGN KEY (product_id) REFERENCES dimension_table(product_id);
```

Добавляем данные в таблицы

```sql
INSERT INTO dimension_table (product_id, product_name, cost) VALUES
(1, 'Товар1', 50),
(2, 'Товар2', 70),
(3, 'Товар3', 60),
(4, 'Товар4', 80),
(5, 'Товар5', 90);
```

```sql
INSERT INTO sales_fact (product_id, sale_date, sale_amount) VALUES
(1, '2023-01-01', 100),
(2, '2023-02-01', 150),
(3, '2023-03-01', 120),
(4, '2023-04-01', 180),
(5, '2023-05-01', 200),
(1, '2023-06-02', 120),
(2, '2023-07-02', 90),
(3, '2023-08-02', 80),
(4, '2023-09-02', 160),
(5, '2023-10-02', 180),
(1, '2023-11-03', 80),
(2, '2023-12-03', 110),
(3, '2023-01-03', 130),
(4, '2023-02-03', 170),
(5, '2023-03-03', 150),
(1, '2023-04-04', 200),
(2, '2023-05-04', 120),
(3, '2023-06-04', 90),
(4, '2023-07-04', 140),
(5, '2023-08-04', 160);
```

Запрос, который рассчитывает сумму продаж определенного товара за определенную единицу времени.

```sql
SELECT 
    d.product_name,
    EXTRACT(MONTH FROM f.sale_date) AS sale_month,
    SUM(f.sale_amount) AS total_sales
FROM 
    sales_fact f
JOIN 
    dimension_table d ON f.product_id = d.product_id
WHERE 
    d.product_name = 'Товар1' AND EXTRACT(MONTH FROM f.sale_date) = 1
GROUP BY 
    d.product_name, sale_month;
```

План запроса

- Выбор данных из таблиц:

Сканирование таблицы ***sales_fact*** для получения данных о продажах.
Сканирование таблицы ***dimension_table*** для получения данных о товарах.

- Соединение таблиц:

Выполнение ***INNER JOIN*** между таблицами ***sales_fact*** и ***dimension_table*** на основе условия ***f.product_id = d.product_id***.

- Фильтрация данных:

Применение условий фильтрации: ***d.product_name = 'Товар1'*** и ***EXTRACT(MONTH FROM f.sale_date) = 1***.

- Группировка данных:

Группировка результатов по ***d.product_name*** и E***XTRACT(MONTH FROM f.sale_date)***.

- Вычисление агрегированных значений:

Расчет суммы продаж для каждой группы с использованием ***SUM(f.sale_amount) AS total_sales***.

- Выборка результата:

Выборка конечного результата, который включает ***product_name***, ***sale_month*** и ***total_sales***.
