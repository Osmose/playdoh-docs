.. _packages:

==========================
pip and friends: Packaging
==========================

*(largely borrowed from Zamboni)*

There are two ways of getting packages in your playdoh-based project. The
first is to install everything using pip. We have our packages separated into
three files:

:src:`requirements/compiled.txt`
    All packages that require (or go faster with) compilation.  These can't be
    distributed cross-platform, so they need to be installed through your
    system's package manager or pip.

:src:`requirements/prod.txt`
    The minimal set of packages you need to run zamboni in production.  You
    also need to get ``requirements/compiled.txt``.

:src:`requirements/dev.txt`
    All the packages needed for running tests and development servers.  This
    automatically includes ``requirements/prod.txt``.


Installing through pip
----------------------

You can get a development environment with ::

    pip install -r requirements/dev.txt -r requirements/compiled.txt


Using the vendor library
------------------------

The other method is to use the /vendor library of all packages and
repositories.

By default, the vendor lib is checked out as a *git submodule* under
``vendor/``. If you *do* need to check it out separately, do::

    git clone --recursive git://github.com/mozilla/playdoh-lib.git ./vendor

Once the playdoh-lib repo has been downloaded to ``/vendor``, you only need to
install the compiled packages.  These can come from your system package manager
or from ::

    pip install -r requirements/compiled.txt


Adding new packages
-------------------

The vendor repo was seeded with ::

    pip install -I --install-option="--home=`pwd`/vendor" --src='vendor/src' -r requirements/dev.txt

    # ..delete some junk from vendor/lib/python...

    # Create the .pth file so Python can find our src libs.
    find src -type d -depth 1 >> zamboni.pth

    # Add all the submodules.
    for f in src/*; do
        pushd $f >/dev/null && REPO=$(git config remote.origin.url) && popd > /dev/null && git submodule add $REPO $f
    done
    git add .


If we wanted to add a new dependency called ``cheeseballs`` to zamboni, you
would add it to ``requirements/prod.txt`` or ``requirements/dev.txt``. This makes it available to
users installing into virtualenvs.

We also need to add the new package to the vendor lib.

First, you then need to update ``vendor/vendor.pth``. Python
uses ``.pth`` files to dynamically add directories to ``sys.path`` (`docs
<http://docs.python.org/library/site.html>`_).

I created ``vendor.pth`` with this::

    find src/ -type d -depth 1 > vendor.pth

Secondly, we need to add the source. There are two ways, depending on how
this project is hosted:

For non-git based repos (hg, CVS, tarball) do::

    pip install -I --install-option="--home=`pwd`/vendor" --src='vendor/src' cheeseballs
    cd vendor
    git add src/cheeseballs
    git commit vendor.pth src/cheeseballs

For a git-based package, add it as a git submodule::

    cd vendor
    git submodule add git://github.com/mozilla/cheeseballs.git src/cheeseballs
    git commit vendor.pth .gitmodules src/cheeseballs

Some packages (like ``html5lib`` and ``selenium``) are troublesome, because
their source lives inside an extra subdirectory ``src/`` inside their checkout.
So they need to be sourced with ``src/html5lib/src``, for example. Hopefully
you won't hit any snags like that.


Adding lots of git submodules
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

As noted in Adding new packages, you want to make your git-based packages *git submodules* inside the vendor
library. To set up the first batch of submodules, something like the following
happened::

    for f in src/*
        pushd $f && REPO=$(git config remote.origin.url) && popd && git submodule add $REPO $f
