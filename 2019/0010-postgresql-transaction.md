# Transaction ใน PostgreSQL

Transaction เป็น feature หนึ่งใน Database หลาย ๆ ตัว, PostgreSQL ก็เป็นหนึ่งใน Database ที่มี Transaction

## Transaction คืออะไร

ถ้าให้อธิบายความหมายของ Transaction แบบย่อ ๆ ก็คือการรวม operations ที่เราจะทำกับ database ให้เหลือแค่ operation เดียว

หมายความว่าทุก operations ต้องสำเร็จถึงจะสำเร็จ ถ้ามี operation ใดไม่สำเร็จ ทุก operation จะไม่สำเร็จ (All or Nothing)

ตัวอย่างการใช้งาน Transaction

```sql
begin;
update accounts set balance = balance + 100 where id = 1;
update accounts set balance = balance - 100 where id = 2;
commit;
```

จากตัวอย่างข้างบน เรา update ตาราง accounts โดยที่ให้

- account id 1 เพิ่ม balance 100
- account id 2 ลด balance 100

ถ้ามี error เกิดขึ้นขณะที่กำลังลด balance ของ account id 2 เราจะไม่สามารถ commit ได้ ทำให้ balance ของ account id 1 ไม่ถูกเพิ่ม

## ปัญหาเมื่อรัน Transaction พร้อมกันหลาย Transactions

เมื่อ Transaction ถูกรันพร้อมกัน มีโอกาส 3 อย่างที่จะเกิดขึ้น

1. Dirty Read

    Transaction เห็น data ของอีก transaction ที่ยังไม่ได้ commit

    1. Data ก่อนรัน transaction

        ```sql
        select * from accounts;
        ```

        | id | balance |
        | --- | --- |
        | 1 | 100 |
        | 2 | 200 |

    1. **Tx#1**

        ```sql
        begin;
        ```

    1. **Tx#2**

        ```sql
        begin;
        ```

    1. **Tx#1**

        ```sql
        update accounts set balance = 300 where id = 1;
        ```

    1. **Tx#2**

        ```sql
        select * from accounts where id = 1;
        ```

        | id | balance |
        | --- | --- |
        | 1 | 300 |

    1. **Tx#1**

        ```sql
        rollback;
        ```

    1. **Tx#2**

        ```sql
        select * from accounts where id = 1;
        ```

        | id | balance |
        | --- | --- |
        | 1 | 100 |

1. Nonrepeatable read

    Transaction อ่าน data 2 รอบ ได้คนละค่า (รอบแรกอ่านก่อนอีก transaction commit,​ รอบที่ 2 อ่านหลัง commit)

    1. Data ก่อนรัน transaction

        ```sql
        select * from accounts;
        ```

        | id | balance |
        | --- | --- |
        | 1 | 100 |
        | 2 | 200 |

    1. **Tx#1**

        ```sql
        begin;
        ```

    1. **Tx#2**

        ```sql
        begin;
        ```

    1. **Tx#1**

        ```sql
        update accounts set balance = 300 where id = 1;
        ```

    1. **Tx#2**

        ```sql
        select * from accounts where id = 1;
        ```

        | id | balance |
        | --- | --- |
        | 1 | 100 |

    1. **Tx#1**

        ```sql
        commit;
        ```

    1. **Tx#2**

        ```sql
        select * from accounts where id = 1;
        ```

        | id | balance |
        | --- | --- |
        | 1 | 300 |

1. Phantom Read

    Transaction อ่าน data 2 รอบ ได้จำนวน records ไม่เท่าเดิม (อีก transaction insert data แล้ว commit)

    1. Data ก่อนรัน transaction

        ```sql
        select * from accounts;
        ```

        | id | balance |
        | --- | --- |
        | 1 | 100 |
        | 2 | 200 |

    1. **Tx#1**

        ```sql
        begin;
        ```

    1. **Tx#2**

        ```sql
        begin;
        ```

    1. **Tx#2**

        ```sql
        select * from accounts where id = 1;
        ```

        | id | balance |
        | --- | --- |
        | 1 | 100 |
        | 2 | 200 |

    1. **Tx#1**

        ```sql
        insert into accounts (id, balance) values (3, 500);
        ```

    1. **Tx#1**

        ```sql
        commit;
        ```

    1. **Tx#2**

        ```sql
        select * from accounts where id = 1;
        ```

        | id | balance |
        | --- | --- |
        | 1 | 100 |
        | 2 | 200 |
        | 3 | 500 |

