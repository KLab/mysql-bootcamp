MySQL setup
============

Trying SQL directly is important even if you use ORM to generate SQL.
Having testable emvironment is highly recommended.

Install
--------

If you has VM for development, MySQL is already there.

On Windows or Mac, download installer from Oracle. http://dev.mysql.com/downloads/installer/

On Linux, use package manager. For example, Debian wheezy::

    $ sudo apt-get install mysql-server mysql-client

You may be asked to set *root* password. This is a *root* user of MySQL. (not for OS user)

Now you can login to mysql with root user.

.. code:: bash

    $ mysql -uroot -p
    Enter password: (enter password here)
    Welcome to the MySQL monitor.  Commands end with ; or \g.
    ...
    Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

    mysql>


Create Database and User
-------------------------

You can use root use for everything. But this is dangerous.
So create additional user for general work is highly recommended.

Creating `testdb` database and user for accessing to the database:

.. code::

    mysql> CREATE DATABASE testdb;
    Query OK, 1 row affected (0.00 sec)

    mysql> GRANT ALL on testdb.* TO testuser@localhost IDENTIFIED BY 'password';
    Query OK, 0 rows affected (0.00 sec)


MySQL checks where user come from. User ``'testuser'@'localhost'`` means `'testuser'`
can login from `'localhost'` (same to MySQL server).

Now you can login MySQL with testuser:

.. code:: bash

    $ mysql -utestuser -ppassword testdb


Now you can create tables in *testdb*.
