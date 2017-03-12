==================================
How-To: Pyramid Hello World on AWS
==================================

* Code Deployment 
* Steps
* Troublepoints

When I tried creating a Pyramid app on AWS there were no definitive examples on
the web. I put together the following How-To to document what I did to get the
hello world app from Pyramid's documentation working on AWS. Please be aware
that this is not a full fledged application, instead it is a single python
script that serves up a very simple application. I also don't attempt to get
into anything more than the basics. Things like SSL, database connections, and
even version control software is ignored. The goal is to get a very rudimentary
app going on AWS for learning purposes.

Code Deployment 
===============

There are a few ways to get code up to AWS. The easiest by far is to use the
web console to upload the files. However, the other approaches are better for
automation.

Ways to get code to AWS:

* Upload via console 
* EB CLI 
* Boto

In this How-To we use the upload via console approach.

Steps
=====

* Create a working python script
* Add a requirements.txt file (optional)
* Create an key to ssh into the EC3 instance (optional)
* Zip up files for deployment
* Create an Elastic Beanstalk environment
* Install supporting modules (skipped requirements.txt)


Create a working python script
------------------------------
As mentioned above, this example implements the hello world code found in
`Pyramid's documentation
<http://docs.pylonsproject.org/projects/pyramid/en/latest/narr/firstapp.html#hello-world>`_.
The code is very simple, but it is meant to be a guide to understanding the
fundamentals of what is needed to get a Pyramid app working.

However, we cannot use this code as is. AWS requires that the "application" is
used as an variable within the code. This is how AWS finds the wsgi
application.  In this case I am turning the code from this:
 
.. code:: python

    from wsgiref.simple_server import make_server
    from pyramid.config import Configurator
    from pyramid.response import Response


    def hello_world(request):
        return Response('Hello %(name)s!' % request.matchdict)

    if __name__ == '__main__':
        config = Configurator()
        config.add_route('hello', '/hello/{name}')
        config.add_view(hello_world, route_name='hello')
        app = config.make_wsgi_app()
        server = make_server('0.0.0.0', 8080, app)
        server.serve_forever()
   
to this:

.. code:: python

    from wsgiref.simple_server import make_server
    from pyramid.config import Configurator
    from pyramid.response import Response


    def hello_world(request):
        return Response('Hello %(name)s!' % request.matchdict)

    config = Configurator()
    config.add_route('hello', '/hello/{name}')
    config.add_view(hello_world, route_name='hello')
    application = config.make_wsgi_app()

    if __name__ == '__main__':    
        server = make_server('', 8000, application)
        server.serve_forever()

Note that I moved the code required for the application variable outside of the
if statment. If you don't do this AWS will complain with

.. code:: bash 

    Target WSGI script '/opt/python/current/app/application.py' does not contain WSGI application 'application'.

The code must also be in file called application.py. 

Add a requirements.txt file (optional)
--------------------------------------

This step is optional, but recomended. It makes installation much easier and 
remoting into the server isn't needed. Alternatively, you can remote in and
install the packages. However, zope.deprecation 4.1.2 must be installed. 

Regardless, if a requirements.txt file is included AWS will install the
modules in this file when deploying the app. This makes deployment much faster
and easier.

However, I did run into another issue. There appears to be an issue with
zope.deprecation. It seems that the latest version of zope.deprecation broke 
something. However, this is a topic for another post.

The requirements.txt that I used contained the following:

.. code:: python

    zope.deprecation==4.1.2
    pyramid==1.8.1

Create an key to ssh into the EC3 instance (optional)
-----------------------------------------------------

If you need to access the EC2 instance via SSH then this step is needed.

This is done through the AWS web console at the `EC2 console
<https://console.aws.amazon.com/ec2/>`_. There navigate to Network & Security > 
Key Pairs under the navigation pane. From there create the key pair and
download the pem file. In a linux environment that pem file needs its 
permissions changed with the following command.

.. code:: bash

    chmod 400 my-key-pair.pem

`AWS
<http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-key-pairs.html#having-ec2-create-your-key-pair>`_ has more information on this.


This key can then be used to SSH into the EC2 instance.

Zip up files for deployment
---------------------------

All that is needed here is to package up application.py and requirements.txt
into a zip file. It can be named anything you want. It just needs to be a zip
file.

Create an Elastic Beanstalk environment
---------------------------------------

Navigate to the Elastic Beanstalk environment console. Click the "Create
Application" button. Pick a name (description is optional) and submit the form.

This will create an empty application. Within this, create an environment. In
the "Create Environment" wizard choose "web server environment". On the next
prompt pick the preconfigured python platform and choose the "Upload your code"
option under the application code field. Then click the prompt to upload your
zip file.

If you need SSH capabilities pick "Configure more options" and navigate to the
security pane. There pick the key pair you created under the EC2 console.

Also note that for SSH you'll need to app inbound access for port 22. This can
be done in the EC2 console under Network & Security > Security Groups. Pick the
relevant security group and add an inbound rule at the bottom of the screen.

If you used a requirements.txt file your application should be complete at this
point.

Install supporting modules (skipped requirements.txt)
-----------------------------------------------------

Once the instance is set up SSH into it, and execute the following commands:

First activate the virtual environment.

.. code:: bash

    . /opt/python/run/venv/bin/activate

Next install zope.deprecation. Remember version 4.1.2
 
.. code:: bash

    sudo /opt/python/run/venv/bin/pip install zope.deprecation==4.1.2

Then install Pyramid

.. code:: bash

    sudo /opt/python/run/venv/bin/pip install pyramid

You may prefer this approach, but if you are setting up multiple instances a 
requirements file is preferred.

At this point your application should be complete.

Troubleshooting
===============

* SSH connection
* application not found
* ImportError: No module named 'zope.deprecation'

SSH connection
--------------

Problem: I wasn't able to connect to the VM instance to troubleshoot issues.

Solution: It turns out that you can't add a security key to a VM instance after 
it has been created. I had to create a new VM to get around this. However, I 
had to create the security key first. When creating the VM the security isn't 
in the basic configuration. It's under the advanced configs. Once this was done
I was able to SSH into the machine.

One other thing to note, I wasn't able to use the web browser connection since
it used Java which doesn't work in Vivaldi.

application not found
---------------------

Problem: AWS wasn't able to find the python application. Server kept emitting the 
following error:

.. code:: bash

    Target WSGI script '/opt/python/current/app/application.py' does not contain WSGI application 'application'.


It turns out that AWS looks for "application" in the code. I simplly changed

.. code:: python 

    app = config.make_wsgi_app()

to

.. code:: python

    application = config.make_wsgi_app()

Also, needed to move it all before the if statment.

ImportError: No module named 'zope.deprecation'
-----------------------------------------------

Problem: something is up?

Solution: I did two things. First, changed to Pyramid 1.7 and zope.deprecation 
4.1.2. Second, added a requirements.txt. The actual problem was that the latest
version of zope.deprecation broke something.
