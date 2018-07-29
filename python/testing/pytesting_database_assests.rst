Pytesting Database Assets
=========================

Created: 2018-07-29

A few years ago I needed a way to test Microsoft SQL Server stored procedures.  Some of these stored procedures were
hundreds or even thousands of lines long, and manual testing was cumbersome.  The following article is a result of this
search for a better way.  This process is not perfect, but it has made maintaining database assets much easier and less
stressful.  The process will continue to change as techniques are perfected, best practices learned, and tools are
discovered and/or created. I invite anyone to contribute and/or add to the conversation.

Database Testing in a Nutshell
------------------------------

Unit testing is the method of individually testing the smallest parts of a software system.  This means isolating a
system's functions, running them individually against sets of conditions, and checking the results against
predefined expectations.  A passing test is when the results match the expectations.  On the other hand, a
failure could be when the results don't match the expectations or an error is raised by a part while it is being tested.
In regards to databases we can test a number of things.  Anything responsible for a decision can be tested.  A few
examples are listed below.

- views
- table constraints
- functions
- triggers 
- stored procedures

There are a few options to accomplish database testing, but we will only review pytest in detail.  A google search as of
the writing of this article implies that SQL Server Data Tools is the preferred method for Microsoft SQL Server.  However, there is an
issue.  `The article for unit testing with SQL Server Data Tools
<https://msdn.microsoft.com/en-us/library/jj851212(v=vs.103).aspx>`_ states that *"this documentation is archived and is
not being maintained."* My interpretation of this is that it's deprecated and will be replaced.  Another option is
TSQLT, and it seems to be referenced by a few others.  My thought is that this is the preferred *free* choice to test
SQL Server assets.  However, I don't like that it alters the database to achieve its goals.  Finally, there is a tool
available from Redgate, but I am not familiar with this option and it isn't cheap.  For Postgres there are a number of
options: pgTap, PGUnit, simple pgunit, and testgres.  However, my focus has been with Microsoft SQL Server, so I can't
comment on which of these Postgres tools are the most relevant.

Finally, there is pytest and it's associated parts.  This article won't go into the details of pytest, but it's
recommended that a `review of the pytest documentation <https://docs.pytest.org/en/latest/index.html>`_ should be
done.  In general though, there are three main components to pytest that will be used in this article: fixtures, tests,
and pytest itself.

With unit testing we need to set up conditions before tests are run, and then tear them down afterwards.  In an ACID
compliant database, transactions meter the flow of additions and updates into the system, and we can leverage them to
augment the set up and tear down processes.  Typically, transactions ensure integrity in the database.  This means that
an addition or update won't be persisted unless we tell the database to commit the change.  We can use this to our
advantage in that we can undo the changes made during set up and testing with a single rollback command.

.. note::

    The approach in this document uses an empty copy of a database.  Empty means that the tables, views, triggers, and
    other assets are present; but the tables contain no data records.  By having an empty database we have complete
    control over the contents of the database.

Fixtures and Tests
------------------

Fixtures are python functions that are used to set up the conditions for a test.  They are denoted with a decorator like
so.

::

    @pytest.fixture
    def something()
        #Set up some_object here
        yield some_object    
        #Tear it down after the yield

Once declared the fixture can be passed as a parameter to a test.  They can be used to set up database connections,
cursors, and data records.

Then there are the tests.  In general these are functions or methods that execute assert statements.  The test function
name must start with test\_, and they need to be located in a file named test_*.py or \*_test.py so that pytest can find
it.  To make things easier, a helper function can be created to call the database asset that is being tested.  This will
cut down on repetitive code and allows for consistency.  


.. note::

    There are different modules that can be used to access the database being tested.  Since the focus of this article
    is Microsoft SQL Server we use pyodbc to access it.  It's up to you to find the module that you're most comfortable
    with.

Setting up pytest
-----------------

My preferred method of installing pytest is with pip into a virtualenv. This means a virtualenv needs to be created, and
this can be done with the following command in the directory of your choice.

::

    python -m venv my_virtualenv_name
    
Once that's done we can install pytest into it.

::

    /where/my/virtualenv/is/installed/my_virtualenv_name/bin/pip install pytest

