python34 for Amazon Linux is Missing It's Wheels
================================================

Created: 2017-03-20

Since writing my first two posts I found out that I was doing it wrong. In the past I would use virtualenv to create
virtual environments. I would also use the activate script which I later found out "drops turds" into your shell
environment. All of this was a result of my upbringing in Python, and I suspect that many people still have this issue to some degree. They use what they know works and stick with it.

It seems that Amazon is no different.

Doing it Right
==============

The proper way to create a virtual environment in Python 3.4 is to use the following

.. code:: bash

    export VENV=~/env
    python3 -m venv $VENV

It turns out that Python has everything you need to create a virtual environment, as long as Python is built correctly.
Also, avoid the activate script. Instead, use your newly created environment variable to invoke parts of your virtual
environment directly. For example:

.. code:: bash

    $VENV/bin/pserve development.ini

It isn't Amazon Linux at Fault 
==============================

Amazon Linux is derived from Red Hat, and Red Hat has a guideline of `not bundling system libraries <https://fedoraproject.org/wiki/Packaging:Guidelines#Bundling_and_Duplication_of_system_libraries>`_. Unfortunately this
meant that a decision was made to not include the pip and setuptools wheels within the ensurepip package.

As a result, trying to create a virtual environment with the commands above are broken, and return the following error:

.. code:: bash
    
    Error: Command '['/home/ec2-user/env/bin/python3', '-Im', 'ensurepip', '--upgrade', '--default-pip']' returned non-zero exit status 1

Long Story Short
================

I have since posted on AWS's forums to get them to include a python34 build that does have the wheels. Hopefully, they do, but it's possible that they won't or that it will take them some time to get AMI's that include the proper Python install. There are work arounds, but they aren't pretty.

I am going to avoid retelling the whole story here, but you can follow `my adventure <https://groups.google.com/forum/#!topic/pylons-discuss/MSAMBzwx7aQ>`_, `the bug on Red Hat <https://bugzilla.redhat.com/show_bug.cgi?id=1263057>`_, `my post on the AWS forum <https://forums.aws.amazon.com/thread.jspa?threadID=251803&tstart=0>`_, or `the stackoverflow thread that led me to the Red Hat bug <http://stackoverflow.com/questions/32618686/how-to-install-pip-in-centos-7>`_. Let's hope that this get resolved quickly and that I can remove the workarounds from my How-Tos.

THANKS
======
Special thanks to the following people for providing support to help solve what was really happening

* Steve Piercy
* Bert JW Regeer
* Tres Seaver
* Michael Merickel 