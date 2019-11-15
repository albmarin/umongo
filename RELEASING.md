# Releasing μMongo

## Prerequisites

- Install [bumpversion](https://pypi.org/project/bumpversion/). The easiest way is to create and activate a virtualenv,
  and then run ``pip install -r requirements_dev.txt``.

## Steps

1. Add an entry to ``HISTORY.rst``, or update the ``Unreleased`` entry, with the
   new version and the date of release. Include any bug fixes, features, or
   backwards incompatibilities included in this release.
2. Commit the changes to ``HISTORY.rst``.
3. Run [bumpversion](https://pypi.org/project/bumpversion/) to update the version string in ``umongo/__init__.py`` and
   ``setup.py``.

   * You can combine this step and the previous one by using the ``--allow-dirty``
     flag when running [bumpversion](https://pypi.org/project/bumpversion/) to make a single release commit.

4. Run ``git push`` to push the release commits to github.
5. Once the CI tests pass, run ``git push --tags`` to push the tag to github and
   trigger the release to pypi.

