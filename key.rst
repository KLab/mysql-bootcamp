Index (key)
============

What's Index
-------------

Index and Key have same meaning.
Index is general word but Key is Database specific word.

Index is for searching record effectively.

.. code:: bash

    $ sudo apt-get install ipython python-mysql
    $ ipython

    In [1]: import MySQLdb
    In [2]: con = MySQLdb.connect(user='testuser', passwd='password', db='testdb')
    In [3]: cur = con.cursor()
    In [4]: cur.execute("CREATE table test1 (id INTEGER PRIMARY KEY AUTO_INCREMENT, val INTEGER)")
    Out[4]: 0L
    In [5]: values = [(i, i*2) for i in range(1, 100000)]
    In [6]: cur.executemany("INSERT INTO test1 VALUES (%s, %s)", values)
    Out[6]: 99999L
    In [7]: con.commit()

    $ mysql -u testuser -ppassword testdb

    mysql> select * from test1 where id=100;
    +-----+------+
    | id  | val  |
    +-----+------+
    | 100 |  200 |
    +-----+------+
    1 row in set (0.00 sec)

    mysql> select * from test1 where val=200;
    +-----+------+
    | id  | val  |
    +-----+------+
    | 100 |  200 |
    +-----+------+
    1 row in set (0.02 sec)

As you can see, select by PK consumes 0.00 sec and select by where consumes
0.02 sec on table having 100000 records.

If 1M records in Table, selecting without Index may consume 0.2sec. This means
only 5 (* CPU cores) querys can be executed in one second.

We need to execute thoughounds of queries in one second.


Data Structure
---------------

InnoDB is primary storage engine of MySQL.

InnoDB uses tree data structure to store and search records.

+-----+-----------+
| id  | name      |
+=====+===========+
| 1   | Kabul     |
+-----+-----------+
| 2   | Qandahar  |
+-----+-----------+
| 3   | Herat     |
+-----+-----------+
| 4   | Mazar     |
+-----+-----------+
| 5   | Amsterdam |
+-----+-----------+
| 6   | Rotterdam |
+-----+-----------+

This table is stored like this:

.. aafig::

                 +--------+
                 | "root" |
                 +--------+
                 |   3    |
                 +--------+
                  |      |
         +--------+      +----+
         |                    |
         V                    V
   +----------------+     +-------+
   |  < 3           |     | "3 =<"| node
   +---+------------+     +-------+
   | 1 | Kabul      |     |   6   |
   +---+------------+     +-------+
   | 2 | Qandahar   |      |     |
   +---+------------+      |     +--+
        ^                  |        |
        |                  V        V     leaf
        |   +----------------+     +---------------+
        +-->|  < 6           |<--->| "6 =<"        |
            +---+------------+     +---+-----------+
            | 3 | Herat      |     | 6 |"Rotterdam"|
            +---+------------+     +---+-----------+
            | 4 | Mazar      |
            +---+------------+
            | 5 | Amsterdam  |
            +---+------------+

This figure is not accurate, but illustrate essence.

Important note:

* Records are sorted by key and split.
* Records stored in leaf node.
* Leaf nodes are linked to next leaf.
* All nodes stored in fixed-size block.
  Node is split when it's size exceeds block size.


How to find record
-------------------

``SELECT * FROM country WHERE id=5``:

.. aafig::

                 +--------+
                 | "root" |
                 +--------+
                 |   3    |
                 +--------+
                  |      o (1) 3 < 5
                  |      |
         +--------+      +----+
         |                    |
         V                    V
   +----------------+     +-------+
   |  < 3           |     | "3 =<"| node
   +---+------------+     +-------+
   | 1 | Kabul      |     |   6   |
   +---+------------+     +-------+
   | 2 | Qandahar   |      o     |
   +---+------------+      |     +------+
        ^                  | (2) 5<6    |
        |                  V            V    leaf
        |   +----------------+     +---------------+
        +-->|  < 6           |<--->| "6 =<"        |
            +---+------------+     +---+-----------+
            | 3 | Herat      |     | 6 |"Rotterdam"|
            +---+------------+     +---+-----------+
            | 4 | Mazar      |
            +---+------------+
            | 5 | Amsterdam  | (3) Find!
            +---+------------+

1. Compare 5 with root node value (3). Since 3 < 5, go right child.
2. Compare 5 with current node value (6). Since 5 > 6, go left child.
3. Current node is leaf. So search 5 in this leaf.

As you saw, InnoDB doesn't need to compare key to all records.
This is why search by Key is fast.


.. note::

    If using sorted array, binary search is fast as Tree.
    But tree structure is faster on deleting and inserting.

How to fetch range
------------------

``SELECT * FROM country WHERE id>=2 LIMIT 3``:

.. aafig::

                 +--------+
                 | "root" |
                 +--------+
                 |   3    |
                 +--------+
         (1) 2<3  o      |
                  |      |
         +--------+      +----+
         |                    |
         V                    V
   +----------------+     +-------+
   |  < 3           |     | "3 =<"|
   +---+------------+     +-------+
   | 1 | Kabul      |     |   6   |
   +---+------------+     +-------+
   | 2 | Qandahar   |      |     |
   +---+------------+      |     +------+
        o   (2) find       |            |
        |                  V            V
        |   +----------------+     +---------------+
        +-->|  < 6           |<--->| "6 =<"        |
 (3) follow +---+------------+     +---+-----------+
     link   | 3 | Herat      |     | 6 |"Rotterdam"|
            +---+------------+     +---+-----------+
            | 4 | Mazar      |
            +---+------------+
            | 5 | Amsterdam  |
            +---+------------+

