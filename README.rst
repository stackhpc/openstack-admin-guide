Template of OpenStack administration guide
==========================================

Customise the guide
-------------------

* Fork this repository
* Customise source/vars.rst
* Customise source/data/deployment.yml

Build the guide
---------------

Prepare your build environment:

.. code-block:: console

   python3 -m venv venv
   source venv/bin/activate
   pip install -r requirements.txt

Then use one of the following build commands:

.. code-block:: console

   make html
   make singlehtml

Run ``make`` to see all possible builds.

PDF builds will require extra packages to be installed. For CentOS 8:

.. code-block:: console

   sudo dnf install -y epel-release

   sudo dnf install -y latexmk make texlive texlive-capt-of texlive-fncychap \
   texlive-framed texlive-needspace texlive-tabulary texlive-titlesec \
   texlive-upquote texlive-wrapfig

   make latexpdf
