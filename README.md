ABOUT
=====
This is a tool to convert Bugzilla bug reports to GitHub Issue Tracker
using Bugzilla XML export and GitHub REST API v3:<br/>
https://developer.github.com/v3/issues/

Converts Bugzilla emails to Github accounts, statuses to states,
Bugzilla components, priorities, severities, resolutions to GitHub labels.

Latest updates:

* 2018-12-22 -- added support for GitHub beta import API
  (activate with `-b` command line option)
* 2018-06-20 -- new better algorithm to post new and deleted bugs
* 2018-06-15 -- added support for deleted bugs (see below)


Notes on GitHub beta import API
-------------------------------
Thanks to Théo Zimmermann and Martin Michlmayr, bugzilla2github now supports
GitHub beta import API. Advantages of the new API:

* should import faster;
* does not trigger notifications for users;
* has a better support for bug comments.

The results of the beta API run is available here:
https://github.com/berestovskyy/bugzilla2github-demo-beta/issues

Note that beta API does not support issue update, so bugzilla2github falls back
to stable API in case of script interrupt.

More about GitHub beta import API:
https://gist.github.com/jonmagic/5282384165e0f86ef105


MANAGING CONFLICTS
------------------
The script deals with existing GitHub issues by renumbering Bugzilla
bug IDs.

Say, if there are already reported issues #1-10 on GitHub and there
are bugs #1-100 in Bugzilla. The script will leave untouched GitHub
issues #1-10, converts Bugzilla bugs #11-100 to GitHub issues #11-100
and then converts Bugzilla bugs #1-10 to GitHub issues #101-110.

At the end, for GitHub issues #1-10 script will add a comment, that
Bugzilla bugs #1-10 were renumbered to #101-110.


DELETED BUGS
------------
Github Issues Tracker allows assigning consecutive IDs only.
So bugzilla2github creates dummy issues in place of a deleted original
Bugzilla bugs.

Say, if there are bugs #1-10 in Bugzilla, then #20-30. The script will
import bugs #1-10 from Bugzilla, then will create dummy bug reports
to fill the IDs #11-19, and at the end will add the rest of Bugzilla
bugs #20-30.



QUICK START
===========
 1. Export Bugzilla issues into an XML file:
    - in Bugzilla select "Search"
    - select all statuses
    - at the very end click an XML icon
    - save the XML into a file

 2. Generate a GitHub access token:
    - on GitHub in top-right corner click on your avatar and then select "Settings"
    - in settings select "Developer settings"
    - select "Personal access tokens"
    - click "Generate new token"
    - type a token description, i.e. "bugzilla2github"
    - select "public_repo" to access just public repositories
    - save the generated token

 3. Dry-run the migration script and check all the warnings.
    **NOTE: Script dry-runs by default. No changes are posted to GitHub.**

    $ bugzilla2github -x export.xml -o GITHUB_OWNER -r GITHUB_REPO -t token

<pre>
    Where:
      -x  source XML file (exported from Bugzilla)
      -o  destination GitHub owner
      -r  destination GitHub repo
      -t  generated GitHub access token
</pre>

 4. Run the migration script again and force the updates adding `-f`:

    $ bugzilla2github -x export.xml -o GITHUB_OWNER -r GITHUB_REPO -t token -f

<pre>
    Where:
      -x  source XML file (exported from Bugzilla)
      -o  destination GitHub owner
      -r  destination GitHub repo
      -t  generated GitHub access token
      -f  force updates (i.e. post changes to GitHub)
</pre>


github2github
=============
github2github is a tool to copy GitHub Issues between repos/owners. Might be useful in some scenarios, when you first convert Bugzilla issues to a test repo and then copy them to a production repo.


DERIVATIVE TOOLS
================
The goal of this script is to provide easy to understand and debug
yet fully functional Bugzilla to Github Issues Tracker converter.

There are few tools based on this script. Each provides its
unique functionality.

Here is the list of tools based on this script you might find useful:

 * https://github.com/LibrePlan/bugzilla2github
 * https://github.com/erikmd/github-issues-import-api-tools
 * https://gist.github.com/Zimmi48/d923e52f64fe17c72852d9c148bfcdc6#file-bugzilla2github