1. Compare 2 with root node value (3). Since 2 < 3, go left child.
2. This is leaf and find record that's id = 2.
3. Scan and follow link until reach limit.

Since leaf node is linked to next leaf, InnoDB can scan rows efficiently.


Secondary Index
---------------

Searching with primary key reaches to records directory.
If you create other keys (index or constraint), InnoDB creates secondary index.

Secondary index is Tree too. But it's leaf node stores primary key instead of
actual record.

For example:

schema::

    create table example (
        id INTEGER PRIMARY KEY,
        a INTEGER,
        b INTEGER,
        KEY a (a),
        KEY ab (a, b)
    )

table::

    +---------+-----+-----+
    | id(PK)  | a   | b   |
    +=========+=====+=====+
    | 1       | 10  | 11  |
    +---------+-----+-----+
    | 2       | 20  | 22  |
    +---------+-----+-----+
    | 3       | 30  | 33  |
    +---------+-----+-----+
    | 4       | 40  | 44  |
    +---------+-----+-----+
    | 5       | 50  | 55  |
    +---------+-----+-----+
    | 6       | 60  | 66  |
    +---------+-----+-----+

index on (a)::

    +-----+-----+
    | a   | id  |
    +=====+=====+
    | 10  | 1   |
    +-----+-----+
    | 20  | 2   |
    +-----+-----+
    | 30  | 3   |
    +-----+-----+
    | 40  | 4   |
    +-----+-----+
    | 50  | 5   |
    +-----+-----+
    | 60  | 6   |
    +-----+-----+

index on (a, b)::

    +-----+-----+-----+
    | a   | b   | id  |
    +=====+=====+=====+
    | 10  | 11  | 1   |
    +-----+-----+-----+
    | 20  | 22  | 2   |
    +-----+-----+-----+
    | 30  | 33  | 3   |
    +-----+-----+-----+
    | 40  | 44  | 4   |
    +-----+-----+-----+
    | 50  | 55  | 5   |
    +-----+-----+-----+
    | 60  | 66  | 6   |
    +-----+-----+-----+

1. Index may be bigger than you think. consumes significant space like table. 

2. All indicies has PK implicitly.

3. When you add PK to index manually, the index contains PK twice.

4. If PK is big (ex, ``VARCHAR(255)``), all indicies is big.

5. Index ``(a, b)`` can be used to search by only ``a``.
   So you should remove ``KEY a (a)``.


Tips: covering index
----------------------

Normally, searching by secondary index cause two step lookup: (1) search PK by index,
(2) search record by PK.

But when all required columns are contained in the index, MySQL doesn't search actual
record.

For example, ``SELECT id FROM example WHERE a BETWEEN 20 AND 50`` only requires ``a``
and ``id``.


Sorting and ranges
-------------------

All index is sorted by lexicographical order::

    (1, 1) < (1, 2) < (1, 3) < (2, 1) < (2, 2) < ...

``SELECT * FROM example WHERE a BETWEEN 20 AND 30 ORDER BY (a, b)`` is efficient.

``SELECT * FROM example WHERE b BETWEEN 20 AND 30 ORDER BY (a, b)`` is not efficient
because MySQL can't use the index ``(a, b)`` for selecting. On this case, MySQL scan
full table to find records matches ``b BETWEEN 20 AND 30`` and copy them to temporary
table. After scan, MySQL sorts the temporary table.


explain
--------

MySQL 5.5 can show how ``SELECT`` query executed by ``EXPLAIN SELECT...`` query.

.. note::

    MySQL 5.6 supports EXPLAINing INSERT, UPDATE, DELETE, ... queries too.
    On MySQL 5.5, you can explain UPDATE and DELETE by replace it to SELECT.

example::

    mysql> explain select * from test1 where val=200;
    +----+-------------+-------+------+---------------+------+---------+------+--------+-------------+
    | id | select_type | table | type | possible_keys | key  | key_len | ref  | rows   | Extra       |
    +----+-------------+-------+------+---------------+------+---------+------+--------+-------------+
    |  1 | SIMPLE      | test1 | ALL  | NULL          | NULL | NULL    | NULL | 100808 | Using where |
    +----+-------------+-------+------+---------------+------+---------+------+--------+-------------+
    1 row in set (0.00 sec)

    mysql> explain select * from test1 where id=100;
    +----+-------------+-------+-------+---------------+---------+---------+-------+------+-------+
    | id | select_type | table | type  | possible_keys | key     | key_len | ref   | rows | Extra |
    +----+-------------+-------+-------+---------------+---------+---------+-------+------+-------+
    |  1 | SIMPLE      | test1 | const | PRIMARY       | PRIMARY | 4       | const |    1 |       |
    +----+-------------+-------+-------+---------------+---------+---------+-------+------+-------+
    1 row in set (0.00 sec)


How to make an effective indicies
------------------------------------

Index has significant cost. So you should not create index everywhere.

Since finding slow query is easier than finding unnecessary index,
I recommend start with minimum, obviously required indicies.

Slow query log is useful feature to find slow query.
It logs queries consumes specified execution time.

You can insert many dummy data to development environment.
Before releasing, check slowlog, find slow queries and consider how to solve.
(Sometimes there is better way than creating index.)

