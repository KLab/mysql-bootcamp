#############
Transaction
#############

This chapter describes Atomicity and Isolation in MySQL.

What's transaction 
=====================

Transaction is an unit of work.

Since many users (even one user) access to system concurrently,
Database should support atomicity and isolation for transaction.

Atomicity means ``do all, or do nothing``.

Consider situation on transferring money from account A to account B.

schema:

.. code:: sql

    create table account (
        id INTEGER PRIMARY KEY AUTO_INCREMENT,
        money INTEGER
    )

code:

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

.. note::

    Both code of php and Python 
