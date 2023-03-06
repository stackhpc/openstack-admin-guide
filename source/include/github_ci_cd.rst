OpenStack CI/CD
===============

Structure
---------

There is a workflow defined for the most commonly used kayobe commands. Some of the mappings are shown below:

.. list-table:: Workflows
   :widths: 15 20
   :header-rows: 1

   * - Kayobe command
     - Github workflow
   * - kayobe overcloud host configure
     - `Run overcloud host configure <https://github.com/rug-cit-hpc/rug-kayobe-config/actions/workflows/run-overcloud-host-configure.yml>`_
   * - kayobe overcloud host package update
     - `Run overcloud host package update <https://github.com/rug-cit-hpc/rug-kayobe-config/actions/workflows/run-overcloud-host-package-update.yml>`_
   * - kayobe overcloud service deploy
     - `Run overcloud service deploy <https://github.com/rug-cit-hpc/rug-kayobe-config/actions/workflows/run-overcloud-service-deploy.yml>`_

This list is not extensive, however the same pattern applies for other `kayobe commands  <https://docs.openstack.org/kayobe/latest/administration/index.html>`_. If the workflow is missing you will need to create a new workflow.

Walkthroughs
------------

A similar process can be used to trigger all Github workflows. For that reason, we will only walkthrough a select few. You can apply the same process to the other workflows.

Walkthrough: Running network connectivity check
-----------------------------------------------

* Navigate to the `Run network connectivity check <https://github.com/rug-cit-hpc/rug-kayobe-config/actions/workflows/run-network-connectivity-check.yml>`_ workflow:

.. image:: _static/github-network-connectivity.png
  :width: 600
  :alt: Run network connectivity

* Click on the ``Run Workflow``  dropdown to expose the job parameters. Select the correct branch you wish to target, then click ``Run workflow``:

.. image:: _static/github-run-workflow.png
  :width: 300
  :alt: Run workflow

* Click on one of the completed runs to see the jobs in the workflow:

.. image:: _static/github-workflow-jobs.png
  :width: 600
  :alt: Workflow jobs

* Click on the job to see the job output:

.. image:: _static/github-job-output.png
  :width: 300
  :alt: Job output

* Click on any of the steps to expand the output for that step:

.. image:: _static/github-job-expanded.png
  :width: 400
  :alt: Expanding job output

* The kayobe output can be seen in the "Run network connectivity check" step.

Walkthrough: Tempest
--------------------

Tempest is a collection of tests that can be run against an OpenStack cloud. These test the various functionalities of the cloud for correct operation.

* Navigate to the `Run tempest <https://github.com/rug-cit-hpc/rug-kayobe-config/actions/workflows/run-tempest.yml>`_ workflow:

.. image:: _static/github-tempest.png
  :width: 600
  :alt: Run tempest

* Click on the ``Run Workflow``  dropdown to expose the job parameters:

.. image:: _static/github-tempest-run.png
  :width: 300
  :alt: Run workflow

* Select one of the available test lists. The following test sets are available:

.. list-table:: Test lists
   :widths: 5 20
   :header-rows: 1

   * - Test list
     - Description
   * - default
     - This is the same as refstack-2019.11-test-list.txt.
   * - refstack-2019.11-test-list.txt
     - This is the `refstack <https://refstack.openstack.org/#/>`_ test list to test OpenStack conformance. You need to
       pass this set to have an 'OpenStack Certified' cloud.
   * - tempest-full
     - This is a list of all tests included in the tempest repository, excluding those provided by tempest plugins.
   * - baremetal
     - This a hand selected test list to exercise Ironic. It can only be run in environments where Ironic has been enabled.

* To view the test results you need to download the archive produced by the upload-artifacts step:

.. image:: _static/github-tempest-archive.png
  :width: 600
  :alt: Test result archive

* Open the archive and view the ``rally-verify-report.html`` file in your web browser:

.. image:: _static/github-tempest-results.png
  :width: 600
  :alt: Test results

Generating a fresh kayobe image
-------------------------------

When to generate a fresh image:

- Updating kayobe version
- Installing new roles or collections
- Want to update OS packages in docker image

Tag a commit within the `rug-kayobe-config <https://github.com/rug-cit-hpc/rug-kayobe-config/>`_ repository. This will trigger a workflow that rebuilds the ``kayobe`` image. The tags follow
semantic versioning. Here is some guidance for when to bump each component:

- MAJOR: Breaking change e.g new major release of kayobe
- MINOR: Non-breaking change that adds new functionality
- PATCH: Bug fix or to update OS packages

Generalised semantic versioning guidelines can be found `here <https://semver.org/>`__.