## Transaction Isolation

Transaction Isolation คือการกำหนดว่าจะให้ Transaction รัน isolate กับ Transaction อื่น ยังไง

> ตารางข้างล่างนี้่เป็น isolation ของ PostgreSQL ซึ่งต่างกับ SQL Standard

| Level | Dirty Read | Nonrepeatable Read | Phantom Read |
| --- | --- | --- | --- | --- |
| Read uncommitted | ไม่เกิดขึ้น | มีโอกาสเกิด | มีโอกาสเกิด |
| Read committed | ไม่เกิดขึ้น | มีโอกาสเกิด | มีโอกาสเกิด |
| Repeatable read | ไม่เกิดขึ้น | ไม่เกิดขึ้น | ไม่เกิดขึ้น |
| Serializable | ไม่เกิดขึ้น | ไม่เกิดขึ้น | ไม่เกิดขึ้น | ไม่เกิดขึ้น |

ถ้าดูจากตารางจะเห็นว่าเราสามารถเรียกใช้ isolation level ได้ 4 levels แต่ PostgreSQL implement แค่ 3 levels (Read uncommitted กับ Read committed ทำงานเหมือนกัน)

โดยที่ค่า default ของ PostgreSQL คือ Read committed

### ความแตกต่างระหว่าง Repeatable read กับ Serializable

ถ้าเรา begin transaction ด้วย isolation `repeatable read` หรือ `serializable`
เมื่อ 2 Transactions พยายามที่จะเขียน ใน record เดียวกัน PostgreSQL จะ lock ไม่ให้อีก Transaction เขียน จนกว่า transaction แรกจะ commit หรือ rollback

แต่ถ้าไม่ได้เขียน record เดียวกัน แต่อ่าน record เดียวกันหล่ะ ?

ลองดูตัวอย่างนี้

1. มีตาราง 3 tables

    - rooms

        | id | name |
        | --- | --- |
        | 1 | room 1 |
        | 2 | room 2 |

    - staffs

        | id | name |
        | --- | --- |
        | 1 | staff 1 |
        | 2 | staff 2 |

    - events (ไม่มี primary key)

        | room_id | staff_id |
        | --- | --- |

    สิ่งที่จะทำก็คือ เราจะหาห้องที่ยังไม่มี event และ staff ที่ว่างงานที่ไม่ได้จัด event มาจัด event

1. หาห้องว่าง

    ```sql
    select id
    from rooms
    where id not in (select room_id from events)
    limit 1
    -- 1
    ```

1. หา staff ว่าง

    ```sql
    select id
    from staffs
    where id not in (select staff_id from events)
    limit 1
    -- 1
    ```

1. สร้าง event

    ```sql
    insert into events
        (room_id, staff_id)
    values
        ({ห้องที่ว่าง}, {staff ที่ว่าง})
    ```

ถ้าเรารันใน `repeatable read` transaction พร้อมกัน 5 transactions เราจะได้ผลลัพท์เป็น

| room_id | staff_id |
| --- | --- |
| 1 | 1 |
| 1 | 1 |
| 1 | 1 |
| 1 | 1 |
| 1 | 1 |

แต่ถ้าเราเปลี่ยน isolation เป็น `serializable` เราจะได้

| room_id | staff_id |
| --- | --- |
| 1 | 1 |

จะมีแค่ 1 transaction ที่จะสามารถ commit ได้ ส่วน transaction ที่เหลือจะได้ error

`could not serialize access due to read/write dependencies among transactions`

และเราต้อง retry transaction เพื่อหาห้องและ staff ที่ว่างคนใหม่
