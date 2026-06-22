Command line tools
==================

nscrub-cli
----------

*nscrub-cli* is the command line tool for configuring nScrub using autocompletion, instead of using the REST API manually.

You can run *nscrub-cli* to get access to the nscrub configuration, without any option or passing user/password and host/port if you customised the nscrub configuration.

.. code-block:: console

   nscrub-cli -h
   nscrub-cli [-v] [-h] [-a <user>:<password>] [-c <host>:<port>]

Run *nscrub-cli* to get a console, then type “help” to get a full list of supported commands. Please refer to the REST API for a description.

A wizard tool *nscrub-add* is also available for creating a new target with a basic configuration, providing the target ID, subnet and service type.

.. code-block:: console

   nscrub-add [options] <target id> <subnet> <target type>

Example:

.. code-block:: console

   nscrub-add WEBSERVER 192.168.1.1/32 web

Please note that further configurations are usually needed to create an optimal configuration based on the specific target, it is possible to use *nscrub-cli* for that.

*nscrub-cli* also allows you to pass a configuration file containing *nscrub-cli* commands using pipe. Example:

.. code-block:: console

   cat nscrub.conf
   targets add TEST 192.168.1.2/32
   targets add TEST 192.168.1.3/32
   cat nscrub.conf | nscrub-cli

Please refer to the *nscrub-cli* help for an updated list of commands like in the example below.

.. code-block:: console

   nscrub-cli
   localhost:8880> help

nscrub-export
-------------

*nscrub-export* can be used to dump/backup the configuration/policies of a specific target (or all targets).

All configured targets can be listed with the list command:

.. code-block:: console

   nscrub-export list
   webserver1
   webserver2

In order to dump the configuration for a specific target run *nscrub-export* with the target name (or 'all' for all targets):

.. code-block:: console

   nscrub-export dump webserver1

In the example below, we backup and restore the configuration for a target using *nscrub-export* in combination with *nscrub-cli*:

.. code-block:: console

   nscrub-export dump webserver1 > webserver1.dump
   cat webserver1.dump | nscrub-cli

