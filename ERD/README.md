- 회원 : member
- 주문 : orders
- 상품 : item
- 주문 상품 : ordered_item

## DDL

```oracle
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

## ERD

<img width="800" alt="ERD" src="https://github.com/user-attachments/assets/6da52c22-3d0e-4864-a5d6-e9f13970f0e6">