.. note::

    There is another, newer approach to this and that's to use pipenv. You may want to explore that
    option instead. However, it is recommended that one of these two approaches is used.

Once installed the easiest way to invoke pytest involves navigating to the root directory of the tests and running

::

    /where/my/virtualenv/is/installed/my_virtualenv_name/bin/pytest

A specific file can be passed to pytest to run just the tests in that file. There are also options that can be included
in the call to pytest, some personal favorites include

- ``-s``: sets the capture method to "no" which allows the output from print statements to be shown.
- ``-m``: runs tests that are marked with a certain string. This is done with the ``@pytest.mark.my_marker_name`` decorator. 
- ``--pdb``: runs the debugger when an error is encountered. This allows inspection of the environment where a test failed.
- ``--lf``: runs only the tests that failed on the last attempt. This is very convenient if there is a large number of tests.

Setting Up Tests
--------------------

Armed with knowledge from the previous section we'll dive into setting up tests.  We'll need fixtures for a connection
and cursor that we mentioned in a prior section.

::

    @pytest.fixture(scope='module')
    def password():
        with open('pass.word') as f:
            pw = f.readline().strip()
        yield pw

    @pytest.fixture(scope='module')
    def cnxn(password):
        cnxn = pyodbc.connect('DSN=bruisedthumb;UID=bruised;PWD={}'.format(password))
        yield cnxn
        cnxn.close()
    
    @pytest.fixture
    def cursor(cnxn):
        cursor = cnxn.cursor()
        yield cursor
        cnxn.rollback()

These three fixtures are all we need to connect to the database.  Notice that a parameter is being passed into the cnxn
fixture.  This is the password fixture, and it's yielding the string that represents the database password.  In this
case we're pulling it from a file. Note that the internals of the password fixture can be different.  We could have
pulled it from an environment variable instead.  The key is that it yields the password.  **Don't hardcode the password
and don't commit the password to your code repository.**  Anyway, we're using that password in our call to the
pyodbc.connect function in the cnxn fixture.  The result of which is a yielded connection object that is used by tests
and other fixtures. Next, ``cnxn.close()`` is the tear down part of the fixture, and it will close the database
connection once pytest determines that the fixture is no longer needed.  The cursor fixture is similar to the cnxn
fixture except it's yielding a cursor.  The difference here is that instead of closing the cursor we're using the cnxn's
rollback method to cancel the database transaction.

Fixture Scope
~~~~~~~~~~~~~

In the example above, the cnxn fixture was passed a scope parameter.  This tells pytest to reuse the fixture until it's
done with the specified scope.  In this case we want to reuse a single database connection with all of our tests. This
saves time as reopening the connection can be time-consuming.  However, we want new cursors with each test as they
rollback the affects of the tests as they're disposed.  One last note, by default pyodbc opens the connection with
auto-commit turned off.  That's why we don't need to start the cursor setup with a database transaction statement.

Fixtures and conftest.py
~~~~~~~~~~~~~~~~~~~~~~~~

You may find out that you have a number of fixtures that should be reused.  These fixtures should be placed in the
conftest.py file. pytest looks at this special file for fixtures to use with the tests it finds.  You may also find out
that you have a large number of global fixtures in your conftest.py file.  One way to get around this is to create other
fixture files and then import those files into conftest.py. Importing the fixtures directly into the test files should
be avoided.

Data Record Fixtures
~~~~~~~~~~~~~~~~~~~~

Connections and cursors aren't the only fixtures we can create.  Fixtures can be used to insert records into the
database using our cursor fixture.  This way, a fresh record set can be built up in the database for testing.  Remember,
the  database starts out empty with each test, so we need to fill it with data for each test.  Just be aware that ID
values will increment in tables where the primary key is an identity field unless steps are taken to avoid this.
Otherwise, the ID values will need to be retrieved once the records are created.  Regardless, once a test is completed
the data will be removed.

Since we're using Python to build our data we can leverage Python modules.  One example is python-dateutil,
specifically its relativedelta module.  It's great for reliably creating dates that need to have certain conditions.
For example, it's easy to generate dates that are always a certain amount of time in the past or future.  The module
also makes it easy to get a date that's at the end of a month regardless of which month it is.  Below is how this would
be done.

