# Кейс-задача №5 — рекомендации и стратегии устранения трудностей

> Основано на анализе из кейс‑задачи №4 (Система на Prolog) и решении кейс‑задачи №3 (DDL MySQL для базы данных «Туризм»)

---

## 1) Краткое резюме

В решении кейс‑задачи №3 мы спроектировали 5 таблиц: `Services`, `Clients`, `Operators`, `Tours` (справочники) и `Orders` (переменная информация). Модель рабочая, но в реальных сценариях туризма возникнут сложности:

1. **Состав заказа**: один заказ может включать *несколько* услуг и туров → текущая колонка `service_id` в `Orders` ограничивает 1:1.
2. **Нормализация направлений**: поле `destination` в `Tours` — текст без справочника.
3. **Цены и валюта**: нет привязки к валюте, нет «снимка цены» на момент заказа.
4. **Целостность и обязательные связи**: внешние ключи местами опциональны, отсутствуют правила удаления/обновления.
5. **Жизненный цикл заказа**: отсутствуют статусы, история оплат.
6. **Индексы и метаданные**: нет индексов по FK и полям поиска; нет `created_at/updated_at`.

Ниже — стратегии устранения, улучшенный DDL и план миграции.

---

## 2) Стратегии устранения трудностей

### 2.1 Состав заказа (многие‑ко‑многим)

**Проблема**: `Orders.service_id` фиксирует только одну услугу на заказ.
**Стратегия**: ввести таблицу **`OrderItems`** (позиции заказа) с полем `item_type` (`tour`/`service`) и ссылками на `Tours` или `Services`. Это масштабируемо, покрывает набор из нескольких туров/услуг.

### 2.2 Нормализация направлений

**Проблема**: `Tours.destination` как свободный текст ведёт к дублированию значений.
**Стратегия**: справочник **`Destinations`** и ссылка `Tours.destination_id`.

### 2.3 Цены, валюта и «снимок цены»

**Проблема**: есть `Services.price` и `Orders.total_price`, но нет валюты; итог может не совпадать с текущими ценами.
**Стратегия**: хранить **цены в позициях заказа** (`OrderItems.unit_price`) и валюту в `Orders.currency_code`. Итог `Orders.total_amount` вычислять суммой позиций (триггерами). Базовые цены в справочниках — как рекомендуемые.

### 2.4 Жизненный цикл и оплаты

**Стратегия**: добавить статусы `Orders.status` (`draft/booked/paid/cancelled`) и таблицу **`Payments`** для учёта оплат.

### 2.5 Целостность и правила удаления

**Стратегия**: делать FK **NOT NULL** там, где связь обязательна; выставить `ON DELETE RESTRICT` (или `CASCADE` по бизнес‑логике), `ON UPDATE CASCADE`.

### 2.6 Индексация и метаданные

**Стратегия**: индексы по внешним ключам и полям поиска; добавить `created_at`, `updated_at` в таблицы с изменяемыми данными.

---

## 3) Улучшенная схема (MySQL 8.0+)

> Вариант **V2 (производственный)**. Меняет модель заказов на «заказ + позиции». Справочник направлений вынесен отдельно. Ниже полный DDL (создание новой схемы для чистоты примера).

