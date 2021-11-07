# Adventures in Django - Django Automated Testing
![header](https://i.postimg.cc/ZnKdGZNs/Django-testing-header.png)
This article assumes that you are familiar with the following:
+ designing Django models 
+ performing CRUD operations on a Django model
+ running a Django migration with `makemigrations` and `migrate`
+ creating unit tests with Python's [unittest](https://docs.python.org/3/library/unittest.html) framework

The purpose of this article is to encourage automated unit testing on Django models before committing the models schema in the database.

At the end of this article you should be able to:

+ create a test script within the Django ecosystem to test basic crud (**c**reating, **r**eading, **u**pdating and **d**eleting) operations on Django models
+ execute tests based on different granularity
+ execute a test script with various verbosity levels

## Django Models and Database

Django models are representations of database tables that belong to a database schema for an application. A database schema typically consists of tables, table fields and their constraints, indexes and table [relationships](https://python.plainenglish.io/adventures-in-django-model-relationships-325e29405cce). Since a Django model is an abstraction of a relational database table, it is implemented as a Python class with data and behavior.  
## Django Migration

![Django-migration-flow](https://i.postimg.cc/pTBjcymn/Django-Migration-Testing-flow.png)

The Django [migration](https://docs.djangoproject.com/en/3.2/topics/migrations/#module-django.db.migrations) process consists of two steps:

1. `makemigrations` command that creates migration files to describe the steps to translate a Python model class to a database table. These migration files reside in the `migrations` sub-directory inside an application directory. 
2. `migrate` command  to implement the steps in the migration files and create the corresponding tables and their relationships and constraints, if any, in the database.

Before executing step 2 of the migration process which commits our models in the database, we should take a detour and write tests for our Django models.  Model testing should provide coverage for:

+ data validation 

+ model behavior 

+ model relationships

+ basic crud operations

Django users can take advantage of model testing using various tools that are built into Django.

## Django dbshell

Since our tests require that migration files already exist for our models, we need to execute step 1 -- `makemigrations` of the migration process.  We can check in the database to ensure that no tables have been created for our models.  Assuming we are using the default `SQLite` database, we can invoke the `dbshell` command in the terminal:

```
$ python manage.py dbshell
SQLite version 3.36.0 2021-06-18 18:36:39
Enter ".help" for usage hints.
sqlite> .tables
sqlite>
```

Running the `.tables` command inside `dbshell` returns an empty row because we haven't yet created any tables in the database via the `migrate` command.

## Sample Application

For the purpose of this article, we are going to define a simple application, `pet`, involving two models, `Owner` and `Pet` where an owner can own one or more pets.  For simplicity, our models are defined as follows:

```python
# models.py
from django.db import models

# An Owner of one or more pets
class Owner(models.Model):
    name = models.CharField(max_length=20)
    def __str__(self):
        return self.name

# A Pet must have an owner
class Pet(models.Model):
    name = models.CharField(max_length=20)
    owner = models.ForeignKey(
        to=Owner,
        on_delete=models.CASCADE  # when an owner is deleted, so is their pet
    )
    def __str__(self):
        return self.name
```

Note that the field option, `on_delete=models.CASCADE` ensures that a `pet`object is deleted when an `Owner` object is deleted.

## Django Test

### Default Test Script

By default, a Django installation provides an empty test file, **tests.py**, for every application.

```shell
$ ls pet
__init__.py   admin.py  migrations/  tests.py
__pycache__/  apps.py   models.py    views.py
```

### TestCase

An empty **tests.py** may look like this:

```
from django.test import TestCase

# Create your tests here.
```

Notice that a `TestCase` class is imported from Django's `test` module. We will be writing our tests the object-oriented way by subclassing `TestCase`  to provide custom data and methods. `django.test.TestCase` itself is a subclass of Python's `unittest.TestCase` that allows a test to execute in isolation within a transaction. 

### Sample Data

In our test, we are going to define `DemoTests`, a subclass of `TestCase`.  We will provide sample data Inside `DemoTests`, by defining  `items`, a Python list of dictionaries matching an owner with their pets as a class property. For example:

```python
class DemoTests(TestCase):
    items = [
        {"owner": 'Katie', "pet": ['Toto', 'Kitty']},
        {"owner": 'Sue', "pet": ['Bunny', 'Scott']},
        {"owner": 'Lynn', "pet": ['Skylar']},
    ]

```

#### Helper Properties

In addition, we are going to define two additional helper class properties:

+ `owner_names`, a list of owner names in `items`
+ `pet_names`, a list of pet names in `items`

For example:

```python
    owner_names = [item["owner"] for item in items]
    # ['Katie', 'Sue', 'Lynn']
    pet_names = [name for item in items for name in item["pet"]]
    # ['Toto', 'Kitty', 'Bunny', 'Scott', 'Skylar']
```

#### Helper Methods

Next, we are going to define a helper class method, `pets_by_name`, that returns a list of pet names based on an owner name using the `items` data. For example:

```python
    # Given an owner name, return their a list of pets or an empty list
    def pets_by_owner(self, owner):
        for item in self.items:
            if item["owner"] == owner:
                return item["pet"]
        return []
```

### setUp() Method

Since our data, `items` , is a Python data structure, we need to convert the content in `items` to DJango models. `TestCase` provides a `setUp()` class method that is called once per transaction. We can define the `setUp()` method to create our Django models based on `items`. For example:

```python
    # Create data in the database once
    def setUp(self):
        for item in self.items:
            owner = Owner.objects.create(name=item["owner"])
            for pet in item["pet"]:
                Pet.objects.create(name=pet, owner=owner)

        self.assertEqual(
            Owner.objects.all().count(),
            len(self.owner_names))
        self.assertEqual(
            Pet.objects.all().count(),
            len(self.pet_names))
```

For each item in `items`, we use Django's [QuerySet API](https://docs.djangoproject.com/en/3.2/ref/models/querysets/#create) to create a model for `Owner` and `Pet` via `objects.create()`. Notice the last two `assertEqual()` methods. The first is to validate that the number of `Owner` objects created is equivalent to the number of owner names in `items`. The second is to validate that the number of `Pet` objects created is equivalent to the number of pet names in `items`.  The `setUp()` method can be a test to ensure that our models are created correctly.

## Define Tests

We successfully created the model objects in the `setUp()` class method. Next, we can define additional tests for the created models as individual class methods. These may include:

+ querying a non-existing owner
+ querying a non-existing pet
+ deleting an existing pet
+ deleting an existing owner
+ adding a new pet to an existing owner
+ updating the name of an existing pet

Each test should be small and specific targeting a particular unit of action. The [rule](https://docs.python.org/3/library/unittest.html#organizing-test-code) for naming test class methods in `unittest` is to prefix each test with "`test`".  For example, the test to query for a non-existing owner might be named `test_owner_not_found()`.

```python
    # Query for a non-existing owner
    def test_owner_not_found(self):
        print(f'\nRunning {self.id()}\n')
        names = list(self.owner_names)
        names.append('Lisa')
        for name in names:
            try:
                owner = get_object_or_404(Owner, name=name)
                self.assertEqual(owner.name, name)
            except Http404:
                print(f"Owner {name} not found")
```

Note that `unittest` provides a class method, [`id()`](https://docs.python.org/3/library/unittest.html#unittest.TestCase.id) , to identify the name of the test class method. We can use this method to display the test name. 

`unittest` runs each test based on the alphabetical order of the test names.  

> Note: The order in which the various tests will be run is determined by sorting the test method names with respect to the built-in ordering for strings.

If you want your tests to run in a particular order, then you have to be creative when naming your tests. For example, `test_01xxxx()` will run before `test_02xxxx()`.

To see sample tests for this article, take a look at my GitHub [repo](https://github.com/mchesler613/django_testing).

## Run Tests

Django's `unittest` test framework provides for a flexible test execution. 

### Run All Tests

To execute all the tests defined in **tests.py**, in our `pet` application, simply type on the console:

```shell
$ ./manage.py test
```

or

```
$ ./manage.py test pet.tests
```

### Run a Test Script

If we have multiple test files, **test1.py** and **test2.py**, inside our application,  `pet`, we can choose to run only tests pertaining to a particular test file. For example:

```
$ ./manage.py test pet.test2
```

will only execute tests within **test2.py**.

### Run a TestCase

If we have multiple `TestCase`s in our **tests.py** file, we can choose to execute a particular `TestCase` named `DemoTests`. For example:

```shell
$ ./manage.py test pet.tests.DemoTests
```

### Run a Test Method

For finer granularity, we can also choose to run only a particular test method, such as `test_owner_not_found()` inside our test script, **tests.py**. For example:

```shell
$ ./manage.py test pet.tests.DemoTests.test_owner_not_found
```

## Test Output

### Default Verbosity

A sample output with a default verbosity of a successful test might be as follows:

```
Creating test database for alias 'default'...
System check identified no issues (0 silenced).
.
.
.
----------------------------------------------------------------------
Ran 3 tests in 0.022s

OK
Destroying test database for alias 'default'...
```

### Test Verbosity Level

If you would like more verbose output when running tests, you can supply an argument `-v` followed by a number, `0`, `1` (default), `2`, or `3` to the `test` command.  The higher the verbosity, the higher the number.  For example:

```shell
$ ./manage.py test -v 2
```

will generate additional output, like so:

```shell
Creating test database for alias 'default' ('file:memorydb_default?mode=memory&c
ache=shared')...
Operations to perform:
  Synchronize unmigrated apps: messages, staticfiles
  Apply all migrations: admin, auth, contenttypes, pet, sessions
Synchronizing apps without migrations:
  Creating tables...
    Running deferred SQL...
Running migrations:
  Applying contenttypes.0001_initial... OK
  Applying auth.0001_initial... OK
  Applying admin.0001_initial... OK
  Applying admin.0002_logentry_remove_auto_add... OK
  Applying admin.0003_logentry_add_action_flag_choices... OK
  Applying contenttypes.0002_remove_content_type_name... OK
  Applying auth.0002_alter_permission_name_max_length... OK
  Applying auth.0003_alter_user_email_max_length... OK
  Applying auth.0004_alter_user_username_opts... OK
  Applying auth.0005_alter_user_last_login_null... OK
  Applying auth.0006_require_contenttypes_0002... OK
  Applying auth.0007_alter_validators_add_error_messages... OK
  Applying auth.0008_alter_user_username_max_length... OK
  Applying auth.0009_alter_user_last_name_max_length... OK
  Applying auth.0010_alter_group_name_max_length... OK
  Applying auth.0011_update_proxy_permissions... OK
  Applying auth.0012_alter_user_first_name_max_length... OK
  Applying pet.0001_initial... OK
  Applying sessions.0001_initial... OK
System check identified no issues (0 silenced).
test_add_pet (pet.test1.DemoTests) ...
Running pet.test1.DemoTests.test_add_pet

ok
....
----------------------------------------------------------------------
Ran 7 tests in 0.022s

OK
Destroying test database for alias 'default' ('file:memorydb_default?mode=memory
&cache=shared')...
```

Notice that the Django `migrations` are performed on a test database, `default`, based on the migration files including `pet.0001_initial.py` created by `makemigrations` .  By default, Django testing ensures that the test database is destroyed after testing is concluded.  For even more verbose output, you can try option `3`.

## Conclusion

Adding unit testing in various stages of Django development is generally good practice for a developer. As a web application grows, its complexity increases. In addition to Django models, Django's testing framework also provides [tools](https://docs.djangoproject.com/en/3.2/topics/testing/tools/) for testing views as well. Ensuring that each part of a Django application is well-tested before deployment is crucial and is part and parcel of a successful web development project.
