.. include:: vars.rst

==========================================
Verifying the Cloud with Rally and Tempest
==========================================

`Rally <https://docs.openstack.org/rally/latest/>`_ is a test framework,
and `Tempest <https://docs.openstack.org/tempest/latest/>`_ is the
OpenStack API test suite.  In this guide, Rally is used to run Tempest tests.

Requirements
------------

OpenStack tests are run from a host with access to the OpenStack APIs
and external network for instances.

The following software environment is needed:

* OpenStack admin credentials, eg public-openrc.sh from kayobe-config/etc/kolla
* A virtualenv setup with python-openstackclient installed

Setup Rally for a new user
--------------------------

A good directory hierarchy would be `~/rally/shakespeare/tempest-recipes`.
This is assumed in this guide.

Install Rally into the virtualenv:

.. code-block:: shell

   pip3 install git+https://github.com/openstack/rally-openstack.git \
        --constraint https://raw.githubusercontent.com/openstack/rally-openstack/master/upper-constraints.txt

Create the Rally test database and configuration file:

.. code-block:: shell

   mkdir -p ~/.rally ~/rally/data
   echo "[database]" | tee ~/.rally/rally.conf
   echo "connection=sqlite:////home/${USER}/rally/data/rally.db" | tee -a ~/.rally/rally.conf
   rally db recreate
   rally verify create-verifier --name default --type tempest
   rally deployment create --fromenv --name production

Check:

.. code-block:: shell

   rally deployment show

Install Shakespeare
-------------------

Shakespeare is used for writing Tempest test configuration.

.. code-block:: shell

   git clone https://github.com/stackhpc/shakespeare.git
   cd shakespeare
   pip3 install -r requirements.txt

Install Tempest Recipe
----------------------

A custom set of Tempest recipes should be maintained for |project_name|,
defining key parameters needed and fine-grained control on the test cases
to run (and which to skip).

In your `shakespeare` directory,
install the Tempest recipes in your Tempest configuration.

.. code-block:: shell
   :substitutions:

   git clone |tempest_recipes|
   ansible-playbook template.yml -e @tempest-recipes/production.yml
   mkdir -p  ../config/production
   rally verify configure-verifier --reconfigure --extend ../config/production/production.conf

Rally invocation
----------------

Invoke the tests (this will take several hours to complete):

.. code-block:: shell

   rally --debug verify start --concurrency 1 --skip-list tempest-recipes/production-skiplist.yml

Report generation
-----------------

Generate an HTML report of the results:

.. code-block:: shell

   rally verify report --type html --to ~/rally/report-$(date -d "today" +"%Y%m%d%H%M").html