```sql
-- === База данных
DROP DATABASE IF EXISTS TourismV2;
CREATE DATABASE TourismV2 CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci;
USE TourismV2;

-- === Справочник направлений
CREATE TABLE Destinations (
    destination_id INT PRIMARY KEY AUTO_INCREMENT,
    name          VARCHAR(255) NOT NULL,
    country       VARCHAR(100) NOT NULL,
    CONSTRAINT uq_destination UNIQUE (name, country)
);

-- === Клиенты
CREATE TABLE Clients (
    client_id   INT PRIMARY KEY AUTO_INCREMENT,
    first_name  VARCHAR(100) NOT NULL,
    last_name   VARCHAR(100) NOT NULL,
    email       VARCHAR(255) NOT NULL UNIQUE,
    phone       VARCHAR(20),
    created_at  TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at  TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

-- === Туроператоры
CREATE TABLE Operators (
    operator_id    INT PRIMARY KEY AUTO_INCREMENT,
    operator_name  VARCHAR(255) NOT NULL,
    contact_number VARCHAR(20),
    email          VARCHAR(255) NOT NULL UNIQUE
);

-- === Туры
CREATE TABLE Tours (
    tour_id        INT PRIMARY KEY AUTO_INCREMENT,
    tour_name      VARCHAR(255) NOT NULL,
    destination_id INT NOT NULL,
    duration_days  INT NOT NULL,
    operator_id    INT NOT NULL,
    base_price     DECIMAL(12,2) NOT NULL DEFAULT 0.00,
    currency_code  CHAR(3) NOT NULL DEFAULT 'USD',
    is_active      TINYINT(1) NOT NULL DEFAULT 1,
    CONSTRAINT fk_tours_destination FOREIGN KEY (destination_id) REFERENCES Destinations(destination_id)
        ON UPDATE CASCADE ON DELETE RESTRICT,
    CONSTRAINT fk_tours_operator FOREIGN KEY (operator_id) REFERENCES Operators(operator_id)
        ON UPDATE CASCADE ON DELETE RESTRICT,
    CONSTRAINT chk_duration CHECK (duration_days >= 1),
    CONSTRAINT chk_base_price CHECK (base_price >= 0)
);
CREATE INDEX idx_tours_dest ON Tours(destination_id);
CREATE INDEX idx_tours_operator ON Tours(operator_id);

-- === Услуги
CREATE TABLE Services (
    service_id   INT PRIMARY KEY AUTO_INCREMENT,
    service_name VARCHAR(255) NOT NULL,
    description  TEXT,
    base_price   DECIMAL(12,2) NOT NULL DEFAULT 0.00,
    is_active    TINYINT(1) NOT NULL DEFAULT 1,
    CONSTRAINT chk_service_price CHECK (base_price >= 0)
);

-- === Заказы (шапка)
CREATE TABLE Orders (
    order_id      INT PRIMARY KEY AUTO_INCREMENT,
    client_id     INT NOT NULL,
    order_date    DATETIME NOT NULL,
    status        ENUM('draft','booked','paid','cancelled') NOT NULL DEFAULT 'draft',
    currency_code CHAR(3) NOT NULL DEFAULT 'USD',
    total_amount  DECIMAL(12,2) NOT NULL DEFAULT 0.00,
    comment       TEXT,
    created_at    TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at    TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    CONSTRAINT fk_orders_client FOREIGN KEY (client_id) REFERENCES Clients(client_id)
        ON UPDATE CASCADE ON DELETE RESTRICT
);
CREATE INDEX idx_orders_client ON Orders(client_id);
CREATE INDEX idx_orders_date   ON Orders(order_date);
CREATE INDEX idx_orders_status ON Orders(status);

-- === Позиции заказа
CREATE TABLE OrderItems (
    order_item_id  INT PRIMARY KEY AUTO_INCREMENT,
    order_id       INT NOT NULL,
    item_type      ENUM('tour','service') NOT NULL,
    tour_id        INT NULL,
    service_id     INT NULL,
    quantity       INT NOT NULL DEFAULT 1,
    unit_price     DECIMAL(12,2) NOT NULL,
    discount_amt   DECIMAL(12,2) NOT NULL DEFAULT 0.00,
    line_total     DECIMAL(12,2) AS (GREATEST(0, quantity * unit_price - discount_amt)) STORED,
    CONSTRAINT fk_item_order   FOREIGN KEY (order_id) REFERENCES Orders(order_id)
        ON UPDATE CASCADE ON DELETE CASCADE,
    CONSTRAINT fk_item_tour    FOREIGN KEY (tour_id)  REFERENCES Tours(tour_id)
        ON UPDATE CASCADE ON DELETE RESTRICT,
    CONSTRAINT fk_item_service FOREIGN KEY (service_id) REFERENCES Services(service_id)
        ON UPDATE CASCADE ON DELETE RESTRICT,
    CONSTRAINT chk_qty CHECK (quantity >= 1),
    CONSTRAINT chk_unit_price CHECK (unit_price >= 0),
    CONSTRAINT chk_discount CHECK (discount_amt >= 0),
    -- Гарантируем, что заполнен либо tour_id, либо service_id в соответствии с item_type
    CONSTRAINT chk_item_target CHECK (
        (item_type = 'tour' AND tour_id IS NOT NULL AND service_id IS NULL) OR
        (item_type = 'service' AND service_id IS NOT NULL AND tour_id IS NULL)
    )
);
CREATE INDEX idx_items_order   ON OrderItems(order_id);
CREATE INDEX idx_items_tour    ON OrderItems(tour_id);
CREATE INDEX idx_items_service ON OrderItems(service_id);

-- === Оплаты (по желанию)
CREATE TABLE Payments (
    payment_id   INT PRIMARY KEY AUTO_INCREMENT,
    order_id     INT NOT NULL,
    paid_at      DATETIME NOT NULL,
    amount       DECIMAL(12,2) NOT NULL,
    method       ENUM('card','cash','transfer','other') NOT NULL,
    status       ENUM('pending','confirmed','failed','refunded') NOT NULL DEFAULT 'pending',
    CONSTRAINT fk_pay_order FOREIGN KEY (order_id) REFERENCES Orders(order_id)
        ON UPDATE CASCADE ON DELETE CASCADE,
    CONSTRAINT chk_payment_amt CHECK (amount >= 0)
);
CREATE INDEX idx_payments_order ON Payments(order_id);

-- === Триггеры на пересчёт тотала заказа
DELIMITER $$
CREATE TRIGGER trg_items_ai AFTER INSERT ON OrderItems FOR EACH ROW
BEGIN
  UPDATE Orders o
  SET o.total_amount = (SELECT IFNULL(SUM(line_total),0) FROM OrderItems WHERE order_id = NEW.order_id)
  WHERE o.order_id = NEW.order_id;
END $$
CREATE TRIGGER trg_items_au AFTER UPDATE ON OrderItems FOR EACH ROW
BEGIN
  UPDATE Orders o
  SET o.total_amount = (SELECT IFNULL(SUM(line_total),0) FROM OrderItems WHERE order_id = NEW.order_id)
  WHERE o.order_id = NEW.order_id;
END $$
CREATE TRIGGER trg_items_ad AFTER DELETE ON OrderItems FOR EACH ROW
BEGIN
  UPDATE Orders o
  SET o.total_amount = (SELECT IFNULL(SUM(line_total),0) FROM OrderItems WHERE order_id = OLD.order_id)
  WHERE o.order_id = OLD.order_id;
END $$
DELIMITER ;
```

