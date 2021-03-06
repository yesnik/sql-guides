# SQLSTATE[HY000]: General error: 1205 Lock wait timeout exceeded; try restarting transaction

This error occures when we try to access data that has been locked by another process.
MySQL waits some time for the lock to be removed and then throws this error.

Something is blocking the execution of our query. 
Most likely another query updating, inserting or deleting from one of the tables in our query. 

## Debug tools

### Look at current processes

```sql
SHOW FULL processlist
```

One of these transactions could be blocking. Pay special attention to:

- processes with big `Time`
- processes in `Sleep` state

### Show current transactions

```sql
SHOW ENGINE INNODB STATUS;
```

### Get value of innodb_lock_wait_timeout

```sql
SHOW VARIABLES LIKE 'innodb_lock_wait_timeout';
```

Default value is 50.

## Reproduce lock wait timeout exceeded error

### Example 1. Not finished transaction

*Note:* In this example we are using *10.1.33-MariaDB MariaDB Server*, innodb version: *5.6.39*.

1. In your test MySQL database create table `test` and populate it with several records:

```sql
CREATE TABLE `test` (
	`id` INT(11) NOT NULL,
	PRIMARY KEY (`id`)
)
ENGINE=InnoDB;

INSERT INTO test (`id`) values (1), (2), (3);
```

2. Open *first* terminal window and connect to database. Open transaction and execute query on deleting records. 
*Don't* close transaction yet.

```sql
BEGIN;
DELETE FROM test WHERE id >= 2;
```

This command locked rows `2, 3` of table `test`.

3. Open *second* terminal window and connect to database. Try to select rows:

```sql
> select * from test;
+----+
| id |
+----+
|  1 |
|  2 |
|  3 |
+----+
3 rows in set (0.00 sec)
```

As we can see, `SELECT` on locked table works.

4. Try to INSERT, UPDATE, DELETE record. Commands will hang for 50 sec and then throws error:

```sql
insert into test (id) values (4);
-- ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction

update test set id = 11 where id = 1;
-- ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction

delete from test where id = 2;
-- ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction
```

*Note:* But we **can delete** not locked row:

```sql
delete from test where id = 1;
-- Query OK, 1 row affected (0.00 sec)
```

5. Let's see process list. In our second terminal execute query:

```sql
show full processlist;
+-------+------+-----------+-------+---------+------+-------+-----------------------+----------+
| Id    | User | Host      | db    | Command | Time | State | Info                  | Progress |
+-------+------+-----------+-------+---------+------+-------+-----------------------+----------+
| 46979 | root | localhost | sales | Sleep   |  950 |       | NULL                  |    0.000 |
| 47160 | root | localhost | sales | Query   |    0 | init  | show full processlist |    0.000 |
+-------+------+-----------+-------+---------+------+-------+-----------------------+----------+
2 rows in set (0.00 sec)
```

As we can see process `46979` is in `Sleep` and it lasts 950 seconds. 
We know that it's a process of MySQL connection in our *first* terminal.

Execute query to kill blocking process:

```sql
kill 46979;
```

6. Records in table were not touched by killed process:

```sql
select * from test;
+----+
| id |
+----+
|  2 |
|  3 |
+----+
```

### Example 2. Update of many records in big table

1. Create table:

```sql
CREATE TABLE `test` (
	`id` INT(11) NOT NULL AUTO_INCREMENT,
	`amount` INT(11) NULL DEFAULT '0',
	`amount2` INT(11) NULL DEFAULT '0',
	PRIMARY KEY (`id`)
)
ENGINE=InnoDB;
```

2. Populate table with data. Create procedure that generates 1 million of rows (~ 4.5 min on my PC):

```sql
DROP PROCEDURE IF EXISTS table_test_populate;

DELIMITER $$
CREATE PROCEDURE table_test_populate()
BEGIN
  DECLARE i INT DEFAULT 0;
  WHILE i < 1000000 DO
    INSERT INTO `test` (`amount`,`amount2`) VALUES (
      FLOOR(RAND()*100),
      FLOOR(RAND()*100)
    );
    SET i = i + 1;
  END WHILE;
END$$
DELIMITER ;
```

Generate about 10 million of rows in test table, the more the better:

```sql
CALL table_test_populate();
```

3. Open *first* terminal and connect to database. Update almost all rows - with `id >= 1000`:

```sql
UPDATE test SET amount = amount + 1 WHERE id >= 1000;
```

4. While previous query is running, open *second* terminal and run queries:

**if id < 1000 - NOT blocked**

*Update*

```sql
UPDATE test SET amount2 = 500 WHERE id = 10;
-- Query OK, 1 row affected (0.58 sec)
-- Rows matched: 1  Changed: 1  Warnings: 0
```

*Delete*

```sql
DELETE FROM test WHERE id = 500;
-- Query OK, 1 row affected (0.08 sec)
```

**if id >= 1000 - BLOCKED**

*Update*

```sql
UPDATE test SET amount2 = 500 WHERE id = 1005;
-- Query OK, 1 row affected (16.57 sec)
-- Rows matched: 1  Changed: 1  Warnings: 0
```

*Delete*

```sql
DELETE FROM test WHERE id = 1005;
-- Query OK, 1 row affected (17.95 sec)
```

**INSERT - NOT blocked**

```sql
INSERT INTO test (amount, amount2) VALUES (11, 22);
-- Query OK, 1 row affected (0.01 sec)
```

**SELECT - NOT blocked**

```sql
SELECT * FROM test WHERE id = 2000;
+------+--------+---------+
| id   | amount | amount2 |
+------+--------+---------+
| 2000 |     92 |      17 |
+------+--------+---------+
1 row in set (0.05 sec)
```
*Note:* This query shows `amount` value before the query in the *first* terminal is finished.
When query in first terminal is finished this SELECT will return `amount = 93`.
