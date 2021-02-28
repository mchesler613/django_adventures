# Adventures in Django: Displaying and Editing Multiple ModelForms in View and Template

In my last article, [Leveraging OO in Models and Views](https://merilynchesler.medium.com/adventures-in-django-leveraging-oo-in-models-and-views-c95bc7e4f37c), I described how we can maximize object-oriented programming in implementing custom behavior in our Django models to hide [model-specific QuerySet API](https://docs.djangoproject.com/en/3.1/ref/models/querysets/) and to take advantage of Django's class-based generic views, such as [`DetailView`](https://docs.djangoproject.com/en/3.1/topics/class-based-views/generic-display/) to reuse code and minimize development time. 

Recall that in Django's MTV (Model-Template-View) architecture, the model is the central component encapsulating data and behavior, the view defines _which_ data from the model will be presented to the user and the template takes over _how_ the data from the view will be presented in HTML. A typical application not only presents read-only data, but also lets a user create new data and update existing data. Creating and updating information is typically done in an [HTML Form](https://developer.mozilla.org/en-US/docs/Learn/Forms) which is encapsulated inside the `<form>...</form>` tag. Django facilitates data collection in a [`Form`](https://docs.djangoproject.com/en/3.1/topics/forms/#the-django-form-class) class. In this article, I'm going to document:
+ how to leverage Django's [`ModelForm`](https://docs.djangoproject.com/en/3.1/topics/forms/modelforms/#django.forms.ModelForm) class which maps a model's fields to an HTML `Form`'s input elements in our demo application
+ how to display multiple `ModelForm`s in a view and template
+ how to edit multiple `ModelForm`s in a view and template
We have a lot to cover, so, let's get started.

This application is a sample Django app that lets a user plans tasks and meetings. For this article, we are focusing only on three models, `Person`, `Bio`, and `Email`. `Person` has a one-to-one relationship with `Bio` and `Email`. 
<script src="https://gist.github.com/mchesler613/d7a15eab07f15015249bb0298903bb39.js"></script>
