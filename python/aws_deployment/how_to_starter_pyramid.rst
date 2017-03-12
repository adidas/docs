How-To: Pyramid Starter on AWS
==============================

Created: 2017-03-07

My previous post showed how to deploy a Pyramid Hello World App to 
AWS. Deploying the Pyramid Starter App is similar. It is diffferent by
the fact that there is more than just a application.py file and a requirements.txt
file. However the concepts are the same. The key thing here is that these steps 
do not require any knowledge of EB CLI. The project can be zipped up and 
uploaded to the AWS instance.

Code Deployment 
===============

We'll use the same zip archive upload via console style that was used in the 
Hello World app post because it doesn't require knowledge of EB CLI or boto.


Steps
=====

- Create a local pyramid app
- Add requirements.txt and application.py
- Deploy to AWS

Create a local pyramid app 
--------------------------

Obviously, this is an open-ended process. For the purposes of this How-To I'll
be using the pyramid starter project. Note that this uses the cookiecutter
utility.

Start by creating a directory for your project with your desired name. I prefer
creating sub-folders under that directory for various the different instances of 
the project, for example dev, prod, and test. I eventually use these for 
branches in a code versioning repository.

In the desired folder create the project by using the following cookiecutter 
command:

.. code:: bash

    cookiecutter https://github.com/Pylons/pyramid-cookiecutter-starter

This will prompt you for a name twice. Use the name of your project.

At this point, you have the foundation of your project. Create a
copy of the project directory, and continue installing the application in that
new "install" copy. This is used to create the requirements.txt so that the 
project won't have a env folder that we don't want in AWS. There are certainly
other ways to do this, but this is the way I chose to do it.

In the "install" copy use virtualenv to create a virtual environment directory 
under the root project folder. Name it env. Use the pip binary under this new virtualenv
to execute the following commands:

.. code:: bash

    env/bin/pip install --upgrade pip setuptools
    env/bin/pip install -e ".[testing]"

At this point the application should be usable. Check by using pserve to test.

.. code:: bash

    env/bin/pserve development.ini

This should result with

.. code:: bash

    Starting server in PID 26608.
    Serving on http://localhost:6543

And you should be able to get the boilerplate pyramid app when you visit
http://localhost:6543.
 
Add requirements.txt and application.py
---------------------------------------

Using the "install" copy create a requirements.txt using pip

.. code:: bash

    env/bin/pip freeze > requirements.txt

Remove the line that contains your application. AWS will not be able to find
it, and this will cause it to error upon deployment. However, there is a way
around this, see the section below named "Install Your App without remoting in".

Copy the requirements.txt to the original, pristine copy of your project.

Add a file called application.py that contains the following code:

.. code:: python 

    from pyramid.paster import get_app, setup_logging
    import os.path
    
    ini_path = os.path.join(os.path.dirname(__file__), 'production.ini')
    setup_logging(ini_path)
    application = get_app(ini_path, 'main')

AWS will use this to invoke your app for incoming requests.

Deploy to AWS
-------------

Once the application is on the AWS EC3 instance we'll need to install the
application via pip. Remote into the instance then find and activate your 
virtualenv. The activate script can be found under 
``/opt/python/run/venv/bin/activate``

Once activated run the following commands to install your app.

.. code:: bash

    cd /opt/python/current/app/
    /opt/python/run/venv/bin/pip install -e ".[testing]"

At this point your app will be working.

Install Your App without remoting in
------------------------------------

To make it so that you don't have to remote into the EC2 instance you can add the
following line to your requirements.txt

``/opt/python/ondeck/app/``

This tells pip to look in this directory for something to install and it is 
where AWS places your app during deployment.

Troubleshooting
===============

Static Resources are not found 
------------------------------

This was a curious problem. It seems that AWS has some automagic that will 
execute some logic if domain.com/static is requested. The way around this is to
change the name of your static asset folder.'' where post_date = '2017-03-07';
