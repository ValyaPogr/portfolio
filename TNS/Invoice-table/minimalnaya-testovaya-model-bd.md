# Минимальная тестовая модель БД

# Создание тестовой БД

Создаём минимальную тестовую модель, на которой можно проверить гипотезы 3НФ и зависимости.

Например, PostgreSQL:

```sql
-- TNS: тестовая БД для кейса "Нормализация таблицы накладных" PostgreSQL
DROP TABLE IF EXISTS invoice_line;
DROP TABLE IF EXISTS invoice;
DROP TABLE IF EXISTS product;
DROP TABLE IF EXISTS seller;

-- Продавец
CREATE TABLE seller (
    seller_id BIGSERIAL PRIMARY KEY,
    seller_name VARCHAR(255) NOT NULL
);

-- Товар
CREATE TABLE product (
    product_id BIGSERIAL PRIMARY KEY,
    product_code INTEGER NOT NULL UNIQUE,
    product_name VARCHAR(255) NOT NULL
);

-- Накладная
CREATE TABLE invoice (
    invoice_id BIGSERIAL PRIMARY KEY,
    -- Уникальность номера накладной пока не задана:
    -- зависит от бизнес-правила и области уникальности номера.
    invoice_number INTEGER NOT NULL,
    invoice_date DATE NOT NULL,
    seller_id BIGINT NOT NULL REFERENCES seller(seller_id)
);

-- Позиция накладной
CREATE TABLE invoice_line (
    invoice_line_id BIGSERIAL PRIMARY KEY,
    invoice_id BIGINT NOT NULL REFERENCES invoice(invoice_id),
    product_id BIGINT NOT NULL REFERENCES product(product_id),

    price NUMERIC(12, 2) NOT NULL,
    quantity NUMERIC(12, 3) NOT NULL,

    -- Для эксперимента пока храним стоимость.
    -- Позже проверим гипотезу: хранить или вычислять.
    total_amount NUMERIC(14, 2) GENERATED ALWAYS AS
        (price * quantity) STORED
);

-- Тестовые данные
INSERT INTO seller (seller_name)
VALUES
    ('ТТТ'),
    ('РТК');

INSERT INTO product (product_code, product_name)
VALUES
    (1, 'яблоки'),
    (2, 'Груши'),
    (3, 'Мандарины');

INSERT INTO invoice (
    invoice_number,
    invoice_date,
    seller_id
)
VALUES
    (1, '2024-01-10', 1),
    (2, '2024-01-15', 2);

INSERT INTO invoice_line (
    invoice_id,
    product_id,
    price,
    quantity
)
VALUES
    (1, 1, 10, 3),
    (1, 2, 11, 3),
    (2, 3, 20, 10),
    (2, 1, 15, 7);

 
-- Проверка исходных данных
SELECT
    i.invoice_number,
    i.invoice_date,
    s.seller_name,
    p.product_code,
    p.product_name,
    il.price,
    il.quantity,
    il.total_amount
FROM invoice_line il
JOIN invoice i ON i.invoice_id = il.invoice_id
JOIN seller s ON s.seller_id = i.seller_id
JOIN product p ON p.product_id = il.product_id
ORDER BY i.invoice_number, il.invoice_line_id;
```

# Что этим скриптом можно проверить

## **1. Повторение данных накладной**

В исходной таблице `Дата накладной` и `Продавец` повторяются для каждой позиции. В модели они находятся в `invoice` и не дублируются.

## **2. Повторение данных товара**

`Код товара = 1` встречается в двух накладных, но название хранится один раз в `product`.

При загрузке тестовых данных различие регистра в названии товара устранено для проверки целевой модели. Вопрос о правилах нормализации/стандартизации наименований является отдельным бизнес-/data-quality вопросом.

## **3. Цена — это свойство позиции, а не товара**

Для товара `1`:

```text
Накладная 1 → цена 10
Накладная 2 → цена 15
```

Поэтому гипотеза `product → price` на этих данных не подтверждается.

## **4. Стоимость**

Проверяем гипотезу:

```pgsql showLineNumbers
SELECT
    invoice_line_id,
    price,
    quantity,
    total_amount,
    price * quantity AS calculated_amount
FROM invoice_line;
```

Получим одинаковые значения `total_amount` и `calculated_amount`.

## **5. Связь Накладная ↔ Товар**

Модель допускает связь Накладная ↔ Товар как M:N, реализованную через `invoice_line`.

```text
Накладная 1 ──┬── Товар 1
              └── Товар 2

Накладная 2 ──┬── Товар 3
              └── Товар 1
```

То есть мы уже видим, что **один товар может встречаться в разных накладных**, а одна накладная содержит несколько товаров.

{% note warning "Внимание" %}

Вопрос о повторном появлении **одного товара в одной накладной** ещё не решён.

{% endnote %}

---

# invoice_id, product_id

{% note warning "Внимание" %}

Я пока не ставила ограничение `UNIQUE (invoice_id, product_id)`

{% endnote %}

Может ли один и тот же товар встречаться в одной накладной несколько раз?

Если нет, тогда: `UNIQUE (invoice_id, product_id)`

Если да, `invoice_line_id` нужен для идентификации конкретной строки, а `(invoice_id, product_id)` не является уникальным ключом.

Но есть ещё один бизнес-вопрос. Если товар встречается дважды в одной накладной, чем должны отличаться эти строки? Например, разной ценой, партией, условиями поставки или чем-то ещё.

---

# Допустимы ли нулевые и отрицательные значения цены и количества?

Сейчас БД разрешает:

```
price = -10
quantity = -5
```

Если отрицательные значения бизнесом не допускаются, то техническую гипотезу:

```pgsql
price NUMERIC(12, 2) NOT NULL CHECK (price >= 0),
quantity NUMERIC(12, 3) NOT NULL CHECK (quantity > 0),
```

---

# BIGSERIAL для тестовой модели

`BIGSERIAL PRIMARY KEY` приемлемо для тестовой модели.

Я **не усложняла** модель `GENERATED ALWAYS AS IDENTITY`, UUID и т.п. Это не влияет на проверяемую гипотезу 3НФ.

&nbsp;

