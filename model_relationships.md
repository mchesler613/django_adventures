# Adventures in Django: Model Relationships

This article assumes a readership who is familiar with database schema design. [Django](https://djangoproject.com/) explains its architecture as following an [_MTV_](https://docs.djangoproject.com/en/3.1/faq/general/#django-appears-to-be-a-mvc-framework-but-you-call-the-controller-the-view-and-the-view-the-template-how-come-you-don-t-use-the-standard-names) (Model-Template-View) paradigm where model is the central component that encapsulates data and behavior for a web application, view is the mechanism that defines what data is to be presented to the user and template dictates how data will be presented to the user via HTML. 

Today I'm going to focus on what model relationships are in Django, but first, a little introduction on database. The Django framework is designed to work with relational database engines in the back end such as [SQLite](https://sqlite.org/index.html), [PostgreSQL](https://www.postgresql.org/), [MySQL](https://www.mysql.com/) and [Oracle](https://www.oracle.com/index.html). However, support for non-relational database such as [MongoDb](https://www.mongodb.com/) is also possible through third-party connectors such as [Djongo](https://www.djongomapper.com/integrating-django-with-mongodb/). From a relational database perspective, data is organized as tables that may be related to one another in one of these associations: one-to-one, one-to-many, many-to-many.

## One-to-One Relationship

Examples of a one-to-one relationship are plenty in the real world. Consider these scenarios:
1. A user profile in a web application is  associated with a unique email address. Some web applications, such as PayPal and LinkedIn, allow a user to have multiple email addresses, but many, such as social media, banking and online shopping, restrict an email address to only one user.

2. A citizen of a country is associated with a passport. Not every citizen requires a passport but if they want to travel outside the country, they must be issued a passport. 

3. An ISBN number is associated with a book, but a book does not have to acquire an ISBN number in order for it to be published. 

In general, to implement a table that is related to another table, a [_FOREIGN KEY_](https://docs.microsoft.com/en-us/previous-versions/sql/sql-server-2008-r2/ms175464(v=sql.105)) designation is assigned to a column to bind the two tables together. In SQL, to implement a one-to-one relationship between two tables, another designation, [_UNIQUE_](https://www.relationaldbdesign.com/database-design/module6/one-to-oneRelationships.php), is usually added to the foreign key column as well. 

In Django, models are translated to database tables via SQL. A model has fields and field types whereas a table has columns and column types. A field or column type tells us how data should be stored, for example, as a string, integer, float, boolean, etc. To implement a one-to-one relationship between models, Django lets us create a special field type, `OneToOneField`, that typically resides in one of the two models. 

To decide which one of the two models should have this OneToOneField, consider this example with two Django models, `Book` and `Isbn`. If a `Book` can exist without an `Isbn`, but an `Isbn` must always be linked to a book, then place the `OneToOneField` in the `Isbn` model.

```py
from django.db import models

class Book(models.Model):
  title = models.CharField(max_length=20)
  ...
  
class Isbn(models.Model):
  isbn = models.CharField(max_length=13)
  book = models.OneToOneField(to=Book, on_delete=models.CASCADE)
  ...
```

In the code example above, we place the `OneToOneField` in the `Isbn` model assigned to the field named, `book`.  Let's investigate the next type of relationship.

## One-to-Many Relationship

Like one-to-one, real world objects may be linked with one another via a one-to-many or many-to-one relationship. Consider these scenarios:
1. A person may have one or more social media accounts, but a social media account may only be linked to one person.
2. A country may have many ambassadors stationed in different countries, but each ambassador may only represent one country.
3. An author may write many books, but each book may only be associated with one author.

We would typically represent a one-to-many relationship in a relational database schema using the FOREIGN KEY designation as well. One would expect Django to provide a `OneToManyField` field type to correspond to the one-to-many relationship.  However, this is not so. Instead, Django provides a `ForeignKey` field which raises some confusion among developers. 

A question that often comes up is 'Where does one place the `ForeignKey` field type?' Suppose we have two Django models, `Book` and `Author` and they are related by a many-to-one relationship. I use one of these two criteria to help me decide where to place the `ForeignKey` field type. 
1. Since a `Book` can only exist because it has an `Author` but an `Author` can exist even without any `Book` to their credit, we should place the `ForeignKey` field type in `Author`.
2. Since a `Book` has a many-to-one relationship with `Author`, we should place the `ForeignKey` field type in the model that has the "many" relationship, which is `Book`.

In both cases, `Book` is the host of the `ForeignKey` field type and it should be assigned to a field such as `author`.

```py
from django.db import models

# Author has a one-to-many relationship with Book
class Author(models.Model):
  name = models.CharField(max_length=20)
  ...
  
# Book has a many-to-one relationship with Author
class Book(models.Model):
  title = models.CharField(max_length=50)
  author = models.ForeignKey(to=Author, on_delete=models.CASCADE)
  ...
```
In the code example above, the `author` field houses the `ForeignKey` field type to show that a `Book` must be associated with an `Author`.  Let's explore the last type of relationship.

## Many-to-Many Relationship

Examples of many-to-many relationship abound in the real world as well. Consider these scenarios:
1. A customer may subscribe to many publications, while a publication may have many subscribers.
2. A person may join many clubs, while a club may have many members.
3. A recipe may have many ingredients, while the same ingredient may belong to many recipes.

In a relational database schema, we typically represent a many-to-many relationship by creating a [_JOIN_](https://entityframework.net/many-to-many-relationship#configure-many-to-many-relationship) table to house two columns that are designated as foreign keys which are themselves primary keys in the join table. This gives rise to the concept of multiple primary keys in a table, also known as [_COMPOSITE PRIMARY KEY_](https://www.relationaldbdesign.com/database-analysis/module2/composite-primary-keys.php). 

From the Django perspective, implementing a composite primary key is not possible due to [issues](https://code.djangoproject.com/wiki/MultipleColumnPrimaryKeys). A [ticket](http://code.djangoproject.com/ticket/373) that requested this feature was opened back in 2005 and is still waiting to be resolved at the time of writing. But not to worry, as Django supports many-to-many relationship between models via a `ManyToManyField` field type.

If two Django models are associated by a many-to-many relationship, we can place the `ManyToManyField` in any of the two models. 

```py
from django.db import models

class Ingredient(models.Model):
  name = models.CharField(max_length=20)
  ...
  
class Recipe(models.Model):
  name = models.CharField(max_length=20)
  ingredients = models.ManyToManyField()
  ...
```

From the code example above, both `Ingredient` and `Recipe` are related by having a many-to-many relationship. The `ManyToManyField` field type is placed in `Recipe` assigned to `ingredients`, but it could also be placed in `Ingredient` and assigned to a field, such as `recipes`.

## Conclusion

The Django framework provides solutions to implement the classic relationships between objects via the `OneToOneField`, `ForeignKey` and `ManyToManyField`. The choice to use the name `ForeignKey` instead of `OneToManyField` or `ManyToOneField` often raises confusion as it breaks the convention of the other two related field types. This article hopefully dispels the confusion among new Django developers.
