#############
 Transaction
#############

This chapter describes Atomicity and Isolation in MySQL.

What's transaction 
=====================

Transaction is an unit of work.

Since many users (even one user) access to system concurrently,
Database should support **atomicity** and **isolation** for transaction.

Atomicity means "do all, or do nothing".
Isolation means result of query in transaction isn't affected by another transaction.

Consider situation on transferring money from account A to account B.

The schema:

.. code:: sql

    create table account (
        id INTEGER PRIMARY KEY AUTO_INCREMENT,
        money INTEGER NOT NULL DEFAULT 0
    )

php with Propel:

.. code-block:: php
    :linenos:

    <?php
    class AccountService {
    // ...
    static function transfer(int $from_id, int $to_id, int $amount) {
        $con = PropelPDO::getConnection('master');
        $con->begin();
        try {
            $from_account = AccountPeer::retrieveByPK($from_id);
            if ($from_account === null) {
                throw new Exception("Invalid account $from_id");
            }
            if ($from_account.getAmount() < $amount) {
                throw new Exception("account $from_id doesn't have enough money. (" .
                                    $from_account->getAmount() . " < $amount)");
            }
            $to_account = AccountPeer::retrieveByPK($to_id);
            if ($from_account === null) {
                throw new Exception("Invalid account $to_id");
            }
            $con->commit();
        } catch (Exception $e) {
            $con->rollback();
            throw $e;
        }
    }

Python with SQLAlchemy:

.. code-block:: python
    :linenos:

    # accountservice.py
    # ...
    def transfer(from_id, to_id, amount):
        with Session.begin():  # get connection and begin transaction
            # commit on exit without exception.
            # Or rollback on exit with exception
            from_account = Session.query(Account).get(from_id)
            if not from_account:
                raise Exception("Bad account id: %r" % (from_id,))

            if from_account.amount < amount:
                raise Exception("account %d doesn't have enough money. (%d < %d)" %
                                (from_id, from_account.amount, amount))

            to_account = Session.query(Account).get(to_id)
            if not to_account:
                raise Exception("Bad account id: %r" % (to_id,))

            from_account.amount -= amount
            to_account.amount += amount

After transaction, money is completly transferred or not transferred.
Total amount should not be changed. This is atomicity.

While this transaction, ``SELECT SUM(amount) FROM account WHERE id IN (:from_id, :to_id)``
from other transaction always shows right value. Other transaction always see state of before or
after transaction but not intermediate state.

.. note::

    MySQL can select transaction isolation level. It's default is "repeatable read".
    This chapter suppose this transaction isolation level.

autocommit
===========
MySQL has **autocommit** mode that is default on.

autocommit means one query is one transaction unless explicitly ``BEGIN``.
When autocommit is disabled, first query starts transaction implicitly and
``COMMIT`` is required to save changes.

example
~~~~~~~

In this chapter, following table is used repeatedly as example.

.. code:: sql

    CREATE TABLE `test1` (
        `id` int(11) NOT NULL,
        `other_id` int(11) DEFAULT NULL,
        `data` int(11) DEFAULT NULL,
        PRIMARY KEY (`id`),
        KEY `idx_1` (`other_id`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;


Start 2 mysql session. Update from one session and check from other session.
Two sessions are separated with indent level.

autocommit=1:

.. code:: sql

    mysql> select @@autocommit;  # Check autocommit is enabled.
    +--------------+
    | @@autocommit |
    +--------------+
    |            1 |
    +--------------+
    1 row in set (0.00 sec)

    mysql> insert into test1 (id, other_id, data) values (1, 100, 123);
    Query OK, 1 row affected (0.01 sec)

                                                    # inserted row can be seen from another session.
                                                    mysql> select * from test1;
                                                    +----+----------+------+
                                                    | id | other_id | data |
                                                    +----+----------+------+
                                                    |  1 |      100 |  123 |
                                                    +----+----------+------+
                                                    1 row in set (0.00 sec)

    # explicitly BEGIN transaction
    mysql> begin;
    Query OK, 0 rows affected (0.00 sec)

    mysql> insert into test1 (id, other_id, data) values (2, 90, 100);
    Query OK, 1 row affected (0.00 sec)

                                                    # new row can't be seen from another session.
                                                    # It's not committed yet.
                                                    mysql> select * from test1;
                                                    +----+----------+------+
                                                    | id | other_id | data |
                                                    +----+----------+------+
                                                    |  1 |      100 |  123 |
                                                    +----+----------+------+
                                                    1 row in set (0.00 sec)

    mysql> commit;
    Query OK, 0 rows affected (0.01 sec)

                                                    # committed row can be seen.
                                                    mysql> select * from test1;
                                                    +----+----------+------+
                                                    | id | other_id | data |
                                                    +----+----------+------+
                                                    |  1 |      100 |  123 |
                                                    |  2 |       90 |  100 |
                                                    +----+----------+------+
                                                    2 rows in set (0.00 sec)


autocommit=0:

.. code:: sql

    # Disable autocommit in this session.
    mysql> set @@autocommit=0;
    Query OK, 0 rows affected (0.00 sec)

    mysql> insert into test1 (id,other_id,data) values (3,80,99);
    Query OK, 1 row affected (0.00 sec)

                                                    # id=3 can't be seen yet.
                                                    mysql> select * from test1;
                                                    +----+----------+------+
                                                    | id | other_id | data |
                                                    +----+----------+------+
                                                    |  1 |      100 |  123 |
                                                    |  2 |       90 |  100 |
                                                    +----+----------+------+
                                                    2 rows in set (0.00 sec)

    mysql> commit;
    Query OK, 0 rows affected (0.00 sec)

                                                    mysql> select * from test1;
                                                    +----+----------+------+
                                                    | id | other_id | data |
                                                    +----+----------+------+
                                                    |  1 |      100 |  123 |
                                                    |  2 |       90 |  100 |
                                                    |  3 |       80 |   99 |
                                                    +----+----------+------+
                                                    3 rows in set (0.00 sec)


MVCC
=====

**MVCC** (Multi Version Concurrency Control) achieves both of concurrent update and isolation.

While execute update transaction, MVCC doesn't overwrite but create **new version**.

First query (including SELECT query) in transaction uses newest version.
Continued SELECT queries sees the version same to first query.

example
~~~~~~~~

.. code:: sql

    mysql> begin;
    Query OK, 0 rows affected (0.00 sec)

    mysql> select * from test1;  # This transaction is set to this version.
    +----+----------+------+
    | id | other_id | data |
    +----+----------+------+
    |  1 |      100 |  123 |
    |  2 |       90 |  100 |
    |  3 |       80 |   99 |
    +----+----------+------+
    3 rows in set (0.00 sec)

                                                    # update (and autocommit) from another session creates new version.
                                                    mysql> update test1 set other_id=70 where id=2;
                                                    Query OK, 1 row affected (0.00 sec)

    # But left session continue to see old version.
    mysql> select * from test1;
    +----+----------+------+
    | id | other_id | data |
    +----+----------+------+
    |  1 |      100 |  123 |
    |  2 |       90 |  100 |
    |  3 |       80 |   99 |
    +----+----------+------+
    3 rows in set (0.00 sec)

    # Stop transaction
    mysql> rollback;
    Query OK, 0 rows affected (0.00 sec)

    # Next (autocommitted) transaction sees newest version.
    mysql> select * from test1;
    +----+----------+------+
    | id | other_id | data |
    +----+----------+------+
    |  1 |      100 |  123 |
    |  2 |       70 |  100 |
    |  3 |       80 |   99 |
    +----+----------+------+
    3 rows in set (0.00 sec)


Lock
=====

MVCC isolates between update transaction and readonly transaction automatically.

But concurrent two update transaction cause **Lost update** problem.

lost update example
~~~~~~~~~~~~~~~~~~~~

When multiple transaction "Read - Modify - Write" in same time, one transaction
overwrites another transaction. This problem is called **Lost Update**.

.. code::

    # A player adds 3 points to team.
    > SELECT point FROM team WHERE team_id=3;
    5

                                                    # Another player in same team adds 5 points.
                                                    > SELECT point FROM team WHERE team_id=3;
                                                    5

    > UPDATE team SET point=8 WHERE team_id=3;

                                                    > UPDATE team SET point=10 WHERE team_id=3;

    # 3 points disappeared...


In php + Propel:

.. code-block:: php
    :linenos:

    <?php
    class QuestService {
        static function doQuest($player, $quest_id) {
            $con = Propel::getConnection('master');
            $con->beginTransaction();
            try {
                // ...
                $team = TeamPeer::retrieveByPK($player->$team_id, $con);
                $team->setPoint($team->getPoint() + $got_point);
                $team->save($con);
                // ...
                $con->commit();
            } except (Exception $e) {
                $con->rollback();
                throw $e;
            }
        }
    }

In Python + SQLAlchemy:

.. code-block:: python
    :linenos:

    # quest_service.py
    def do_quest(player, quest_id):
        with Session.begin():
            #...
            team = Session.query(Team).get(player.team_id)
            team.point += got_point
            #...


lock example
~~~~~~~~~~~~~~

Update query automatically acquires lock. ``SELECT ... FOR UPDATE`` also acquire lock.
And ``SELECT ... FOR UPDATE`` also ignore MVCC and read newest state.

.. code:: sql

    # A player adds 3 points to team.
    > BEGIN
    > SELECT point FROM team WHERE team_id=3 FOR UPDATE;
    5

                                                    # Another player in same team adds 5 points.
                                                    > BEGIN
                                                    > SELECT point FROM team WHERE team_id=3 FOR UPDATE;
                                                    # ... waiting...

    > UPDATE team SET point=8 WHERE team_id=3;
    > COMMIT

                                                    8
                                                    > UPDATE team SET point=13 WHERE team_id=3;
                                                    > COMMIT

Propel doesn't support `FOR UPDATE`.
So we customize code generator to make ``retrieveByPk4Update()`` automatically.
Here is php + Propel example:

.. code-block:: php
    :linenos:

    <?php

    // ...
    class QuestService {
        static function doQuest($quest_id, $player) {
            $con = Propel::getConnection('master');
            $con->beginTransaction();
            try {
                // ...
                $team = TeamPeer::retrieveByPK4Update($player->$team_id, $con);
                $team->setPoint($team->getPoint() + $got_point);
                $team->save($con);
                // ...
                $con->commit();
            } except (Exception $e) {
                $con->rollback();
                throw $e;
            }
        }
    }

SQLAlchemy supports ``FOR UPDATE`` as ``query.with_lockmode('update')``.
You can use this not only for selecting by PK.

.. code-block:: python
    :linenos:

    # quest_service.py
    def do_quest(quest_id, player):
        with Session.begin():
            #...
            team = Session.query(Team).with_lockmode('update').get(player.team_id)
            team.point += got_point
            #...


