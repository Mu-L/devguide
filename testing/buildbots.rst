.. _buildbots:

======================
Working with buildbots
======================

.. highlight:: bash

To assert that there are no regressions in the :ref:`development and maintenance
branches <devcycle>`, Python has a set of dedicated machines (called *buildbots*
or *build workers*) used for continuous integration.  They span a number of
hardware/operating system combinations.  Furthermore, each machine hosts
several *builders*, one per active branch: when a new change is pushed
to this branch on the public `GitHub repository <https://github.com/python/cpython>`_, all corresponding builders
will schedule a new build to be run as soon as possible.

The build steps run by the buildbots are the following:

* Check out the source tree for the change which triggered the build
* Compile Python
* Run the test suite using :ref:`strenuous settings <strenuous_testing>`
* Clean up the build tree

It is your responsibility, as a core developer, to check the automatic
build results after you push a change to the repository.  It is therefore
important that you get acquainted with the way these results are presented,
and how various kinds of failures can be explained and diagnosed.

In case of trouble
==================

Please read this page in full. If your questions aren't answered here and you
need assistance with the buildbots, a good way to get help is to either:

* contact the ``python-buildbots@python.org`` mailing list where all buildbot
  worker owners are subscribed; or
* contact the release manager of the branch you have issues with.

Buildbot failures on pull requests
==================================

The ``bedevere-bot`` on GitHub will put a message on your merged Pull Request
if building your commit on a stable buildbot worker fails. Take care to
evaluate the failure, even if it looks unrelated at first glance.

Not all failures will generate a notification since not all builds are executed
after each commit. In particular, reference leaks builds take several hours to
complete so they are done periodically. This is why it's important for you to
be able to check the results yourself, too.

Triggering on pull requests
===========================

To trigger buildbots on a pull request you need to be a CPython triager or a
core team member. If you are not, ask someone to trigger them on your behalf.

The simplest way to trigger most buildbots on your PR is with the
:gh-label:`🔨 test-with-buildbots` and :gh-label:`🔨 test-with-refleak-buildbots`
labels. (See :ref:`github-pr-labels`.)

These will run buildbots on the most recent commit. If you want to trigger the
buildbots again on a later commit, you'll have to remove the label and add it
again.

If you want to test a pull request against specific platforms, you can trigger
one or more build bots by posting a comment that begins with:

.. code-block:: none

   !buildbot regex-matching-target

For example to run both the iOS and Android build bot, you can use:

.. code-block:: none

   !buildbot ios|android

bedevere-bot will post a comment indicating which build bots, if
any, were matched. If none were matched, or you do not have the
necessary permissions to trigger a request, it will tell you that too.

The ``!buildbot`` comment will also only run buildbots on the most recent
commit. To trigger the buildbots again on a later commit, you will have to
repeat the comment.

Checking results of automatic builds
====================================

There are three ways of visualizing recent build results:

* The Web interface for each branch at https://www.python.org/dev/buildbot/,
  where the so-called "waterfall" view presents a vertical rundown of recent
  builds for each builder.  When interested in one build, you'll have to
  click on it to know which commits it corresponds to.  Note that
  the buildbot web pages are often slow to load, be patient.

* The command-line ``bbreport.py`` client, which you can get from
  https://code.google.com/archive/p/bbreport. Installing it is trivial: just add
  the directory containing ``bbreport.py`` to your system path so that
  you can run it from any filesystem location.  For example, if you want
  to display the latest build results on the development ("main") branch,
  type::

      bbreport.py -q 3.x

* The buildbot "console" interface at https://buildbot.python.org/all/
  This works best on a wide, high resolution
  monitor.  Clicking on the colored circles will allow you to open a new page
  containing whatever information about that particular build is of interest to
  you.  You can also access builder information by clicking on the builder
  status bubbles in the top line.