EXAMPLE
=======
Dry-run:

    $ bugzilla2github -x export.xml -o berestovskyy -r bugzilla2github-demo -t TOKEN
    ===> Converting Bugzilla Bugs to GitHub Issues...
        Source XML file:    export.xml
        Dest. GitHub owner: berestovskyy
        Dest. GitHub repo:  bugzilla2github-demo
    WARNING: unable to convert email to GitHub login: email2@example.org
    ===> Checking if all the labels exist on GitHub...
    WARNING: label 'tests' does not exist on GitHub
        creating label 'tests' on GitHub...
    Skipping POST... (use -f to force updates)
        label 'enhancement' exists on GitHub
        label 'invalid' exists on GitHub
        label 'bug' exists on GitHub
    ===> Checking if all the assignees exist on GitHub...
        assignee 'berestovskyy' exists on GitHub
    ===> Posting Bugzilla reports to GitHub Issue Tracker...
    Checking GitHub issue #1...
        issue #1 already exists on GitHub, skipping...
    Checking GitHub issue #2...
        adding Bugzilla Bug 2 as GitHub issue #2...
        adding a new issue #2 on GitHub...
        updating issue #2 on GitHub...
    Checking GitHub issue #3...
        renumbering Bugzilla Bug 1 to GitHub issue #3...
        adding a new issue #3 on GitHub...
        updating issue #3 on GitHub...
        updating comment on GitHub issue #1...
        no comment found on #1, adding a new one...
        adding a comment to issue #1 on GitHub...
    Checking GitHub issue #4...
        adding a dummy (deleted) bug as #4...
        adding a new issue #4 on GitHub...
        updating issue #4 on GitHub...
    Checking GitHub issue #5...
        adding a dummy (deleted) bug as #5...
        adding a new issue #5 on GitHub...
        updating issue #5 on GitHub...
    Checking GitHub issue #6...
        adding a dummy (deleted) bug as #6...
        adding a new issue #6 on GitHub...
        updating issue #6 on GitHub...
    Checking GitHub issue #7...
        adding Bugzilla Bug 7 as GitHub issue #7...
        adding a new issue #7 on GitHub...
        updating issue #7 on GitHub...
    ===> All done.


Post changes to GitHub:

    $ bugzilla2github -x export.xml -o berestovskyy -r bugzilla2github-demo -t TOKEN -f
    WARNING: the repo will be UPDATED! No backups, no undos!
    Press Ctrl+C within next 5 seconds to cancel the update:
    ===> Converting Bugzilla Bugs to GitHub Issues...
        Source XML file:    export.xml
        Dest. GitHub owner: berestovskyy
        Dest. GitHub repo:  bugzilla2github-demo
    WARNING: unable to convert email to GitHub login: email2@example.org
    ===> Checking if all the labels exist on GitHub...
    WARNING: label 'tests' does not exist on GitHub
        creating label 'tests' on GitHub...
        label 'enhancement' exists on GitHub
        label 'invalid' exists on GitHub
        label 'bug' exists on GitHub
    ===> Checking if all the assignees exist on GitHub...
        assignee 'berestovskyy' exists on GitHub
    ===> Posting Bugzilla reports to GitHub Issue Tracker...
    Checking GitHub issue #1...
        issue #1 already exists on GitHub, skipping...
    Checking GitHub issue #2...
        adding Bugzilla Bug 2 as GitHub issue #2...
        adding a new issue #2 on GitHub...
        updating issue #2 on GitHub...
    Checking GitHub issue #3...
        renumbering Bugzilla Bug 1 to GitHub issue #3...
        adding a new issue #3 on GitHub...
        updating issue #3 on GitHub...
        updating comment on GitHub issue #1...
        no comment found on #1, adding a new one...
        adding a comment to issue #1 on GitHub...
    Checking GitHub issue #4...
        adding a dummy (deleted) bug as #4...
        adding a new issue #4 on GitHub...
        updating issue #4 on GitHub...
    Checking GitHub issue #5...
        adding a dummy (deleted) bug as #5...
        adding a new issue #5 on GitHub...
        updating issue #5 on GitHub...
    Checking GitHub issue #6...
        adding a dummy (deleted) bug as #6...
        adding a new issue #6 on GitHub...
        updating issue #6 on GitHub...
    Checking GitHub issue #7...
        adding Bugzilla Bug 7 as GitHub issue #7...
        adding a new issue #7 on GitHub...
        updating issue #7 on GitHub...
    ===> All done.


Final results
-------------
The results of the example run is available here:
https://github.com/berestovskyy/bugzilla2github-demo/issues
