================================
S3 backend for Python's keyring
================================

This module adds an `AWS S3`_ backend to Python's keyring_ module. The S3
backend will store the keyring credentials in an S3 bucket and use client and
server side encryption to keep the credentials safe both during transit and at
rest. This backend is quite handy when you want to distribute credentials across
multiple machines. Access to the backend and to the encryption keys can be
finely tuned using AWS IAM policies.

.. _AWS S3: https://aws.amazon.com/s3/
.. _keyring: https://pypi.python.org/pypi/keyring
.. _Key Management System: https://aws.amazon.com/kms/


Installation
------------

You can install the [stable release from Pipy](https://pypi.org/project/s3keyring/) by FindHotel (previously InnovativeTravel)

    pip install s3keyring

To install the development version of this fork.

    pip install git+https://github.com/AdRoll/s3-keyring


For Keyring Admins only: setting up the keyring
-----------------------------------------------

If you are just a user of the keyring and someone else has set up the keyring
for you then you can skip this section and go directly to ``For Keyring Users:
accessing the keyring`` at the end of this README. Note that you will need 
administrator privileges in your AWS account to be able to set up a new keyring 
as described below.


S3 bucket
~~~~~~~~~

The S3 keyring backend requires you to have read/write access to a S3 bucket.
If you want to use bucket ``mysecretbucket`` to store your keyring, you will
need to attach the following `IAM policy`_ to all the IAM user accounts or
roles that will have read and write access to the keyring::

    {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Effect": "Allow",
                "Action": [
                    "s3:ListBucket"
                ],
                "Resource": "arn:aws:s3:::mysecretbucket",
                "Condition": {}
            },
            {
                "Effect": "Allow",
                "Action": [
                    "s3:DeleteObject",
                    "s3:GetObject",
                    "s3:PutObject"
                ],
                "Resource": "arn:aws:s3:::mysecretbucket/*"
            }
        ]
    }

.. _IAM policy: http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/iam-policies-for-amazon-ec2.html

You can easily create a policy that grants read-only access to the keyring by
removing the ``s3:PutObject`` and ``s3:DeleteObject`` actions from the policy
above.


Encryption key
~~~~~~~~~~~~~~

You need to create a `KMS encryption key`_. Write down the ID of the
KMS key that you create. You will need to communicate this KMS Key ID to all
keyring users.

.. _KMS encryption key: http://docs.aws.amazon.com/kms/latest/developerguide/create-keys.html


**IMPORTANT**: You will need to grant read access to the KMS key to every IAM
user or role that needs to access the keyring.


For Keyring users: how to access the keyring
---------------------------------------------


One-time configuration
~~~~~~~~~~~~~~~~~~~~~~

If you haven't done so already, you will need to configure your local
installation of the AWS SDK.

If you don't have it already, you'll need AWS CLI: https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html

Then run::

    aws configure

You also need to ensure that you are using version 4 of the AWS Signature for
authenticated requests to S3::

    aws configure set s3.signature_version s3v4


Then you can simply run::

    s3keyring configure

Your keyring administrator will provide you with the ``KMS Key ID``,
``Bucket`` and ``Namespace`` configuration options. Option ``AWS profile``
allows you to specify the local `AWS CLI profile`_ you want to use to sign all
requests sent to AWS when accessing the keyring. Most users will want to use
the ``default`` profile. 

.. _AWS CLI profile: http://docs.aws.amazon.com/cli/latest/userguide/cli-chap-getting-started.html#cli-multiple-profiles

**IMPORTANT**: when deploying the `s3keyring` in EC2 instances that are granted
access to the keyring by means of an `IAM role` you should not specify a
custom AWS profile.

.. _IAM role: http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/iam-roles-for-amazon-ec2.html


You can configure the ``s3keyring`` module without user input by setting the
following environment variables: ``KEYRING_BUCKET``, ``KEYRING_NAMESPACE``, 
``KEYRING_KMS_KEY_ID``, ``KEYRING_AWS_PROFILE``. If these environment variables
are properly set then you can configure the ``s3keyring`` module with::

    s3keyring configure --no-ask



