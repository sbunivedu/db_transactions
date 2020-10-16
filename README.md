# Database Transactions

This example demonstrates the use of SQLite transactions on the following database:
* accounts(account_no, balance)
* account_changes(change_no, account_no, flag, amount, changed_at)

The data set is in [script1.txt](script1.txt) and the code examples can be run on https://sqliteonline.com/

* transfer $1000 from account 100 to 200 and log the changes to the `account_changes` table in a single transaction
```sql
BEGIN TRANSACTION;

UPDATE accounts
SET balance = balance - 1000
WHERE account_no = 100;

UPDATE accounts
SET balance = balance + 1000
WHERE account_no = 200;

INSERT INTO account_changes(account_no,flag,amount,changed_at)
VALUES(100,'-',1000,datetime('now'));

INSERT INTO account_changes(account_no,flag,amount,changed_at)
VALUES(200,'+',1000,datetime('now'));

COMMIT;
```
The result should look as follows:
```sql
SELECT * FROM accounts;
+------------+---------+
| account_no | balance |
+------------+---------+
|        100 |   19100 |
|        200 |   11100 |
+------------+---------+

SELECT * FROM account_changes;
+-----------+------------+------+--------+---------------------+
| change_no | account_no | flag | amount | changed_at          |
+-----------+------------+------+--------+---------------------+
|         2 |        100 | -    |   1000 | 2020-04-12 10:52:24 |
|         3 |        200 | +    |   1000 | 2020-04-12 10:52:24 |
+-----------+------------+------+--------+---------------------+
```

* a transaction should abort if any of its operation fails or an explicit `ROLLBACK` statement is executed.
```sql
BEGIN TRANSACTION;

UPDATE accounts
SET balance = balance - 20000
WHERE account_no = 100;

INSERT INTO account_changes(account_no,flag,amount,changed_at)
VALUES(100,'-',20000,datetime('now'));

COMMIT;
```
The tables should stay unchanged as the result.

Transactions in SQLite are serializable.

The following MySQL example demonstrates the use of transactions with different isolation level on the following database:
* User(ID, Name)

The data set is in [script2.txt](script2.txt) and the code examples can be run on a MySQL database.

* "dirty read" could happen in a transaction with "read uncommitted" isolation level.
```sql
-- session 1
START TRANSACTION;
INSERT INTO User VALUES (2, 'bill');
                                    -- session 2
                                    SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
                                    START TRANSACTION;
                                    SELECT * FROM User;
                                     +----+------+
                                     |  2 | bill |
                                     +----+------+

ROLLBACK;
                                    SELECT * FROM USER;
                                    Empty set (0.00 sec)
```

* "non-repeatable read" could happen in a transaction with "read committed" isolation level.
```sql
-- session 1
SELECT * FROM User WHERE ID=2;
+----+------+
|  2 | bill |
+----+------+
                                    -- session 2
                                    SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
                                    START TRANSACTION;
                                    SELECT * FROM User WHERE ID=2;
                                    +----+------+
                                    |  2 | bill |
                                    +----+------+
UPDATE User SET Name='bob' WHERE ID=2;
                                    SELECT * FROM User WHERE ID=2;
                                    +----+------+
                                    |  2 | bob  |
                                    +----+------+

```

* "repeatable read" isolation level indeed guaranties repeatable read.
```sql
-- session 1
SELECT * FROM User WHERE ID=2;
+----+------+
|  2 | bill |
+----+------+
                                    -- session 2
                                    SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
                                    START TRANSACTION;
                                    SELECT * FROM User WHERE ID>=2;
                                    +----+------+
                                    |  2 | bill |
                                    +----+------+
UPDATE User SET Name='bob' WHERE ID='2';
INSERT INTO User VALUES (3, 'jack');
                                    SELECT * FROM User WHERE ID>=2;
                                    +----+------+
                                    |  2 | bill |
                                    +----+------+
                                    SELECT * FROM User FOR UPDATE;
                                    +----+------+
                                    | ID | Name |
                                    +----+------+
                                    |  2 | bob  |
                                    |  3 | jack |
                                    +----+------+
```

In MySQL, "repeatable read" isolation level seems to behave the same way as "serializable". The last statement `SELECT * FROM User FOR UPDATE;` show a surprise resulting from a "locking read", which is unrelated to "isolation level".

References
* https://www.sqlitetutorial.net/sqlite-transaction
* https://www.sqlite.org/isolation.html
* https://www.mysqltutorial.org/mysql-transaction.aspx
* http://www.herongyang.com/MySQL/Transaction-Isolation-Level-Read-Uncommitted.html
* http://www.herongyang.com/MySQL/Transaction-Isolation-Level-Read-Committed.html
* http://www.herongyang.com/MySQL/Transaction-Isolation-Level-Repeatable-Read.html
* Phantom reads are impossible on MySQL with repeatable read isolation level because of the locking mechanism used in innoDB as described in the following documents:
https://blogs.oracle.com/mysqlinnodb/entry/repeatable_read_isolation_level_in
