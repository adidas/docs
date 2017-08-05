How-To: Pyramid Starter on AWS
==============================

Created: 2017-03-07

My previous post showed how to deploy a Pyramid Hello World App to  AWS. Deploying the Pyramid Starter App is similar.
It is diffferent by the fact that there is more than just a application.py file and a requirements.txt file. However the
concepts are the same. The key thing here is that these steps  do not require any knowledge of EB CLI. The project can
be zipped up and  uploaded to the AWS instance.

Code Deployment 
===============

We'll use the same zip archive upload via console style that was used in the  Hello World app post because it doesn't
require knowledge of EB CLI or boto.


Steps
=====

- Create a local pyramid app
- Add requirements.txt and application.py
- Deploy to AWS

Create a local pyramid app 
--------------------------

Obviously, this is an open-ended process. For the purposes of this How-To I'll be using the pyramid starter project.
Note that this uses the cookiecutter utility.

Start by creating a directory for your project with your desired name. I prefer creating sub-folders under that
directory for various the different instances of  the project, for example dev, prod, and test. I eventually use these
for  branches in a code versioning repository.

In the desired folder create the project by using the following cookiecutter  command:

.. code:: bash

    cookiecutter https://github.com/Pylons/pyramid-cookiecutter-starter

This will prompt you for a name twice. Use the name of your project. It will also ask for what type of templating engine
you want to use. I prefer Mako, but I am finding Jinja2 isn't that terrible. Do your research and make a choice if you
are unfamiliar with the choices.

At this point, you have the foundation of your project. Use the following to create a virtual environment in your
project directory.

.. code:: bash

    python -m venv env

Note: You can also set an enviroment variable to make things easier.

.. code:: bash

    env/bin/pip install --upgrade pip setuptools
    env/bin/pip install -e ".[testing]"

At this point the application should be usable. Check by using pserve to test.

.. code:: bash

    env/bin/pserve development.ini

This should result with something like the following:

.. code:: bash

    Starting server in PID 26608.
    Serving on http://localhost:6543

And you should be able to get the boilerplate pyramid app when you visit http://localhost:6543.
 
Add requirements.txt and application.py
---------------------------------------

Create a requirements.txt using pip

.. code:: bash

    env/bin/pip freeze > requirements.txt

Remove the line that contains your application. AWS will not be able to find it, and this will cause it to error upon
deployment. However, there is a way around this, see the section below named "Install Your App without remoting in".

Also, change the version of zope.deprecation to 4.1.2. zope.deprecation has issues because the python build for Amazon
Linux is missing wheels for ``python -m venv`` to work correctly. As a result it has been found that the python used in
Amazon Linux is not setting up the lib directories correctly in virtual environments, and this is causing issues with
the zope.deprecation and zope.interface installs. One is being installed to a directory under lib64 and the other is
being installed under lib.

You can read more about this `here <http://bruisedthumb.com/post/2017-03-20>`_.

Next, add a file called application.py that contains the following code:

.. code:: python 

    from pyramid.paster import get_app, setup_logging
    import os.path
    
    ini_path = os.path.join(os.path.dirname(__file__), 'production.ini')
    setup_logging(ini_path)
    application = get_app(ini_path, 'main')

AWS will use this to invoke your app for incoming requests.

Deploy to AWS
-------------

I won't review how to create the Elastic Beanstalk instance here. Instead look at my `Hello Pyramid <http://bruisedthumb.com/post/2017-03-05#create-an-elastic-beanstalk-environment>`_ article for the section on how to create an environment.

Once the application is on the AWS EC2 instance we'll need to install the application via pip. Remote into the instance
then find your virtual envronment. Use its pip to install your application.

Run the following commands to install your app.

.. code:: bash

    cd /opt/python/current/app/
    /opt/python/run/venv/bin/pip install -e ".[testing]"

At this point your app will be working.

Install Your App without remoting in
------------------------------------

To make it so that you don't have to remote into the EC2 instance you can add the following line to your
requirements.txt

``/opt/python/ondeck/app/``

This tells pip to look in this directory for something to install and it is  where AWS places your app during
deployment.

Troubleshooting
===============

Static Resources are not found 
------------------------------

This was a curious problem. It seems that AWS has some automagic that will  execute some logic if domain.com/static is
requested. The way around this is to change the name of your static asset folder.

pkg-resources can't be installed
--------------------------------

It turns out Ubuntu may cause pip to report that pkg-resources is installed. `This is a bug <http://stackoverflow.com/questions/39577984/what-is-pkg-resources-0-0-0-in-output-of-pip-freeze-command>`_. If you are working from Ubuntu and encounter this while creating your requirements.txt file you can safely remove it from your requirements.txt.

Complete Code
=============

The code used in this How-To can be found at my `github account <https://github.com/danclark5/aws_pyramid_starter>`_.
It even has it packaged up into a zip file that can be uploaded to a AWS instance.

Last Update: 2017-03-22
