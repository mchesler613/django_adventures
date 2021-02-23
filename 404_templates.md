# Adventures in Django: Customizing 404 templates

![Custom 404 Page](https://i.postimg.cc/wMH5RtbF/project-404-2021-02-22-19-12-07.jpg)
There are many [HTTP status codes](https://en.wikipedia.org/wiki/List_of_HTTP_status_codes) that can be returned from a web server; one of them is the infamous `404 Not found` error code. Django handles the `Http404` exception by displaying a default error page for our application. 

If we set the option `DEBUG=True` in our project's **settings.py** file, we will see this.
![404 Debug=True](https://i.postimg.cc/jd8TRZRj/DEBUG-TRUE-404-2021-02-22-19-15-55.jpg)

Otherwise, if `DEBUG=False`, we will see this instead.
![404 Debug=False](https://i.postimg.cc/Z0zdn48r/DEBUG-FALSE-DEFAULT-404-Page-2021-02-22-19-34-56.jpg)

In a production setting, the default `404` page can be replaced by a custom template. Django requires that we name this template **404.html** and recommends that it be placed in the _root template directory_. This is the **templates** subdirectory residing in the inner project directory.  
```
myproject
   |--myapp
   |----migrations
   |--myproject
   |----__pycache__
   |--templates     <==
```

We also need to make sure the `DIRS` key in the `TEMPLATES` variable in our project's **settings.py** file is not an empty list.
```py
TEMPLATES = [
    {
        'BACKEND': 'django.template.backends.django.DjangoTemplates',
        'DIRS': [str(BASE_DIR.joinpath('myproject/templates'))],              # <==
        'APP_DIRS': True,
        'OPTIONS': {
            'context_processors': [
                'django.template.context_processors.debug',
                'django.template.context_processors.request',
                'django.contrib.auth.context_processors.auth',
                'django.contrib.messages.context_processors.messages',
            ],
        },
    },
]
```
Notice in the above example that the `DIRS` key is assigned a concatenated string of the `BASE_DIR` value and `myproject/templates`.

![Custom 404 Page](https://i.postimg.cc/wMH5RtbF/project-404-2021-02-22-19-12-07.jpg)

If Django cannot locate the **404.html** template in the root template directory, then Django will look for it in the app's **templates** directory if `APP_DIRS` is set to `True`. 

```
myproject
   |--myapp
   |----__pycache__
   |----migrations
   |------__pycache__
   |----templates        <==
   |------planner
   |--myproject
   |----__pycache__
   |--templates     
```

Having two versions of **404.html** residing in two different **templates** directories (root and app) may be helpful, but we have to configure our project's **settings.py** file to let Django know which version we want displayed on our website.  If we want the app's **404.html** to take over, then we need to reset the `DIRS` key back to an empty list, `[]`.

```py
TEMPLATES = [
    {
        'BACKEND': 'django.template.backends.django.DjangoTemplates',
        'DIRS': [],              # <==
        'APP_DIRS': True,
        'OPTIONS': {
            'context_processors': [
                'django.template.context_processors.debug',
                'django.template.context_processors.request',
                'django.contrib.auth.context_processors.auth',
                'django.contrib.messages.context_processors.messages',
            ],
        },
    },
]
```
![Custom 404 Page](https://i.postimg.cc/xTrjHwYk/app-404-2021-02-22-19-10-24.jpg)

## Conclusion
Django makes it rather easy to customize error pages besides **404.html**. We can also provide custom **400.html**, **403.html** and **500.html** pages for HTTP `Bad Request`, `Forbidden` and `Internal Server Error` [error codes](https://docs.djangoproject.com/en/3.1/ref/views/) respectively. Thank you for reading!