> Примечание: `CHECK`‑ограничения применяются в MySQL 8.0.16+. Для более старых версий используйте триггеры/прикладную валидацию.

---

## 4) План миграции с текущей схемы (из кейс‑задачи №3)

**Шаг 0.** Бэкап базы `Tourism`.

**Шаг 1.** Добавьте в `Tours` колонны `base_price DECIMAL(12,2) NOT NULL DEFAULT 0.00`, `currency_code CHAR(3) NOT NULL DEFAULT 'USD'`.

**Шаг 2.** Создайте `Destinations`, перенесите уникальные `destination` из `Tours`:

```sql
CREATE TABLE Destinations (... как выше ...);
INSERT INTO Destinations (name, country)
SELECT DISTINCT destination AS name, '—' AS country FROM Tours;  -- при необходимости заполнить страну позднее
ALTER TABLE Tours ADD COLUMN destination_id INT;
UPDATE Tours t JOIN Destinations d ON d.name = t.destination
SET t.destination_id = d.destination_id;
ALTER TABLE Tours
  ADD CONSTRAINT fk_tours_destination FOREIGN KEY (destination_id) REFERENCES Destinations(destination_id),
  DROP COLUMN destination;  -- после проверки
```

**Шаг 3.** Создайте `OrderItems` и перенесите данные из `Orders`:

```sql
-- Туры как позиции (если в старой схеме тур был в Orders)
INSERT INTO OrderItems (order_id, item_type, tour_id, service_id, quantity, unit_price, discount_amt)
SELECT o.order_id, 'tour', o.tour_id, NULL, 1, t.base_price, 0
FROM Orders o JOIN Tours t ON t.tour_id = o.tour_id
WHERE o.tour_id IS NOT NULL;

-- Услуги как позиции
INSERT INTO OrderItems (order_id, item_type, tour_id, service_id, quantity, unit_price, discount_amt)
SELECT o.order_id, 'service', NULL, o.service_id, 1, s.base_price, 0
FROM Orders o JOIN Services s ON s.service_id = o.service_id
WHERE o.service_id IS NOT NULL;

-- Пересчёт тотала по триггеру (или явно):
UPDATE Orders o
JOIN (SELECT order_id, SUM(line_total) AS sum_total FROM OrderItems GROUP BY order_id) x
  ON x.order_id = o.order_id
SET o.total_amount = x.sum_total;
```

