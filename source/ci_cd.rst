.. include:: vars.rst

========================================================
Continuous Integration (CI) and Continuous Delivery (CD)
========================================================

.. ifconfig:: deployment['ci_cd'] == 'github'

   The |project_name| cloud includes CI/CD provided by GitHub Actions.

   .. include:: include/github_ci_cd.rst

.. ifconfig:: deployment['ci_cd'] == 'none'

   The |project_name| cloud does not include CI/CD.
