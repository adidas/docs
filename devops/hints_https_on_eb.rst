Hints: Pulling Files from S3 upon Elastic Beanstalk Deployment
====================================

Created: 2017-04-16

Over the last few weeks (disclaimer: I only have an hour or two each morning) I have been learning by trial and error
the best way to deploy a HTTPS-enabled application. Three possible options are through the AWS Certificate Manager
(ACM), Let's Encrypt, and using your own CA issued certificate. There might be others, but these are the methods I
tried. In this article I will go over each and what ended up working for me.

As a general rule I try to take the path of least resistance. If a method requires learning something I don't
immediately know, and it looks like it will take more than a few seconds to understand I move on to the next best
prospective option. Of course if I exhaust every path then I will go back and learn the tech needed. In this example, EB CLI.

We'll be going over

* AWS Certificate Manager
* Let's Encrypt
* BYO certificate 
* The option that did work

AWS Certificate Manager
=======================

This is by far the easiest to use. It can be used through the web console and is pretty intuitive.
However, it is too simplistic and that means you might not be able to use it, or you'll have to adapt your project around it.
Below I list out the benefits and drawbacks that I noticed.

Benefits:
* Easy to use.
* Quick to get started.
* Little knowledge needed to deploy a HTTPS app.

Drawbacks
* Terminates HTTPS at the load balancer.
* Can't be used with single instance deployments

The benefits speak for themselves. I actually had an Elastic Beanstalk instance up and running with this approach very quickly. However, I ran into an issue. Since the HTTPS requests were terminated at a load balancer my app was creating HTTP links. One of my requirements was no HTTP. Everything would be redirected to HTTPS.

This approach started to become more involved, and the rigidness of ACM made me look else where. I could have made my application serve up everything via HTTPS, but that felt like I was avoiding the real problem. Also, there were a few posts I found sporadically that mentioned that ACM didn't work with single instance environments. All of these made me think that this probably wasn't going to work out, at least easily. 

Let's Encrypt
=============

This option sounded awesome. They seem to have a large community backing them up, documentation is every where, and it was free! However, there wasn't any documentation by AWS, and since I had a very vague understanding of HTTPS at the start of this I didn't know how to apply the existing documentation very easily.

What I found was that a HTTP server was needed by the Let's Encrypt script to establish trust, and I wasn't comfortable playing with my existing server as it was working just fine. I did try to get it working with another test instance, but I kept running into issues. All I wanted was the certificate. To make things more complicated the certificates only lasted a few months. This means that many people recommended setting up a cron job to renew the certificate monthly.

These points all created too much resistance for me, and I decided I'd try the next option.

BYO Certificate
===============

The traditional way to get HTTPS working is to create a private key and a certificate signing request (CSR), submit it to a cerficate authority (CA), and get a certificate and chain bundle in return. The private key, certificate, and chain can then added to the server, and once the web server is configured, HTTPS requests can be properly handled.

The benefits here are

* Plenty of documentation for many combinations, even for python on AWS Elastic Beanstalk.
* Options to verify who you are.
* Relatively cheap. It only cost me $10 a year for a single domain. I've spent more money on more questionable things.
* Best option to get a basic understanding of how HTTPS works

Drawbacks:

* Have to pay some money.
* Verification is tougher than ACM
* Have to go through a third party for the certificate.
* It can be finicky to set up on AWS if you don't know what you're doing, but it was the easiest of the three for me. 

Between learning how HTTPS works and the vast amount of documentation this this yielded the best results for me.

The Option that Worked for me
=============================

At this point it should be clear that I went with BYOC (bring your own certificate). I went with a single domain certificate to keep things simple. However, it took me time to figure out the proper way to get AWS to do what I wanted. I quickly realized that I wasn't going to be able to use the web console to do this. It lacks the ability to customize certain configurations that are need to implement HTTPS properly.

I will note that some reading is required at this point. There are `plenty of resources out there on HTTPS <https://www.google.com/search?q=HTTPS%20basics#q=HTTPS+TLS+basics>`_ to get understanding of SSL (and why it isn't used anymore), TLS, and the various things that go into making HTTPS happen. You could probably get away with skimming the material, but I would strongly recommend learning it if you are securing an application where privacy is needed. If you have a login then read up on it.

Get the Certificate
-------------------

I used an existing instance to generate the certificate signing request (CSR). I followed the `instructions <https://www.namecheap.com/support/knowledgebase/article.aspx/467/67/how-do-i-generate-a-csr-code>`_ on Namecheap. My instance uses Apache, and `their instructions <https://www.namecheap.com/support/knowledgebase/article.aspx/9446/0/apache-opensslmodsslnginx>`_ worked out great. Basically, all I had to do was 

.. code:: bash

openssl req -new -newkey rsa:2048 -nodes -keyout server.key -out server.csr

and then follow the prompts. The important bit is Common Name. It has to be the domain that you are securing.

Once this bit is done save both the server.key and the server.csr. You will need them both later. To confirm the files, you can inspect them with the text editor of your choice. The server key will contain a header that says something like "-----BEGIN RSA PRIVATE KEY-----", and the CSR will say "-----BEGIN CERTIFICATE REQUEST----."

Then go to your certificate authority (CA) of choice and submit your CSR. I bought a PositiveSSL certificate from Comodo through Namecheap. The first step is to submit the CSR which can be done by either pasting the text into the submission form or upload. Then Comodo will require that you use one of three methods to validate who you are. They are email, hosting a verification doc, or creating a CNAME record.

Once validated they will email you the certificate and a bundle file. Both are needed for proper certificate deployment.

Deploying the Cerficate and Supporting Artifacts
------------------------------------------------

There is no way of getting around it. You need to use `configuration files <http://docs.aws.amazon.com/elasticbeanstalk/latest/dg/ebextensions.html>`_. An .ebextension file was the easiest way for me to get AWS to import and configure HTTPS. I also used `EB CLI <http://docs.aws.amazon.com/elasticbeanstalk/latest/dg/eb-cli3.html>`_ To make environment deployments easier

.. Disclaimer:: This approach forced me to learn EB CLI which is a very powerful tool compared to AWS's web console. In my past articles I described how python is broken with AWS. I think it is possible to add in a working python install using EB CLI by installing a python build that properly supports pip, but that is an topic for another article.
