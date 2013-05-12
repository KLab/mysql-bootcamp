MySQL setup
============

Trying SQL directly is important even if you use ORM to generate SQL.
Having testable emvironment is highly recommended.

Install
--------

If you has VM for development, MySQL is already there.

Otherwise, use package manager (ex, ``sudo apt-get install mysql``)
or use installer provided by Oracle.

Create User
------------

Login mysql with admin user.

.. code::

    mysql> create user 'testuser' identified by 'password';


Create database
----------------

.. code::

    mysql> create database 'testdb'

Connect
--------

.. code:: bash

    $ mysql -utestuser -p testdb
    passowrd: 