::

    from datetime import date
    from dateutil.relativedelta import relativedelta

    d = date.today()
    eom_two_months_ago = d + relativedelta(months=-2, day=31)

It doesn't matter if the month in question is 31, 30, 29, or 28 days long; day=31 will figure it out.

Our First Test
~~~~~~~~~~~~~~

It's good to start with something simple, for example:

::

    def test_something(cursor):
        rs = cursor.execute('select id from some_table').fetchval()
        assert len(rs) == 0

This will confirm a few things:

- the tests can connect to the database
- pytest is working as expected
- the connection and cursor fixtures are set up correctly

Once this test passes the rest of the testing suite can be created.

A Sample Test
-------------

This section refers to the code `in my pytesting_db repo <https://github.com/danclark5/pytesting_db>`_.  It has the same
cnxn and cursor fixtures, but it has some others to insert data into the empty test database.  Either way `the database
schema <https://github.com/danclark5/pytesting_db/blob/master/database_setup.sql>`_ shows we have three tables, a view
and a stored procedure.  The stored procedure, get_project_cost, accepts a project id and then calculates and returns
the total cost for the project.  Running the tests will show that the cost that it's returning is incorrect.  The stored
procedure is returning 101.45 instead of 200.42.  At this point a look at the stored procedure is needed to find the
problem.  This problem is that the stored procedure is adding the cost and quantity instead of multiplying them.  This
is an easy fix by changing the expression to multiplication.

The view is also returning an error.  This view is meant to denormalize the data so that each row has the project name
as well as the name of the supply item.  What we're finding is that it's returning sixteen rows instead of eight.  In
this case a look into the logic in the view reveals that the view's join condition is incorrect.  It's generating a
cross join which is a situation where each row in the first table is joined to each row in the second.  This is fixed by
correcting the join condition so that project_id field on project_supplies is mated to id on projects table.

`Here are the alter statements needed to correct the issues
<https://github.com/danclark5/pytesting_db/blob/master/database_fixes.sql>`_.

Benefits, Drawbacks, and Next Steps
-----------------------------------

This approach to testing database assets has served me well.  It has the appearance of being able to handle database
migrations in that changes to the connection and cursor fixtures should be all that are needed to allow a testing suite
to work with a different database product, but this is something I haven't done yet.  Of course there might be other
changes needed if the source database uses product specific features.  

The fixtures are reusable for different tests, but there is a degree of brittleness here as different tests may have
different needs.  Then again, if something does break there will be instant feedback.  We can also group the tests and
fixtures into their own files and pytest will be able to collect them all and run them.

Since we are using pytest we can also leverage Python's vast library of modules.  In addition to python-dateutil's
relativedelta the pdb module can drop us into any place in our tests with a debugger to inspect the test states.

Finally, another benefit is being able to use the assert keyword in place of remembering or looking up assert functions
as is the case with the unittest module.

However, there are a few disadvantages.  It's command-line based with no GUI to help.  However, there are tools such as
`PyCharm <https://www.jetbrains.com/pycharm/>`_ that address this.   Code coverage is rendered useless as pytest can't
see into the database and inspect the assets.  There isn't an equivalent to TSQLT's AssertEqualsTable, but it wouldn't
take much to make an equivalent.

That said, there are a few next steps for this process.  First, rethinking how to write stored procedures and functions
so that they are geared towards unit testing.  If they are too long then we run into complicated situations that attempt
to test every combination possible by a stored procedure.  I've witnessed this, and it becomes very difficult to create
tests as there were vast categories of test conditions that all needed to be accounted for.

It would also be nice to figure out a better way to import the data into the database. It might be better to use a
test_source_database to copy the data instead of using individual inserts.

Closing Thoughts
----------------
I started looking for a solution years ago, and I was desperate.  There wasn't a reliable way to test my database assets
without spending an absurd amount of time manually testing things.  This approach has given me confidence in my database
assets, and has made making changes much easier and less stressful.  I hope this helps someone else, and it'll be
something I'll continue to use as long as I have database assets to maintain.

If you have something you'd like to contribute or point out please let me know.