If you like IRC, having an IRC client open to the #python-dev-notifs channel on
irc.libera.chat is useful.  Any time a builder changes state (last build
passed and this one didn't, or vice versa), a message is posted to the channel.
Keeping an eye on the channel after pushing a commits is a simple way to get
notified that there is something you should look in to.

Some buildbots are much faster than others.  Over time, you will learn which
ones produce the quickest results after a build, and which ones take the
longest time.

Also, when several commits are pushed in a quick succession in the same
branch, it often happens that a single build is scheduled for all these
commits.

Stability
=========

A subset of the buildbots are marked "stable".  They are taken into account
when making a new release.  The rule is that all stable builders must be free of
persistent failures when the release is cut.  It is absolutely **vital**
that core developers fix any issue they introduce on the stable buildbots,
as soon as possible.

This does not mean that other builders' test results can be taken lightly,
either.  Some of them are known for having platform-specific issues that
prevent some tests from succeeding (or even terminating at all), but
introducing additional failures should generally not be an option.

Flags-dependent failures
========================

Sometimes, while you have run the :ref:`whole test suite <runtests>` before
committing, you may witness unexpected failures on the buildbots.  One source
of such discrepancies is if different flags have been passed to the test runner
or to Python itself.  To reproduce, make sure you use the same flags as the
buildbots: they can be found out simply by clicking the **stdio** link for
the failing build's tests.  For example::

   ./python.exe -Wd -E -bb  ./Lib/test/regrtest.py -uall -rwW

.. note::
   Running ``Lib/test/regrtest.py`` is exactly equivalent to running
   ``-m test``.

Ordering-dependent failures
===========================

Sometimes the failure is even subtler, as it relies on the order in which
the tests are run.  The buildbots *randomize* test order (by using the ``-r``
option to the test runner) to maximize the probability that potential
interferences between library modules are exercised; the downside is that it
can make for seemingly sporadic failures.

The ``--randseed`` option makes it easy to reproduce the exact randomization
used in a given build.  Again, open the ``stdio`` link for the failing test
run, and check the beginning of the test output proper.

Let's assume, for the sake of example, that the output starts with:

.. code-block:: none
   :emphasize-lines: 6

   ./python -Wd -E -bb Lib/test/regrtest.py -uall -rwW
   == CPython 3.3a0 (default:22ae2b002865, Mar 30 2011, 13:58:40) [GCC 4.4.5]
   ==   Linux-2.6.36-gentoo-r5-x86_64-AMD_Athlon-tm-_64_X2_Dual_Core_Processor_4400+-with-gentoo-1.12.14 little-endian
   ==   /home/buildbot/buildarea/3.x.ochtman-gentoo-amd64/build/build/test_python_29628
   Testing with flags: sys.flags(debug=0, inspect=0, interactive=0, optimize=0, dont_write_bytecode=0, no_user_site=0, no_site=0, ignore_environment=1, verbose=0, bytes_warning=2, quiet=0)
   Using random seed 2613169
   [  1/353] test_augassign
   [  2/353] test_functools

You can reproduce the exact same order using::

   ./python -Wd -E -bb -m test -uall -rwW --randseed 2613169

It will run the following sequence (trimmed for brevity):

.. code-block:: none

   [  1/353] test_augassign
   [  2/353] test_functools
   [  3/353] test_bool
   [  4/353] test_contains
   [  5/353] test_compileall
   [  6/353] test_unicode

If this is enough to reproduce the failure on your setup, you can then
bisect the test sequence to look for the specific interference causing the
failure.  Copy and paste the test sequence in a text file, then use the
``--fromfile`` (or ``-f``) option of the test runner to run the exact
sequence recorded in that text file::

   ./python -Wd -E -bb -m test -uall -rwW --fromfile mytestsequence.txt

In the example sequence above, if ``test_unicode`` had failed, you would
first test the following sequence:

.. code-block:: none

   [  1/353] test_augassign
   [  2/353] test_functools
   [  3/353] test_bool
   [  6/353] test_unicode

And, if it succeeds, the following one instead (which, hopefully, shall
fail):

.. code-block:: none

   [  4/353] test_contains
   [  5/353] test_compileall
   [  6/353] test_unicode

Then, recursively, narrow down the search until you get a single pair of
tests which triggers the failure.  It is very rare that such an interference
involves more than **two** tests.  If this is the case, we can only wish you
good luck!

.. note::
   You cannot use the ``-j`` option (for parallel testing) when diagnosing
   ordering-dependent failures.  Using ``-j`` isolates each test in a
   pristine subprocess and, therefore, prevents you from reproducing any
   interference between tests.


Transient failures
==================

While we try to make the test suite as reliable as possible, some tests do
not reach a perfect level of reproducibility.  Some of them will sometimes
display spurious failures, depending on various conditions.  Here are common
offenders:

* Network-related tests, such as ``test_poplib``, ``test_urllibnet``, etc.
  Their failures can stem from adverse network conditions, or imperfect
  thread synchronization in the test code, which often has to run a
  server in a separate thread.

* Tests dealing with delicate issues such as inter-thread or inter-process
  synchronization, or Unix signals: ``test_multiprocessing``,
  ``test_threading``, ``test_subprocess``, ``test_threadsignals``.

When you think a failure might be transient, it is recommended you confirm by
waiting for the next build.  Still, even if the failure does turn out sporadic
and unpredictable, the issue should be reported on the bug tracker; even
better if it can be diagnosed and suppressed by fixing the test's
implementation, or by making its parameters - such as a timeout - more robust.

.. seealso::
   :ref:`buildworker`
