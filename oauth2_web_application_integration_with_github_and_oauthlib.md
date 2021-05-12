# Integrating OAuth2 in Django with GitHub and OAuthlib

![OAuth2 Web Application Flow](https://i.postimg.cc/9Ffkq1Bm/OAuth2-Web-Application-Integration-with-Git-Hub-and-OAuthlib-by-Merilyn-Chesler.png)

## Introduction

If you have several social media accounts, such as Google, Twitter, Facebook, and GitHub, and would prefer to use one of these accounts to log into websites that support third-party logins, you should read this article. The ability to seamlessly log in is thanks to [OAuth2](https://oauth.net/),  a secure open protocol for authorization and access to protected third-party account data. 

The OAuth2 protocol comes with various flows to handle different types of applications:

+ web application

+ single page application

+ mobile application

+ others

This article focuses on a Django web application that integrates with [GitHub](http://github.com) as the third-party OAuth2 provider.

## Objectives
Several Django extensions let you integrate third-party logins into your application via OAuth2. With these extensions, you only need to write minimal code to use them. However, if you are adventurous like me and want to learn how to implement an OAuth2 flow in your Django app, grab a cup of your favorite brew and have a seat.  By the time you finish reading this article, you might be able to:

+ explain what OAuth2 is

+ explain the OAuth2 flow for a web application

+ learn how to integrate OAuth2 into your Django app 


## GitHub Requirements
To integrate with GitHub, you will need:

+ a GitHub account

+ a GitHub [Developer OAuth app](https://github.com/settings/developers) account 

	+ GitHub provides you with a _client id_ and _client secret_
	+ You provide GitHub with a HTTPS-callable URI endpoint, also known as _redirect uri_ or callback URL.

 

## Django Requirements
To fully integrate with GitHub's OAuth App, you will need to run a secure Django app with SSL.

The extensions I will be using in this article are:

+ [oauthlib](https://github.com/oauthlib/oauthlib), a Python framework which implements both OAuth1 or OAuth2 logic rather intuitively. Many Django packages that support third-party OAuth2 logins are built on top of this framework.
+ [requests](https://github.com/psf/requests), a Python HTTP library that lets you send HTTP/1.1 requests rather intuitvely. For a wondeful tutorial to get you going, I recommend [W3Schools](https://www.w3schools.com/PYTHON/module_requests.asp).


## OAUth2 Web Application Flow

![OAuth2 Web Application Flow](https://i.postimg.cc/25VJ8K3Z/Oauth2-Web-Application-Flow-Merilyn-Chesler.png)

In a nutshell, your Django app serves as a client while GitHub serves as a server. The end goal is for our Django client to retrieve a publicly available GitHub user profile after successfully authenticating the user. The OAuth2 Web Application flow consists of three steps:

### 1. Authorize GitHub to Access your Profile

Authorize the developer OAuth app that we registered at GitHub to begin the process of authentication. Access GitHub's authorization URL, located at `'https://github.com/login/oauth/authorize'`, with the following parameters:

+ `client_id`, the credential GitHub supplied us
+ `redirect_uri`, a callback URL at our site upon successful authorization from GitHub
+ `scope`, the scope of the profile data requested from GitHub including user, [gists](https://gist.github.com/) and repositories. If we don't supply it, we get everything.
+ `allow_signup`, true by default, GitHub will allow a new account registration during authorization.
+ `state`, a randomly-generated string to pass to GitHub during authorization and to verify during callback

The authorization process may look like this:
![Authorization Prompt](https://aaronparecki.com/oauth-2-simplified/oauth-authorization-prompt.png) 

### 2. Exchange Authorization Code for an Access Token

When our Django client app receives a callback from GitHub after a successful authorization, we will receive an authorization `code` and a `state` information. It is our responsibility to verify that the `state` information is an identical copy of the one we supplied in the previous step. Doing this will check for any malicious attempts from hackers also known as [CSRF](https://docs.djangoproject.com/en/1.11/ref/csrf/) or Cross Site Request Forgery.  

With these parameters:

+ authorization `code`
+ `client_id`
+ `client_secret`
+ `redirect_uri`

we can then fetch an `access_token` from GitHub. 

### 3. Fetch GitHub Profile with Access Token

With the `access_token`, and an authorization header, we can request GitHub for the authorized GitHub profile based on the `scope` we specified in step 1.

## Django App Flow

After retrieving the GitHub user profile, we can do the following in our Django app.

### 1. Create a Django User

We can integrate the GitHub user profile data with a Django user account. We can write additional logic to create a Django User if it doesn't exist or reuse an existing User object and go from there.

![GitHub to Django User](https://i.postimg.cc/dt9F3zHL/Git-Hub-to-Django-User.png)

### 2. Display a Welcome page

This is a sample page of our app to welcome the Django User that we either retrieved from our database or just created.

![Welcome Page](https://i.postimg.cc/bwGysmKb/2021-04-25-16-08-21.jpg)

### 3. Display Secret Content

This is a sample page of our app that contains content that only an authenticated Django user can view.

![Secret Page](https://i.postimg.cc/GhSRSN8R/2021-04-21-14-12-03.jpg)

## Integration with OAuthLib and Requests

To integrate our Django app with `oauthlib`, we need to install the package.

```
$ pip install oauthlib
```

To use the `requests` package, we need to install it with:

```
$ pip install requests
```

## Django Views
We need to write Django views to:

+ authorize GitHub to log in as us (see step 1), e.g. `github_login()`
+ process the callback from GitHub after a successful authorization, e.g. `CallbackView.as_view()`
+ display a welcome message after a successful authentication, e.g. `WelcomeView.as_view()`
+ display secret content for authenticated users on a secret page, e.g. `PageView.as_view()`
+ log us out from the Django app, e.g. `logout_request()`

Corresponding to the above views, we need to configure the URL endpoints in our app's **urls.py**:

```py
urlpatterns = [
  path('login/', views.github_login, name='login'),  
  path('callback/', CallbackView.as_view(), name='callback'), 
  path('welcome/', WelcomeView.as_view(), name='welcome'),    		
  path('page/', PageView.as_view(), name='page'),
  path('logout/', views.logout_request, name='logout'),
  ...
]
```
Let's dive into each of the above-mentioned views.

## Login View

Our login view is represented by the function, `github_login(request)` which basically sets up a GitHub authorization URL with [parameters](https://docs.github.com/en/developers/apps/authorizing-oauth-apps#1-request-a-users-github-identity) to redirect to. The components of this URL are:

+ GitHub's authorization URL, `https://github.com/login/oauth/authorize`
+ `client_id`
+ `state`
+ `redirect_uri`
+ `scope`
+ `allow_signup`

Here is a walkthrough for each step. We will define a `github_login(request)` function-based view like this:

```py
def github_login(request):
```

+ We retrieve the `client_id` from the project's **settings.py** file. 

```py
   client_id = settings.GITHUB_OAUTH_CLIENT_ID
```

+ We create a Web Application client from `oauthlib` with the `client_id`.

```py
   client = WebApplicationClient(client_id)
```
+ To generate the `state` data, we use the function `token_urlsafe(16)` from a standard Python [`secrets`](https://docs.python.org/3/library/secrets.html) package. Since we will be retrieving the `state` information later in the callback view, we will store the `state` data inside a Django [session](https://docs.djangoproject.com/en/3.2/topics/http/sessions/). The Django Sessions framework allows us to store temporary data on the server side and is very useful for us in this scenario. 

```py
   request.session['state'] = secrets.token_urlsafe(16)
```
+ We will call [`client.prepare_request_uri()`](https://oauthlib.readthedocs.io/en/latest/oauth2/clients/webapplicationclient.html?highlight=prepare_request_body(#oauthlib.oauth2.WebApplicationClient.parse_request_uri_response)) method to prepare a complete URL to redirect using the `authorization_url`, `redirect_uri`, `scope`, `state` and `allow_signup` variables. In this example, we limit the `scope` to `read:user` (read-only public user profile) and disable new GitHub account registrations by setting `allow_signup` to `'false'`. We will save the return value of this call in `url`. 

```py
   authorization_url = 'https://github.com/login/oauth/authorize'

   url = client.prepare_request_uri(
      authorization_url,
      redirect_uri = settings.GITHUB_OAUTH_CALLBACK_URL,
      scope = ['read:user'],
      state = request.session['state'],
      allow_signup = 'false'
)

```
+ If we print `url`, we could expect something in this format:

```
https://github.com/login/oauth/authorize?response_type=code&client_id=xxxxxxxx&redirect_uri=https://example.com/callback&scope=read:user&state=D8VAo311AAl_49LAtM51HA&allow_signup=false
```

+ Lastly, we will pass `url` as an argument to the [`HttpResponseRedirect()`](https://docs.djangoproject.com/en/3.2/ref/request-response/#django.http.HttpResponseRedirect) Django function which we will call before returning from the view. 

```py
    return HttpResponseRedirect(url)
```
If all goes well, we would expect a callback from GitHub's authorization API. Let's go to the next view, the callback view.

## Callback View
In this view, we will be doing several exciting things, among them:

+ retrieving an authorization code from GitHub
+ exchanging this code with an access token from GitHub
+ retrieving user profile data from GitHub
+ creating a Django user or retrieving an existing user
+ logging the user in
+ redirecting to a welcome view

Let's get started. 

+ We will define our callback view as a subclass of `TemplateView`, a class-based generic view. 

```py
class CallbackView(TemplateView):
```

+ We will define a `get()` method to implement the callback logic.

```py
def get(self, request, *args, **kwargs):
```

+ We will retrieve the data that is sent back to us by [GitHub OAuth API](https://docs.github.com/en/developers/apps/authorizing-oauth-apps#2-users-are-redirected-back-to-your-site-by-github), mainly the authorization `code` and the `state`. GitHub says that this temporary code will expire in ten minutes.

```py
    # Retrieve these data from the request URL
    data = self.request.GET
    code = data['code']
    state = data['state']
```

For security purposes, we should verify that the `state` content is the same as the one we issued in `github_login()`. Since we saved the `state` data in a Django session previously, we can retrieve it and compare it with what we just received and act accordingly. I used the [Django Messages Framework](https://docs.djangoproject.com/en/3.2/ref/contrib/messages/) to store useful information that will be displayed to the end user in the Django template.

```py
    if state != self.request.session['state']:
      messages.add_message(
        self.request,
        messages.ERROR,
        "State information mismatch!"
      )
      return HttpResponseRedirect(reverse('github:welcome'))
    else:
      del self.request.session['state']
```

+ Next, we want to exchange the authorization `code` with an access token from GitHub. We will get these variables ready:

```py
   token_url = 'https://github.com/login/oauth/access_token'
   client_id = settings.GITHUB_OAUTH_CLIENT_ID
   client_secret = settings.GITHUB_OAUTH_SECRET
```

+ We will create a Web Application client from `oauthlib` based on `client_id`.

```py
   client = WebApplicationClient(client_id)
```

+ We will prepare a request body to access the token using the [`client.prepare_request_body`](https://oauthlib.readthedocs.io/en/latest/oauth2/clients/webapplicationclient.html#oauthlib.oauth2.WebApplicationClient.prepare_request_body) method with the variables -- `code`, `redirect_uri`, `client_id`, and `client_secret`.  We will save the return value of this method in `data`.

```py
   data = client.prepare_request_body(
      code = code,
      redirect_uri = settings.GITHUB_OAUTH_CALLBACK_URL,
      client_id = client_id,
      client_secret = client_secret
   )
```
Next, we will post a request at GitHub's `token_url` by calling the [`requests.post`](https://2.python-requests.org/en/master/api/#requests.post) method from the `requests` package using the variables -- `token_url` and `data` and saving the return value in `response`.

```py
    response = requests.post(token_url, data=data)
```

+ To make the response manageable as a Python dictionary, we call  [`client.parse_request_body_response()`](https://oauthlib.readthedocs.io/en/latest/_modules/oauthlib/oauth2/rfc6749/clients/base.html#Client.parse_request_body_response)with  `response.text`

```py
    client.parse_request_body_response(response.text)
```

+ The dictionary is saved as `client.token`. A sample response might be:

```
{
   'access_token': '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',
   'scope': ['read:user'],
   'token_type': 'bearer'
}
```

+ Next, we prepare a HTTP GET request with a special _Authorization_ header that bears this format:

```
Authorization: token token_value
```
We format this header and save it in `header`.

```py
header = {'Authorization': 'token {}'.format(client.token['access_token'])}
```

+ We use [`requests.get()`](https://2.python-requests.org/en/master/api/?highlight=requests.get#requests.get) to send a GET request at GitHub's URI endpoint, `'https://api.github.com/user'`, using our nicely-formated header to request profile data.


```py
   response = requests.get('https://api.github.com/user', headers=header)
```

+ We would like the result in JSON format, so we call `response.json()` and save the result in `json_dict`.

```py
   json_dict  = response.json()
```

+ Depending on our application, we may be interested in the following information:

```py
'login' => json_dict['login'],
'name' => json_dict['name'],
'bio' => json_dict['bio'],
'blog' => json_dict['blog'],
'email' => json_dict['email'],
'avatar_url' => json_dict['avatar_url'],
```

The `email` value may be set to `None` if the account holder prefers to keep it [private](https://github.com/settings/emails).

+ We may integrate a GitHub profile with a Django user. We can use GitHub's `login` credential as a Django `User`'s `username`. We can create a Django `User` based on the GitHub profile if it didn't exist before, or we can reference an existing `User` from our Django database. Assuming we save the `User` object as `user`, we should then log the `user` in.

```py
   login(self.request,user)
```

+ Finally, we redirect callback logic to a welcome view before returning from callback.

```py
    return HttpResponseRedirect(reverse('github:welcome'))
``` 

## Welcome View

This view is called after we successfully authenticated a Django user which is related to its GitHub account. We subclass our view from `TemplateView` to minimize coding (two lines!!).  The content of this view is in the template, **welcome.html**.

```py
class WelcomeView(TemplateView):
    template_name = 'welcome.html'
```

## Secret Page View

This view is called to display selective content based on whether a Django user is authenticated or not.  Like the `WelcomeView`, this view also has two lines.

```py
class HomeView(TemplateView):  
   template_name = 'home.html'
```

## Logout View

The logout view should log the current authenticated user out of the Django app. Its name should not be the same as the Django `logout()` function, otherwise the app will crash due to infinite recursion and stack overflow.

The view can display a confirmation message that the user has been logged out and render a home page. For example:

```py

def logout_request(request):    
   logout(request)    
   messages.add_message(request, messages.SUCCESS, "You are successfully logged out")    
   return render(request, 'home.html')
```

## Conclusion

With both `oauthlib` and `requests`, we can implement an OAuth2 Web Application flow in a Django app that integrates with a third-party provider such as GitHub. Although we can also use Django packages that are built using these Python packages and are well-integrated with many third-party providers, implementing an OAuth2 client ourself is more educational and satisfying. 

A demo app based on this article is available [here](https://aws.djangodemo.com/auth) if you would like to test it out. Thanks for reading and Happy Django!
