FAQ
===

How do I set AWS Region or Profile via the CLI?
-----------------------------------------------

These configuration items, and others, can be set from the CLI using
:ref:`templating <stack_group_config_templating>`. The syntax for setting the AWS region is presented below. The
syntax for replacing the other items is the same, with the word ``region``
substituted out.

In config.yaml:

.. code-block:: yaml

   region: {{ var.region }}

On the CLI:

.. code-block:: text

    sceptre --var "region=<your region name>" COMMAND

Should I use a Parameter or Sceptre User Data?
----------------------------------------------

Parameters are the standard way of passing StackGroup-specific configuration to
a CloudFormation template. They offer:

-  Native support from CloudFormation
-  A high degree of customisability, as described in the `AWS documentation`_

However, parameters suffer from the following limitation:

-  They cannot alter the template based on the parameter value

Sceptre user data was added to fix this limitation. It is configuration that is
passed directly to a template, and so can be used to change the template based
on the configuration item’s value.

For example, if the Sceptre user data item ``number_of_azs`` is passed to a
subnet tier template, the value could be used to create different numbers of
subnets. This cannot be done with native parameters.

In general, native CloudFormation parameters should be used in almost all
cases. Sceptre user data should only be used when the user wants to alter the
template based on the value of a config value.

.. _faq_stackconfig_env:

Why are there two ways to supply environment variables in stack config files?
-----------------------------------------------------------------------------

It is possible to replace values in stack config files with environment
variables in two ways.

The first is by using templating, and the syntax
``{{ environment_variable.VALUE }}``. Any value in a
config file may be replaced using this method.

The second is by using a resolver, and the syntax:

.. code-block:: yaml

   parameters:
     param_1: !environment_variable VALUE

This second syntax is only able to replace parameter values.

The difference between these two methods lies in when the value of the
environment variable is obtained. The templating method obtains the environment
variable when Sceptre is first invoked. The resolver method obtains the
environment variable just before the operations ``create``, ``update``,
``launch`` operation is executed (Note that these are the only commands which
require parameters). This difference becomes most apparent with the use of the
``launch`` command. As multiple stacks are launched, early stacks can set
environment variables (using Hooks) which can then be read in by later stacks.
This is only possible using resolvers.

In order to use environment variables set by Hooks run by a stack previously
built in the same ``launch`` command, the environment variable resolver must be
used.

.. _AWS documentation: http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/parameters-section-structure.html

.. _using_connection_manager:

How do I call AWS services or use AWS-based tools in my custom hook/resolver/template handler?
----------------------------------------------------------------------------------------------
In order to call AWS services in your custom hook/resolver/template handler properly, you should use
the IAM configurations of the stack where the resolver is being used (unless you need to use a
different configuration for a specific reason). This means your hook/resolver/handler should honor the
``profile``, ``region``, and ``iam_role`` configurations as set for your project and/or Stack Config.
Simply invoking ``boto3.client('s3')`` is _not_ going to regard those and could end up using the
wrong credentials or not even working.

There is a simple interface available for doing this properly, the
:py:class:`sceptre.connection_manager.ConnectionManager`. The ConnectionManager is an interface that
will be pre-configured with each stack's profile, region, and iam_role and will be ready for you to use.
If you are using an `iam_role`, it will automatically assume that role via STS for making calls to
AWS so you can just use it the way you want. It is accessible on hooks and resolvers via
``self.stack.connection_manager`` and on template_handlers via ``self.connection_manager``.

There are three public methods on the ConnectionManager:

- :py:meth:`sceptre.connection_manager.ConnectionManager.call` can be used to directly call a boto3
  API method on any api service and return the result from boto3. This is perfect when you just need
  to invoke a boto3 client method without any additional capabilities.
- :py:meth:`sceptre.connection_manager.ConnectionManager.get_session` can be used to get a boto3 Session
  object. This is very useful if you need to work with Boto3 Resource objects (like an s3 Bucket) or
  if you need to create and pass the bot3 session, client, or resource to a third-party framework.
- :py:meth:`sceptre.connection_manager.ConnectionManager.create_session_environment_variables` creates
  a dictionary of environment variables used by AWS sdks with all the relevant connection information.
  This is extremely useful if you are needing to invoke other SDKs using ``subprocess`` and still need
  the Stack's connection information honored.


Using the connection manager, you can use `boto3 <https://boto3.amazonaws.com/v1/documentation/api/latest/index.html>`_
to perform any AWS actions you need:

.. code-block:: python

   # For example, in your custom resolver:
   def resolve(self):
       # You can invoke a lower-level service method like...
       obj = self.stack.connection_manager.call('s3', 'get_object', {'Bucket': 'my-bucket', 'Key': 'my-key'})
       # Or you can create higher-level resource objects like...
       bucket = self.stack.connection_manager.get_session().resource('s3').Bucket('my-bucket')
       # Or if you need to invoke a third-party tool via a subprocess, you can create the necessary environment
       # variables like this:
       environment_variables = self.stack.connection_manager.create_session_environment_variables(
           include_system_envs=True
       )
       list_output = subprocess.run(
           'aws s3 list-bucket',
           shell=True,
           env=environment_variables,
           capture_output=True
       ).stdout


My CI/CD process uses ``sceptre launch``. How do I delete stacks that aren't needed anymore?
---------------------------------------------------------------------------------------------

Running the ``launch`` command is a very useful "1-stop-shop" to apply changes from Stack Configs,
creating stacks that don't exist and updating stacks that do exist. This makes it a very useful
command to configure your CI/CD system to invoke. However, sometimes you need to delete a stack that
isn't needed anymore and you want this automatically applied by the same process.

This "clean up" is complicated by the fact that Sceptre doesn't know anything that isn't in its
Stack and StackGroup Configs; If you delete a Stack Config, Sceptre won't know to clean it up.

Therefore, the way to accomplish this "clean up" operation is to perform the change in 3 steps:

1. First, add ``obsolete: True`` to the Stack Config(s) you want to clean up.
   For more information on ``obsolete``, see the :ref:`Stack Config entry on it<obsolete>`.
2. Update your CI/CD process to run ``sceptre launch --prune`` instead of ``sceptre launch``. This
   will cause all stacks marked as obsolete to be deleted going forward.
3. Once your CI/CD process has cleaned up all the obsolete stacks, delete the local Stack Config files
   you marked as obsolete in step 1, since the stacks they create have all been deleted.

.. note::

   Using ``obsolete: True`` will not work if any other stacks depend on that stack that are
   not themselves obsolete. Attempting to prune any obsolete stacks that are depended on by
   non-obsolete stacks will result in Sceptre immediately failing the launch.
