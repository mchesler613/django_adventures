# Adventures in Django: Leveraging OO in Models and Views

In my previous article on [Model Relationships](https://github.com/mchesler613/django_adventures/blob/main/model_relationships.md), I described how Django implements one-to-one, one-to-many and many-to-many model relationships using the `OneToOneField`, `ForeignKey` and `ManyToManyField`. Today I'm going to document how to leverage the power of object-oriented programming when writing Django models and views.

In a Django app that plans an employee's tasks and meetings, much like a calendar app, we would like to be able to:
+ provide a directory listing of employees
+ schedule tasks for an employee
+ schedule meetings for employees
+ discover the pending tasks and meetings for the current day when an employee logs in

I'm going to highlight the five Django models that are involved in this app -- `Person`, `Bio`, `Email`, `Task` and `Meeting`. Basically, a `Person` has a one-to-one relationship with `Bio` and `Email`, one-to-many relationship with `Task` and many-to-many relationship with `Meeting`. 

![Model Relationships](https://i.postimg.cc/QCyHRWHh/Model-Relationships.png)

## MTV Revisited
Let's revisit the MTV (Model-Template-View) paradigm that Django is based on. Recall that the model is the central component encapsulating data and behavior, the view defines which data from the model is to be presented to the user and the template defines how data from the model is presented to the user.

![Model-Template-View](https://i.postimg.cc/3NNSyhtn/Model-Template-View.png)

As you can see from the illustration, there is a pair of tightly-coupled entities: model-view and template-view. The model and view interact with each other; so do the template and view. But the template does not communicate with the model at all as we separate an application's user interface from the application logic. The Django view acts as an intermediary between the model and the template instead. 

So, if we want to present an HTML page, **person_detail.html**, with information compiled from `Person`, `Email`, `Bio`, `Task` and `Meeting`, we would need a Django view, such as `PersonDetailView`, to collect all this information for us and pass it to the Django template described in **person_detail.html**. 

![Django Template](https://i.postimg.cc/X73X0vDf/Person-Detail-Page-2021-02-17-17-37-37.jpg)

There is a lot of information presented in the Django template. Let's examine each one.
+ The name, `Raya`, is saved in a `Person` model instance. 
+ The email, `raya@djangoschool.com` is saved in an `Email` model instance.
+ The image and the text, `Project Manager Extraordinaire`, are saved in a `Bio` model instance.
+ The `Tasks Due Today` information is saved in a `Task` model instance.
+ The `Today's Meetings` information is saved in a `Meeting` model instance.

Let's explore what the Django view does to compile all this information.

## Django View

Django provides two ways to implement views -- function or class. In this app, the Django view was implemented using a class, `PersonDetailView`, that derives from Django's class-based generic view, [`DetailView`](https://docs.djangoproject.com/en/3.1/topics/class-based-views/generic-display/). Django provides generic views such as `DetailView` and `ListView` because they are common views for present detailed information or a directory listing. The main benefits of inheriting a class-based view is to reuse code whenever possible and focus on the specific needs of our application.

Earlier, I mentioned that the view and the template are tightly coupled. This means that the template does not interact with the model at all and uses whatever information it needs to display from the view itself. This information is captured in a Python dictionary of key-value pairs returned by the generic view's method, [`.get_context_data()`](https://docs.djangoproject.com/en/3.1/ref/class-based-views/mixins-single-object/#django.views.generic.detail.SingleObjectMixin.get_context_data), also known as the `context`. A sample `context` may have the following information:
```py
context	
{
 'object': <Person: Raya>,
 'person': <Person: Raya>,
 'view': <planner.views.PersonDetailView object at 0x0000000004A6B970>
}
```
where the `object` and `person` both represent an instance of the model `Person` and the `view` is the instance of `PersonDetailView` inherited from `DetailView`. A sample implementation of `PersonDetailView` may look like this:

```py
class PersonDetailView(DetailView):
  template_name = "planner/person_detail.html"  # This tells Django which template to use 
  model = Person                                # This tells Django which model to use
```

This simple implementation would be enough if all one needs to present is the details of the `Person` model instance. However, we can extend the `context` by adding more key-value pairs. To do this, we need to override the default implementation of `.get_context_data()`. In this demo app, the **person_detail.html** template has additional information about tasks and meetings. So, here's my implementation of `PersonDetailView`.
```py
class PersonDetailView(DetailView):
  template_name = "planner/person_detail.html"  # This tells Django which template to use 
  model = Person                                # This tells Django which model to use

  def get_context_data(self, **kwargs):         # override the default implementation
    context = super().get_context_data(**kwargs)
    person = Person.objects.get(pk=self.kwargs['pk'])
    context['meetings_today'] = person.meetings_today()
    context['tasks_due_today'] = person.tasks_due_today()
    # assert False          # Uncomment this line to display the values of `context` on the browser
    return context
```
Notice that I added two keys, `meetings_today` and `tasks_due_today`, to `context` by assigning them the return values of `person.meetings_today()` and `person.tasks_due_today()`. These are custom methods that exhibit specific behavior of the `Person` model in addition to many others. They basically return a Django `QuerySet` of model instances that meet specific conditions. 

Here is the implementation of these methods:
```py
class Person(models.Model):
  ...
  # return all tasks due today
  def tasks_due_today(self):
    return self.task_set.filter(deadline=date.today())

  # return all meetings scheduled for today
  def meetings_today(self):
    return self.meeting_set.filter(participants__name=self.name).filter(date=date.today())
```
By encapsulating calls to the [Django Model QuerySet API](https://docs.djangoproject.com/en/3.1/ref/models/querysets/) within the model as much as possible, we promote modularity and code reuse. Remember that adding methods to a Django model does not require any database migration as doing so doesn't affect the database schema at all.  

Notice also the statement, `assert False`, that is commented in the code sample above. This statement is useful for debugging and will display traceback information on your browser even if there is no error. You can obtain the `context` information that is passed from the view to the template in the traceback. 

For example:
```py
context	
{'meetings_today': <QuerySet [<Meeting: Manager Sync, Meeting for Managers Only, Wed, 2021-02-17, 09:30:00 with <QuerySet [<Person: Raya>, <Person: Kim>]>>, <Meeting: Onboarding, Onboarding with Raya and Kenya, Wed, 2021-02-17, 17:15:10 with <QuerySet [<Person: Raya>, <Person: Kenya>]>>]>,
 'object': <Person: Raya>,
 'person': <Person: Raya>,
 'tasks_due_today': <QuerySet []>,
 'view': <planner.views.PersonDetailView object at 0x0000000004A6B970>}
```
Notice that `meetings_today` and `tasks_due_today` have been added to the `context`.

## Conclusion
We can apply useful object-oriented techniques such as adding custom methods when implementing our Django models and inheriting our views from Django's class-based generic views. By leverage our knowledge of object-oriented design and programming, we can save a lot of precious time from reusing and reducing the code we write and simplify our code maintenance. If you have benefited from this article, kindly give positive feedback and share with others too.  Thank you for reading!
