========================
Team and repository tags
========================

.. image:: https://governance.openstack.org/tc/badges/mistral.svg
    :target: https://governance.openstack.org/tc/reference/tags/index.html

Mistral
=======

Workflow Service for OpenStack cloud. This project aims to provide a mechanism
to define tasks and workflows without writing code, manage and execute them in
the cloud environment.

Installation
~~~~~~~~~~~~

The following are the steps to install Mistral on debian-based systems.

To install Mistral, you have to install the following prerequisites::

 $ apt-get install python-dev python-setuptools libffi-dev \
   libxslt1-dev libxml2-dev libyaml-dev libssl-dev

**Mistral can be used without authentication at all or it can work with
OpenStack.**

In case of OpenStack, it works **only with Keystone v3**, make sure **Keystone
v3** is installed.


Install Mistral
---------------

First of all, clone the repo and go to the repo directory::

  $ git clone https://git.openstack.org/openstack/mistral.git
  $ cd mistral


**Devstack installation**

Information about how to install Mistral with devstack can be found
`here <https://docs.openstack.org/mistral/latest/contributor/devstack.html>`_.

Configuring Mistral
~~~~~~~~~~~~~~~~~~~

Mistral configuration is needed for getting it work correctly with and without
an OpenStack environment.

#. Install and configure a database which can be *MySQL* or *PostgreSQL*
   (**SQLite can't be used in production.**). Here are the steps to connect
   Mistral to a *MySQL* database.

   * Make sure you have installed ``mysql-server`` package on your Mistral
     machine.
   * Install *MySQL driver* for python::

     $ pip install mysql-python

     or, if you work in virtualenv, run::

     $ tox -evenv -- pip install mysql-python

     NOTE: If you're using Python 3 then you need to install ``mysqlclient``
     instead of ``mysql-python``.

   * Create the database and grant privileges::

      $ mysql -u root -p
      mysql> CREATE DATABASE mistral;
      mysql> USE mistral
      mysql> GRANT ALL PRIVILEGES ON mistral.* TO 'mistral'@'localhost' \
             IDENTIFIED BY 'MISTRAL_DBPASS';
      mysql> GRANT ALL PRIVILEGES ON mistral.* TO 'mistral'@'%' IDENTIFIED BY 'MISTRAL_DBPASS';

#. Generate ``mistral.conf`` file::

    $ oslo-config-generator --config-file tools/config/config-generator.mistral.conf \
      --output-file etc/mistral.conf.sample

#. Copy service configuration files::

    $ sudo mkdir /etc/mistral
    $ sudo chown `whoami` /etc/mistral
    $ cp etc/event_definitions.yml.sample /etc/mistral/event_definitions.yml
    $ cp etc/logging.conf.sample /etc/mistral/logging.conf
    $ cp etc/policy.json /etc/mistral/policy.json
    $ cp etc/wf_trace_logging.conf.sample /etc/mistral/wf_trace_logging.conf
    $ cp etc/mistral.conf.sample /etc/mistral/mistral.conf

#. Edit file ``/etc/mistral/mistral.conf`` according to your setup. Pay attention
   to the following sections and options::

    [oslo_messaging_rabbit]
    rabbit_host = <RABBIT_HOST>
    rabbit_userid = <RABBIT_USERID>
    rabbit_password = <RABBIT_PASSWORD>

    [database]
    # Use the following line if *PostgreSQL* is used
    # connection = postgresql://<DB_USER>:<DB_PASSWORD>@localhost:5432/mistral
    connection = mysql://<DB_USER>:<DB_PASSWORD>@localhost:3306/mistral

#. If you are not using OpenStack, add the following entry to the
   ``/etc/mistral/mistral.conf`` file and **skip the following steps**::

    [pecan]
    auth_enable = False

#. Provide valid keystone auth properties::

    [keystone_authtoken]
    auth_uri = http://keystone-host:port/v3
    auth_url = http://keystone-host:port
    auth_type = password
    username = <user>
    password = <password>
    user_domain_name = <usually 'Default'>
    project_name = <project name>
    project_domain_name = <usually 'Default'>

#. Register Mistral service and Mistral endpoints on Keystone::

    $ MISTRAL_URL="http://[host]:[port]/v2"
    $ openstack service create --name mistral workflowv2
    $ openstack endpoint create mistral public $MISTRAL_URL
    $ openstack endpoint create mistral internal $MISTRAL_URL
    $ openstack endpoint create mistral admin $MISTRAL_URL

#. Update the ``mistral/actions/openstack/mapping.json`` file which contains
   all available OpenStack actions, according to the specific client versions
   of OpenStack projects in your deployment. Please find more detailed
   information in the ``tools/get_action_list.py`` script.

Before the First Run
--------------------

After local installation you will find the commands ``mistral-server`` and
``mistral-db-manage`` available in your environment. The ``mistral-db-manage``
command can be used for migrating database schema versions. If Mistral is not
installed in system then this script can be found at
``mistral/db/sqlalchemy/migration/cli.py``, it can be executed using Python
command line.

To update the database schema to the latest revision, type::

  $ mistral-db-manage --config-file <path_to_config> upgrade head

To populate the database with standard actions and workflows, type::
  $ mistral-db-manage --config-file <path_to_config> populate

For more detailed information about ``mistral-db-manage`` script please check
file ``mistral/db/sqlalchemy/migration/alembic_migrations/README.md``.

Running Mistral API server
--------------------------

To run Mistral API server::

  $ tox -evenv -- python mistral/cmd/launch.py --server api --config-file <path_to_config>

Running Mistral Engines
-----------------------

To run Mistral Engine::

  $ tox -evenv -- python mistral/cmd/launch.py --server engine --config-file <path_to_config>

Running Mistral Task Executors
------------------------------

To run Mistral Task Executor instance::

  $ tox -evenv -- python mistral/cmd/launch.py --server executor --config-file <path_to_config>

Note that at least one Engine instance and one Executor instance should be
running in order for workflow tasks to be processed by Mistral.

If you want to run some tasks on specific executor, the *task affinity* feature
can be used to send these tasks directly to a specific executor. You can edit
the following property in your mistral configuration file for this purpose::

    [executor]
    host = my_favorite_executor

After changing this option, you will need to start (restart) the executor. Use
the ``target`` property of a task to specify the executor::

    ... Workflow YAML ...
    task1:
      ...
      target: my_favorite_executor
    ... Workflow YAML ...

Running Multiple Mistral Servers Under the Same Process
-------------------------------------------------------

To run more than one server (API, Engine, or Task Executor) on the same
process::

  $ tox -evenv -- python mistral/cmd/launch.py --server api,engine --config-file <path_to_config>

The value for the ``--server`` option can be a comma-delimited list. The valid
options are ``all`` (which is the default if not specified) or any combination
of ``api``, ``engine``, and ``executor``.

It's important to note that the ``fake`` transport for the ``rpc_backend``
defined in the configuration file should only be used if ``all`` Mistral
servers are launched on the same process. Otherwise, messages do not get
delivered because the ``fake`` transport is using an in-process queue.

Project Goals 2017
------------------

#. **Complete Mistral documentation**.

   Mistral documentation should be more usable. It requires focused work to
   make it well structured, eliminate gaps in API/Mistral Workflow Language
   specifications, add more examples and tutorials.

   *Definition of done*:
   All capabilities are covered, all documentation topics are written using
   the same style and structure principles. The obvious sub-goal of this goal
   is to establish these principles.

#. **Complete Mistral Custom Actions API**.

   There has been the initiative in Mistral team since April of 2016 to
   refactor Mistral actions subsystem in order to make the process of
   developing Mistral actions easier and clearer. In 2017 we need to complete
   this effort and make sure that all APIs are stable and it’s well-documented.

   *Definition of done*:
   All API interfaces are stable, existing actions are rewritten using this new
   API, OpenStack actions are also rewritten based on the new API and moved to
   mistral-extra repo. Everything is well documented and the doc has enough
   examples.

#. **Finish Mistral multi-node mode**.

   Mistral needs to be proven to work reliably in multi-node mode. In order
   to achieve it we need to make a number of engine, executor and RPC
   changes and configure a CI gate to run stress tests on multi-node Mistral.

   *Definition of done*:
   CI gate supports MySQL, all critically important functionality (join,
   with-items, parallel workflows, sequential workflows) is covered by tests.

#. **Reduce workflow execution time**.

   *Definition of done*: Average workflow execution time reduced by 30%.

Project Resources
-----------------

* `Mistral Official Documentation <https://docs.openstack.org/mistral/latest/>`_

* Project status, bugs, and blueprints are tracked on
  `Launchpad <https://launchpad.net/mistral/>`_

* Additional resources are linked from the project
  `Wiki <https://wiki.openstack.org/wiki/Mistral/>`_ page

* Apache License Version 2.0 http://www.apache.org/licenses/LICENSE-2.0