Configuration profiles
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

You can use ``s3keyring`` to store (read) secrets in (from) more than one
backend S3 keyring. A typical use case is creating different keyrings for 
different user groups that have different levels of trust. For instance your 
keyring administrator may have setup a S3 keyring that only IAM users with admin
privileges can access. Using the bucket, KMS Key ID and namespace provided by 
your keyring admin you can configure a separate ``s3keyring`` profile to access
that admins-only keyring::

    s3keyring --profile administrators configure

Your keyring admin may have also setup a separate S3 keyring to store secrets 
that need to be accessed by EC2 instances that act as website workers in a 
project you are working on. To access that keyring you would configure a 
second ``s3keyring`` profile::

    s3keyring --profile website-workers configure

Then, to store and retrieve secrets in the administrators keyring::

    s3keyring --profile administrators set SERVICE ACCOUNT PASSWORD 
    s3keyring --profile administrators get SERVICE ACCOUNT


And you could do the same for the ``website-workers`` keyring using option
``--profile website-workers``.


Custom configuration files
~~~~~~~~~~~~~~~~~~~~~~~~~~

By default `s3keyring` configuration is store in ``~/.s3keyring.ini``. However, 
you can also tell s3keyring to use a custom configuration file. In the CLI::

    # Store the configuration in a custom config file
    s3keyring --config /path/to/custom_config_file.ini configure
    # Read the configuration from a custom config file
    s3keyring --config /path/to/custom_config_file.ini get SERVICE ACCOUNT

When using the module API::

    from s3keyring.s3 import S3Keyring
    kr = S3Keyring(config_file='/path/to/custom_config_file.ini')
    kr.get_password('service', 'username')



Usage
-----

The ``s3keyring`` module provides the same API as Python's `keyring module`_.
You can access your S3 keyring programmatically from your Python code like
this::

    from s3keyring.s3 import S3Keyring
    kr = S3Keyring()
    kr.set_password('service', 'username', '123456')
    assert '123456' == kr.get_password('service', 'username')
    kr.delete_password('service', 'username')
    assert kr.get_password('service', 'username') is None


You can also use the keyring from the command line::

    # Store a password
    s3keyring set service username 123456
    # Retrieve it
    s3keyring get service username
    # Delete it
    s3keyring delete service username


As of version 0.7.0 s3-keyring also includes a caching mechanism for a namespace.
This works by saving a flat JSON file mapping keys to their passwords. This
allows for applications to pull down a single cache file instead of many
individual passwords to speed up launch times::

  # Update cache
  s3keyring build_cache
  # Retrieve cache
  s3keyring get_cache


.. _keyring module: https://pypi.python.org/pypi/keyring


Recommended workflow
~~~~~~~~~~~~~~~~~~~~

This is how I use ``s3keyring`` in my Python projects.

Let's assume that my project root directory looks something like this::

   setup.py
   my_module/
             __init__.py


In my project root directory I run::

    s3keyring --config my_module/.s3keyring.ini configure

I keep the generated ``.s3keyring.ini`` file as part of my project source code
(i.e. under version control). Then I paste the the code below in 
``my_module/__init__.py``::

    import os
    import inspect
    from s3keyring.s3 import S3Keyring
    
    __module_dir__ = os.path.dirname(inspect.getfile(inspect.currentframe()))
    __s3keyring_config_file__ = os.path.join(__module_dir__, '.s3keyring.ini')
    keyring = S3Keyring(config_file=__s3keyring_config_file__)


Then in my project code I store and retrieve secrets as follows::

    from my_module import keyring
    
    keyring.set_password('service', 'username', '123456')
    assert keyring.get_password('service', 'username') == '123456'


Who do I ask?
-------------

* German Gomez-Herrero, <german@innovativetravel.eu>
