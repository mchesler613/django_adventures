# Deploying Django on AWS and Bitnami
I recently attempted to deploy my Django projects on [Amazon Lightsail](https://lightsail.aws.amazon.com/) and took notes for myself on the issues that came up for me. For starters, the AWS documentation for deploying a plain Django app without any database can be found [here](https://aws.amazon.com/getting-started/hands-on/deploy-python-application/). 

On the AWS console, we will find many Bitnami applications to choose from, including Django. 
![Bitnami Apps](https://d1.awsstatic.com/LightsailAssets/django5.8b5e50b084f97517143d00c3ee5d9dee7937f66a.png)
If we run into any trouble deploying Django that is Bitnami-related, we can find support from [Bitnami](https://bitnami.com/support). 

## Error: Attempt to write a readonly database
Since the AWS tutorial is for a barebones Django app, it doesn't provide documentation for apps with database. If our project uses a database like mine, we will most likely encounter this error -- `Attempt to write a readonly database` while the `DEBUG=True` setting is enabled in **settings.py**. 

Running a production server using Apache requires a different command than `python manage.py runserver`. As noted in the AWS tutorial, the command would be:
```
$ sudo /opt/bitnami/ctlscript.sh restart apache
```
If we were to locate the `apache` process on our terminal with `ps -ef | grep apache` we will see something like this:
```
bitnami@ip-xxx-xx-x-xx:/opt/bitnami/projects/iplan$ ps -ef | gre
p apache
root     26520     1  0 16:17 ?        00:00:00 /opt/bitnami/apache/bin/httpd -f /opt/bitnami/apache/conf/httpd.conf
daemon   26523 26520  0 16:17 ?        00:00:01 /opt/bitnami/apache/bin/httpd -f /opt/bitnami/apache/conf/httpd.conf
daemon   26524 26520  0 16:17 ?        00:00:01 /opt/bitnami/apache/bin/httpd -f /opt/bitnami/apache/conf/httpd.conf
daemon   26525 26520  0 16:17 ?        00:00:01 /opt/bitnami/apache/bin/httpd -f /opt/bitnami/apache/conf/httpd.conf
daemon   26526 26520  0 16:17 ?        00:00:00 /opt/bitnami/apache/bin/httpd -f /opt/bitnami/apache/conf/httpd.conf
daemon   26527 26520  0 16:17 ?        00:00:00 /opt/bitnami/apache/bin/httpd -f /opt/bitnami/apache/conf/httpd.conf
```
Since we ran the apache server as `sudo`, we will find multiple server processes - One that is owned by `root` and the others by the user `daemon`. `daemon` as defined by [Linux](https://www.man7.org/linux/man-pages/man7/daemon.7.html) is a service process that operates in the background servicing other processes. There is also a `daemon` [user and group](https://refspecs.linuxbase.org/LSB_3.0.0/LSB-PDA/LSB-PDA/usernames.html#TBL-REQUIREDUSERS) associated with this process. 

When we deployed Django on AWS, we were logged in as the user, `bitnami`. To enable the `daemon` process to read from and write to our default database, we need to configure the read and write permissions to the database file. To get to the database file, the `daemon` process also needs the same permissions for the project root directory. So, we will have to do two things:
1. Change group ownership of the project root directory and the database file to `daemon`. For example:
```
$ sudo chgrp daemon project_directory project_directory/database_file
```
2. Make the project root directory and the database file writable by the `daemon` group. For example:
```
$ sudo chmod g+w project_directory project_directory/database_file
```
To see if the database error goes away, try reloading the Django app on the browser.

## Setting Server Timezone
The Apache server's default timezone is in `UTC`. This may not be what we want. It's helpful to align the server timezone with our local timezone to make debugging easier. I would refer you to this [link](https://it.playswellwithflavors.com/2020/03/25/bitnami-set-timezone/) to change the timezone.

## Deploying Multiple Projects On the Same Server

Recall a section of the [AWS](https://aws.amazon.com/getting-started/hands-on/deploy-python-application/) tutorial that addresses configuring our project **wsgi.py** file which looks like this.
```
import os
import sys
sys.path.append('/opt/bitnami/projects/tutorial')
os.environ.setdefault("PYTHON_EGG_CACHE", "/opt/bitnami/projects/tutorial/egg_cache")
os.environ.setdefault("DJANGO_SETTINGS_MODULE", "tutorial.settings")
from django.core.wsgi import get_wsgi_application
application = get_wsgi_application()
```
If we want to run multiple projects on the same server, it's best to not set the default  `DJANGO_SETTINGS_MODULE` and `PYTHON_EGG_CACHE` to point to a particular project. Doing so for multiple projects will lead to [unexpected results](https://stackoverflow.com/questions/11505576/deploying-multiple-django-apps-on-apache-with-mod-wsgi). 

Instead of this:
```py
os.environ.setdefault("PYTHON_EGG_CACHE", "/opt/bitnami/projects/tutorial/egg_cache")
os.environ.setdefault("DJANGO_SETTINGS_MODULE", "tutorial.settings")
```
do this:
```py
os.environ["PYTHON_EGG_CACHE"] = "/opt/bitnami/projects/tutorial/egg_cache"
os.environ["DJANGO_SETTINGS_MODULE"] = "tutorial.settings"
```

We can extend the bitnami configuration found in **/opt/bitnami/apache2/conf/bitnami/bitnami.conf**, to include other projects. For example:
```py
<VirtualHost _default_:80>
    WSGIScriptAlias /project1 /opt/bitnami/projects/project1/project1/wsgi.py

    <Directory /opt/bitnami/projects/project1>
        AllowOverride all
        Require all granted
        Options FollowSymlinks
    </Directory>
 
    WSGIScriptAlias /project2 /opt/bitnami/projects/project2/project2/wsgi.py

    <Directory /opt/bitnami/projects/project2>
        AllowOverride all
        Require all granted
        Options FollowSymlinks
    </Directory>
 
</VirtualHost>
 
Include "/opt/bitnami/apache/conf/bitnami/bitnami-ssl.conf"
```
Notice the last line that includes the **bitnami-ssl.conf** file. If we secure our production server with `SSL` (a good recommendation), we will need to add the same multiple-project configuration in **bitnami-ssl.conf** under `<VirtualHost _default_:443>`.

## Obtain a Static IP address for our Django Server

When we first created a Lightsail instance on AWS, we were given an IP address, for example, 192.0.2.143.
![IP Address](https://d1.awsstatic.com/LightsailAssets/django10.3a7d2a117c080dc690baa9c471a1baad26ad89ba.png)

Note that this IP address is temporary and will change each time we reboot the instance. This would mean we have to update the `ALLOWED_HOSTS` variable in our project **settings.py** file each time the IP address changes per reboot. 

It is best to create a static IP address that we can use throughout the lifetime of the hosted Django apps.

For example, instead of this:
```py
ALLOWED_HOSTS = ['192.0.2.143']
```
we can do this:
```py
ALLOWED_HOSTS = 'awesomedjango.com'
```

Consult this other AWS [tutorial](https://lightsail.aws.amazon.com/ls/docs/en_us/articles/lightsail-create-static-ip) to set up a static IP address.

## Restarting the Apache Server
Unlike the development Django server, the production Apache server does not automatically detect changes that we make in the Django configuration or application. We need to restart the Apache server for our changes to take effect. For example:
```
$ sudo /opt/bitnami/ctlscript.sh restart apache
```

## Log Files
When we disable debugging for our app by setting `DEBUG=False` in our project **settings.py** file, we can no longer see Django's detailed traceback on the web page when an error occurs.

For example:
![Django Traceback](https://external-content.duckduckgo.com/iu/?u=https%3A%2F%2Fi.stack.imgur.com%2FtpBWn.png&f=1&nofb=1)

But we can find a shortened traceback in the log files generated by Apache. There are two log files of interest:
+ **/opt/bitnami/apache/logs/access_log**
+ **/opt/bitnami/apache/logs/error_log**

A sample error log file from Bitnami contains this:
```
bitnami@ip-xxx-xx-x-xx:~$   File "/opt/bitnami/python/lib/python
3.8/site-packages/asgiref/local.py", line 96, in __del__
NameError: name 'TypeError' is not defined
Exception ignored in: <function Local.__del__ at 0x7f1ecee3c550>
Traceback (most recent call last):
  File "/opt/bitnami/python/lib/python3.8/site-packages/asgiref/
local.py", line 96, in __del__
NameError: name 'TypeError' is not defined
```
These log files can get really big, but we are mostly interested in the latest access and error messages. Dedicate a terminal to display only the last several lines. For example, type:
```
$ tail -f /opt/bitnami/apache/logs/error_log
```
and optionally run it in the background, by typing <kbd>Ctrl</kbd><kbd>Z</kbd> followed by `bg` in the terminal.

## Conclusion
Depending on your projects and apps, you may need to provide additional configuration for your production server. For additional helpful tips in deploying Django in a production environment, we can consult this [documentation](https://docs.djangoproject.com/en/3.1/howto/deployment/checklist/).  

As you can see, deploying our Django app on a production server on the cloud such as AWS requires a lot of work. But don't let that deter you as once you have successfully overcome the hurdles, you will be proud of what you have accomplished and learn a lot that may earn you a ticket to a satisfying career. Happy Django!
