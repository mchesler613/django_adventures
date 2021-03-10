# Adventures in Django: Django Mail with FormView and Simple Captcha
![Cover Image](https://i.postimg.cc/dtx47yXH/Django-Mail-Form-View-Simple-Captcha.png)
It's common in most websites to allow visitors to interact with them through an [HTML Form](https://developer.mozilla.org/en-US/docs/Learn/Forms). To distinguish a human visitor from a machine or bot, a [captcha](https://en.wikipedia.org/wiki/CAPTCHA) challenge is typically included in the form as well. Some of these forms facilitate communication between a visitor and the website. Submitting one of these forms usually trigger an email to be sent to a specific address. Django provides an easy-to-use `send_mail()` API to send data over the [`SMTP`](https://en.wikipedia.org/wiki/Simple_Mail_Transfer_Protocol) protocol. In this article, I'm going to show how to create an HTML Form using Django's `FormView` class with [_Simple Captcha_](https://django-simple-captcha.readthedocs.io/en/latest/usage.html#) and the `send_mail()` API. Let's get started.

## Install Simple Captcha

Simple Captcha, like the name suggests, is really simple to use. Follow the [instructions](https://django-simple-captcha.readthedocs.io/en/latest/usage.html#installation) to accomplish the following:
+ Install this package via `pip install django-simple-captcha`
+ Add `captcha` to `INSTALLED_APPS` in **settings.py**, for example:
  ```py
  INSTALLED_APPS = [
      'captcha',
      ...
  ]
  ```
+ Run a migration via `python manage.py migrate`
+ Add an entry in the project's **urls.py**, for example:
  ```py
  urlpatterns += [
    path('captcha/', include('captcha.urls')),
  ]
  ``` 

You can also configure the type of challenge for the captcha by overriding the `CAPTCHA_CHALLENGE_FUNCT` setting in the project's **settings.py**. The default is a four-letter word challenge. 

## Create a Django Form

In this demo, we want to create a common contact form for a website. In **forms.py**, create a Django class, `ContactForm`, to represent this contact form. It should inherit from Django's `Form` class. For example:
```py
from django import forms
from captcha.fields import CaptchaField  # add a Simple Captcha field

class ContactForm(forms.Form):
  name = forms.CharField()
  email = forms.EmailField()
  subject = forms.CharField()
  message = forms.CharField(widget=forms.Textarea)
  captcha = CaptchaField()
```
Notice that we added a `captcha` field from Simple Captcha's `captcha.fields` module. Here is a sample contact form.

![Blank Contact Form](https://i.postimg.cc/85TP2pgL/Blank-Contact-Form-2021-03-10-1-01-05.jpg)

## Add a Django View for our Contact Form

Next, we create a corresponding view, `ContactView`, for our Django form, `ContactForm`, that inherits from [`FormView`](https://docs.djangoproject.com/en/3.1/ref/class-based-views/generic-editing/#formview). `FormView` is a view that specifically displays a form, validates it, displays any validation error and redirects to a url upon a successful validation.

To display a form, we need to specify the template name for it, such as **contact.html** in the view's `template_name` variable. We also need to bind the `ContactForm` to the view in the `form_class` variable. Lastly, we need to specify a url to redirect to upon successful validation, such as `/thanks/` in the `success_url` variable.  For example:

```py
from django.views.generic.edit import FormView
from mail.forms import ContactForm

class ContactView(FormView):
  template_name = 'contact.html'
  form_class = ContactForm
  success_url = '/thanks/'
```

## Add Custom Behavior to our Contact View

The `ContactView` is not yet fully-functioning unless we add custom behavior to it. We need to override two inherited methods:
+ `.is_valid()`, which is called to validate a form 
+ `.form_valid()`, which is called when valid form data has been posted

Recall that we have included a captcha field in our form. We need to know how to evaluate this field as part of our form validation. Simple Captcha has made it really easy for us. All we have to do is to set a variable, `human`, to be `True` if the form is valid. So, our code for `.is_valid()` will look like this.

```py
class ContactView(FormView):
  ...
  # This method will validate the form with Simple Captcha
  def is_valid(self, form):
    if form.is_valid():
      human = True    # set for Simple Captcha
```

Next, we need to figure out what to do when our form has been successfully validated. There are basically three tasks we need to accomplish:
+ Send off an email based on our validated form data
+ Respond to our visitor with an acknowledgement of thanks and a success or fail message
+ Return an Http response

In our view, we would like to personalize a thank-you message with the outcome of `send_mail()`. `send_mail()` returns `1` if it is successful and `0` if not. So, we would like to pass some data (the web visitor name and the `send_mail()` outcome) to the template so that the template can personalize the message. We therefore define the named URL pattern for the success url, `/thanks/`, in **urls.py** as follows:

```py
# project urls.py
from mail.views import ThanksView

urlpatterns = [
    path('thanks/<str:visitor>/<int:success>', ThanksView.as_view(), name='thanks'),
    ...
    ]
```
Notice that we also need another view, `ThanksView`, to process the data it receives when the success url is invoked. But let's focus on the `.form_valid()` method for now, which will look like this.
```py
  def form_valid(self, form):
    response = form.send_email()

    # Redefine 'success_url' to pass data to the 'thanks' template
    self.success_url = reverse('thanks', kwargs={'visitor':form.cleaned_data['name'], 'success':response})

    # Return a Http Response
    return super().form_valid(form)
```
Notice that we call the `.send_email()` method for our `ContactForm` which hasn't yet been defined. Since our `ContactForm` contains all the fields in the form, it makes sense to delegate the responsibility of sending email to the form instead of the view. `form.send_email()` will return a value of `0` (fail) or `1` (success) which is saved in `response`.

Next we need to redefine the `success_url` because it is expecting two keyword arguments in the url pattern, `thanks/<str:sender>/<int:success>`. We call the [`reverse()`](https://docs.djangoproject.com/en/3.1/ref/urlresolvers/#django.urls.reverse) function with the named url pattern, `thanks`, and the keyword argument dictionary, `kwargs` with key-value pairs for `visitor` and `success`. When a form has been validated, its field names and values are available in [`cleaned_data`](https://docs.djangoproject.com/en/3.1/ref/forms/api/#django.forms.Form.cleaned_data), a Python dictionary of key-value pairs. So we assign the `visitor` key the `cleaned_data` value for the `name` form field and the `success` key the value of `response`.

Lastly, we return a Http response by calling the parent's `.form_valid()` method. Before we write the `.send_email()` method for our `ContactForm`, we will first tackle understanding the Django's `send_mail()` API.

## Django send_mail() API

Let's learn more about the Django's [`send_mail()`](https://docs.djangoproject.com/en/3.1/topics/email/#send-mail) API whose signature is as follows: 
```py
send_mail(
  subject,         # This is the subject of the email
  message,         # This is the body of the email
  from_email,      # this is the sender of the email. If value is None, Django uses value from settings.DEFAULT_FROM_EMAIL whose default is 'webmaster@localhost'
  recipient_list,  # this is a list of receipient addresses
  fail_silently=False, # Django will raise an error when there is a problem
  auth_user=None,  # This is email account user. If not provided, Django uses settings.EMAIL_HOST_USER
  auth_password=None, # This is the password of the email account. If not provided, Django uses settings.EMAIL_HOST_PASSWORD
  connection=None, # This is the email backend. If not provided, Django uses settings.EMAIL_BACKEND
  html_message=None # This is the HTML formatted message
)
```
Notice that there are configuration variables that Django expects to be defined in **settings.py** such as `EMAIL_HOST_USER` and `EMAIL_HOST_PASSWORD`. But the complete listing is as follows:
+ host: EMAIL_HOST for example, `smtp.yourdomain.com`
+ port: EMAIL_PORT for example, (TLS): `587` or (SSL): `465`
+ username: EMAIL_HOST_USER, for example, `you@yourdomain.com`
+ password: EMAIL_HOST_PASSWORD, for example, your `you@yourdomain.com` account password
+ use_tls: EMAIL_USE_TLS, `True` or `False`
+ use_ssl: EMAIL_USE_SSL, `True` or `False`
+ timeout: [EMAIL_TIMEOUT](https://docs.djangoproject.com/en/3.1/ref/settings/#std:setting-EMAIL_TIMEOUT)
+ ssl_keyfile: [EMAIL_SSL_KEYFILE](https://docs.djangoproject.com/en/3.1/ref/settings/#std:setting-EMAIL_SSL_KEYFILE)
+ ssl_certfile: [EMAIL_SSL_CERTFILE](https://docs.djangoproject.com/en/3.1/ref/settings/#std:setting-EMAIL_SSL_CERTFILE)

The `EMAIL_HOST` has to be a legitimate SMTP server with a valid email account capable of sending and receiving email messages. If you own a shared hosting account on one of the popular [web hosting providers](https://en.wikipedia.org/wiki/Web_hosting_service), you will be able to purchase a domain, create an email account and utilize your domain's SMTP service. 

Please note that you shouldn't store sensitive information inside **settings.py** such as passwords. One option is to export sensitive data as environment variables and retrieve them in **settings.py**. For example, to export an environment variable, `EMAIL_HOST_PASSWORD`, type in the terminal:
```
$ export EMAIL_HOST_PASSWORD=your_secret_password
```
To retrieve `EMAIL_HOST_PASSWORD` in **settings.py** use `os.getenv()`. For example:
```py
# settings.py
import os
EMAIL_HOST_PASSWORD = os.getenv('EMAIL_HOST_PASSWORD')
```
**Remember to restart the Django server after you have exported your environment variables otherwise the variables will be undefined!**

## Add a send_email() Method to the Form

We can add behavior to our `ContactForm` by allowing it to send email when a form has been submitted and validated. We can add a `send_email()` method that accomplishes the following:
+ Configure the subject of the mail.
+ Read the values from the validated form fields in `.cleaned_data` and save them as variables.
+ Configure the body of the mail based on the saved cleaned data.
+ Configure the sender of the email. If we use Django's `send_mail()`, the sender should be set to the value found in `settings.EMAIL_HOST_USER`.
+ Configure the receipient(s) of the email. This can be a single receipient or a list.
+ Lastly, we call Django's `send_mail()` API with our subject, body, sender, and receipient list.

For example:
```py
from django.core.mail import send_mail
from djangomail import settings

class ContactForm(forms.Form):
  ...
  # Call this method after the form is validated
  def send_email(self):
    origin = settings.SITE_URL + reverse('contact')
    subject = '[ContactForm] from %s' %origin
    name = self.cleaned_data['name']
    email = self.cleaned_data['email']
    message = self.cleaned_data['message']
    body = """
    Name: %s
    Email: %s
    Content: %s
    """ %(name, email, message)
    sender = settings.EMAIL_HOST_USER
    receipient = 'you@example.com'

    response = 0

    try:
      response = send_mail(
        subject,
        body,
        sender,
        [receipient],
        fail_silently=False,
        #auth_user='no_such_user',  # uncomment this line to test for a failed send_mail()
      )
    except smtplib.SMTPException:
      pass

    return response
```
Notice that we've provided an `origin` variable that is assigned the website url, `SITE_URL`, defined in our project **settings.py** file concatenated with the `contact` named URL pattern defined in the project's **urls.py**. We use `origin` to format the `subject` of the mail message. 
We assume that `.send_mail()` will be called after a form is validated, so that we can have access to the `.cleaned_data` dictionary to format the `body` of our message with `name`, `email` and `message`. Django expects the receipient argument in Django's `send_mail()` to be a list. 

When we set `fail_silently` to `False`, Django will raise an [`smtplib.SMTPException`](https://docs.python.org/3/library/smtplib.html#module-smtplib) exception when there is an error. We place the call to Django's `send_mail()` inside a `try-except` block because we want to gracefully handle any error that is raised. We return the `response` value that is either set to `1` if `send_mail()` is successful or remains `0` if not. 

## Add a Django View for the Success URL

Recall that the `success_url` for our ContactView is handled by another view, `ThanksView`, which inherits from `TemplateView`. This view will have a `template_name` variable whose value is `thanks.html`. `ThanksView` needs to pass data to the template, so it will have a method, `get()` or `get_context_data()`. For example:
```py
from django.views.generic.base import TemplateView

class ThanksView(TemplateView):
  template_name = 'thanks.html'

  def get(self, request, *args, **kwargs):
    context = super().get_context_data(**kwargs)
    context['success'] = self.kwargs['success']
    context['visitor'] = self.kwargs['visitor']

    return render(request, self.template_name, context)
```
When the view receives an `Http GET` request through the `thanks/<str:visitor>/<int:success>` URL pattern, the `get()` method will be invoked. It retrieves the values for `visitor` and `success` in the `kwargs` keyword argument and assigns them to the respective keys in the `context` dictionary to be passed to the `thanks.html` template.

## Demo Screenshots

So, we've reached the end of our article. Let's show you a few screenshots of our implementation of a contact form. Here is a completed contact form.
![Completed Contact Form](https://i.postimg.cc/9063jbws/Completed-Form-2021-03-10-1-02-44.jpg)

Here is a successful form submission and response.
![Success Send Mail](https://i.postimg.cc/SsQkZgcr/Email-Sent-2021-03-09-20-07-24.jpg)

Here is a response from a failed email transmission.
![Failed Send Mail](https://i.postimg.cc/cLQW0pmh/Email-Not-Sent-2021-03-09-20-05-07.jpg)

## Conclusion

Developing our contact page with Django using the `FormView` is not as straightforward as we would like. Here are some of our challenges:
+ We need to figure out how to retrieve validated form data. Since this data is only available as `.cleaned_data` when a form is successfully validated via `.form_valid()`, we need to overload this method. 
+ We need to determine how to pass some of the cleaned data to the `thanks.html` template to form a personalized message. By formatting the `thanks` URL pattern to take arguments, we can retrieve these arguments in our `TemplateView` and store them in a context dictionary. 
+ We need to decide where to place the logic of sending email. Should it be the responsibility of the Django view or the Django form? Since the form is knowledgable about its fields and values, it is only natural that the form be the one to format and send email messages through Django's `send_mail()` function. 

Another challenge we have is saving sensitive information like passwords to be utilized by the `send_mail()` API. One solution is to utilize environment variables, but there are other alternatives as well that we can explore sometime in the future. However, the easiest part of this development is deploying Simple Captcha, true to its name! I hope you have found this lengthy tutorial useful enough to continue to Django!