**Шаг 4.** Упростите `Orders`: удалите `service_id`, `tour_id`, `total_price` (устаревает), добавьте `status`, `currency_code`.

```sql
ALTER TABLE Orders
  ADD COLUMN status        ENUM('draft','booked','paid','cancelled') NOT NULL DEFAULT 'draft',
  ADD COLUMN currency_code CHAR(3) NOT NULL DEFAULT 'USD',
  ADD COLUMN total_amount  DECIMAL(12,2) NOT NULL DEFAULT 0.00;
ALTER TABLE Orders DROP FOREIGN KEY Orders_ibfk_1; -- имя FK уточните по SHOW CREATE TABLE
ALTER TABLE Orders DROP FOREIGN KEY Orders_ibfk_2;
ALTER TABLE Orders DROP FOREIGN KEY Orders_ibfk_3;
ALTER TABLE Orders DROP COLUMN service_id, DROP COLUMN tour_id, DROP COLUMN total_price;
```

**Шаг 5.** Добавьте индексы и правила `ON DELETE/UPDATE` (см. DDL V2). Прогоните тесты из раздела 5.

---

## 5) Проверки качества данных и пример запросов

**Проверка сиротских ссылок:**

```sql
SELECT oi.order_item_id FROM OrderItems oi LEFT JOIN Orders o ON o.order_id = oi.order_id WHERE o.order_id IS NULL;
SELECT t.tour_id        FROM Tours t        LEFT JOIN Operators op ON op.operator_id = t.operator_id WHERE op.operator_id IS NULL;
```

**Проверка положительности и длительности:**

```sql
SELECT * FROM Tours WHERE duration_days < 1;
SELECT * FROM OrderItems WHERE unit_price < 0 OR discount_amt < 0 OR quantity < 1;
```

**Пример вставки и выборки:**

```sql
-- Клиент, оператор, направление, тур
INSERT INTO Clients(first_name,last_name,email) VALUES ('Ivan','Petrov','ivan@example.com');
INSERT INTO Operators(operator_name,email) VALUES ('Sunny Travel','ops@sunny.travel');
INSERT INTO Destinations(name,country) VALUES ('Bali','Indonesia');
INSERT INTO Tours(tour_name,destination_id,duration_days,operator_id,base_price,currency_code)
VALUES('Bali Relax', 1, 7, 1, 1200.00, 'USD');

-- Заказ + позиции
INSERT INTO Orders(client_id, order_date, status, currency_code)
VALUES (1, NOW(), 'booked', 'USD');
INSERT INTO OrderItems(order_id,item_type,tour_id,quantity,unit_price,discount_amt)
VALUES (1,'tour',1,1,1200.00,100.00);
INSERT INTO Services(service_name, base_price) VALUES ('Airport transfer', 30.00);
INSERT INTO OrderItems(order_id,item_type,service_id,quantity,unit_price)
VALUES (1,'service',1,2,30.00);

-- Итог по заказу (пересчитан триггером)
SELECT o.order_id, o.total_amount,
       SUM(oi.line_total) AS check_sum
FROM Orders o JOIN OrderItems oi ON oi.order_id = o.order_id
WHERE o.order_id = 1
GROUP BY 1,2;
```

---

## 6) Связь с кейс‑задачей №4: план обучения команды

Экспертная система обучения (кейс №4) подсказала фокусы обучения по ролям. Для устранения выявленных трудностей:

* **Разработчик/DB‑инженер**: SQL‑практика и качество кода → `m_sql_practice`, `m_code_quality`.
* **Аналитик данных**: основы SQL и визуализация для проверок данных → `m_sql_intro`, `m_dataviz`.
* **QA/тестировщик**: фундамент тестирования и тесты данных → `m_testing_basics` (добавить тест‑кейсы на целостность БД).
* **PM**: управление бэклогом задач миграции → `m_agile_scrum`, `m_jira`.

**Как применить:**

1. индексы, NOT NULL, статусы заказов; чек‑листы QA.
2. `OrderItems`, миграция данных, триггеры на тотал.
3. `Destinations`, отчёты и дашборды контроля качества.
