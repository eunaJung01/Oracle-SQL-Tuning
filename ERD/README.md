# ERD & DDL, Dummy Data Insertion

### Contents

- [ERD](#ERD)
- [DDL](#DDL)
- [DDL with NOT NULL Constraints](#DDL-with-NOT-NULL-constraints)
- [PL/SQL for Dummy Data Insertion](#plsql-for-dummy-data-insertion)

<br/>
<br/>

## ERD

- 회원 : `MEMBER`
- 주문 : `ORDERS`
- 상품 : `ITEM`
- 주문 상품 : `ORDERED_ITEM`

<br/>

<p align="center"><img width="800" alt="ERD" src="https://github.com/user-attachments/assets/6da52c22-3d0e-4864-a5d6-e9f13970f0e6">

<br/>
<br/>
<br/>

## DDL

```sql
CREATE TABLE member
(
    member_id NUMBER
        GENERATED ALWAYS AS IDENTITY
            (START WITH 1 INCREMENT BY 1)
        PRIMARY KEY,

    name      VARCHAR2(50),
    city      VARCHAR2(50),
    street    VARCHAR2(50),
    zipcode   NUMBER(5)
);

CREATE TABLE orders
(
    order_id   NUMBER
        GENERATED ALWAYS AS IDENTITY
            (START WITH 1 INCREMENT BY 1)
        PRIMARY KEY,

    member_id  NUMBER,
    order_date DATE,
    status     VARCHAR2(50),

    CONSTRAINT order_fk
        FOREIGN KEY (member_id)
            REFERENCES member (member_id)
);

CREATE TABLE item
(
    item_id        NUMBER
        GENERATED ALWAYS AS IDENTITY
            (START WITH 1 INCREMENT BY 1)
        PRIMARY KEY,

    name           VARCHAR2(50),
    price          NUMBER(5),
    stock_quantity NUMBER(5)
);

CREATE TABLE ordered_item
(
    ordered_item_id NUMBER
        GENERATED ALWAYS AS IDENTITY
            (START WITH 1 INCREMENT BY 1)
        PRIMARY KEY,

    order_id        NUMBER,
    item_id         NUMBER,
    price           NUMBER(5),
    quantity        NUMBER(5),

    CONSTRAINT ordered_items_fk
        FOREIGN KEY (order_id)
            REFERENCES orders (order_id),

    CONSTRAINT ordered_items_fk2
        FOREIGN KEY (item_id)
            REFERENCES item (item_id)
);
```

<br/>
<br/>

## DDL with NOT NULL Constraints

```sql
CREATE TABLE member
(
    member_id NUMBER
        GENERATED ALWAYS AS IDENTITY
            (START WITH 1 INCREMENT BY 1)
        PRIMARY KEY,

    name      VARCHAR2(50) NOT NULL,
    city      VARCHAR2(50) NOT NULL,
    street    VARCHAR2(50) NOT NULL,
    zipcode   NUMBER(5)    NOT NULL
);

CREATE TABLE orders
(
    order_id   NUMBER
        GENERATED ALWAYS AS IDENTITY
            (START WITH 1 INCREMENT BY 1)
        PRIMARY KEY,

    member_id  NUMBER NOT NULL,
    order_date DATE   NOT NULL,
    status     VARCHAR2(50) NOT NULL,

    CONSTRAINT order_fk
        FOREIGN KEY (member_id)
            REFERENCES member (member_id)
);

CREATE TABLE item
(
    item_id        NUMBER
        GENERATED ALWAYS AS IDENTITY
            (START WITH 1 INCREMENT BY 1)
        PRIMARY KEY,

    name           VARCHAR2(50) NOT NULL,
    price          NUMBER(5)    NOT NULL,
    stock_quantity NUMBER(5)    NOT NULL
);

CREATE TABLE ordered_item
(
    ordered_item_id NUMBER
        GENERATED ALWAYS AS IDENTITY
            (START WITH 1 INCREMENT BY 1)
        PRIMARY KEY,

    order_id        NUMBER NOT NULL,
    item_id         NUMBER NOT NULL,
    price           NUMBER(5) NOT NULL,
    quantity        NUMBER(5) NOT NULL,

    CONSTRAINT ordered_items_fk
        FOREIGN KEY (order_id)
            REFERENCES orders (order_id),

    CONSTRAINT ordered_items_fk2
        FOREIGN KEY (item_id)
            REFERENCES item (item_id)
);
```

<br/>
<br/>

## PL/SQL for Dummy Data Insertion

```SQLPL
BEGIN
    FOR i IN 1..100000
        LOOP
            INSERT INTO member (name, city, street, zipcode)
            VALUES ('Member ' || i,
                    'City ' || MOD(i, 100),
                    'Street ' || MOD(i, 1000),
                    MOD(i, 90000) + 10000);
        END LOOP;
    COMMIT;

    FOR i IN 1..100000
        LOOP
            INSERT INTO orders (member_id, order_date, status)
            VALUES (MOD(i, 100000) + 1,
                    SYSDATE - MOD(i, 365),
                    CASE MOD(i, 3)
                        WHEN 0 THEN 'PENDING'
                        WHEN 1 THEN 'SHIPPED'
                        ELSE 'DELIVERED'
                        END);
        END LOOP;

    FOR i IN 1..1000
        LOOP
            INSERT INTO item (name, price, stock_quantity)
            VALUES ('Item ' || i,
                    MOD(i, 90) + 10,
                    MOD(i, 1000) + 1);
        END LOOP;

    FOR i IN 1..100000
        LOOP
            INSERT INTO ordered_item (order_id, item_id, price, quantity)
            VALUES (MOD(i, 100000) + 1,
                    MOD(i, 1000) + 1,
                    MOD(i, 90) + 10,
                    MOD(i, 10) + 1);
        END LOOP;

    COMMIT;
END;
/
```
