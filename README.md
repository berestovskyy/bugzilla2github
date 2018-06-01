ABOUT
=====
This is a tool to convert Bugzilla bug reports to GitHub Issue Tracker.

Converts Bugzilla emails to Github accounts, statuses to states,
Bugzilla components, priorities, severities, resolutions to GitHub labels.

MANAGING CONFLICTS
------------------
The script deals with existing GitHub issues by renumbering Bugzilla
bug IDs. Say if there are already reported issues #1-10 on GitHub
and there are bugs #1-100 in Bugzilla.

The script will leave untouched GitHub issues #1-10, converts Bugzilla
bugs #11-100 to GitHub issues #11-100 and then converts Bugzilla
bugs #1-10 to GitHub issues #101-110.

At the end, for GitHub issues #1-10 script will add a comment, that
Bugzilla bugs #1-10 were renumbered to #101-110.


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

 3. Dry-run the migration script and check all the warnings. **NOTE: Script dry-runs by default. No changes are posted to GitHub.**

    $ bugzilla2github -x export.xml -o github_owner -r github_repo -t token

<pre>
    Where:
      -x  source XML file (exported from Bugzilla)
      -o  destination GitHub owner
      -r  destination GitHub repo
      -t  generated GitHub access token
</pre>

 4. Run the migration script again and force the updates adding `-f`:

    $ bugzilla2github -x export.xml -o github_owner -r github_repo -t token -f

<pre>
    Where:
      -x  source XML file (exported from Bugzilla)
      -o  destination GitHub owner
      -r  destination GitHub repo
      -t  generated GitHub access token
      -f  force updates (i.e. post changes to GitHub)
</pre>

EXAMPLE RUN
===========
Dry-run:

    $ bugzilla2github -x export.xml -o github_owner -r github_repo -t access_token
    ===> Converting Bugzilla reports to GitHub Issues...
    	Source XML file:     export.xml
	Dest. GitHub owner: github_owner
	Dest. GitHub repo:  github_repo
    WARNING: unable to convert email to GitHub login: email@gmail.com
    ===> Checking Bugzilla bug IDs are continuous...
    ===> Checking all the labels exist on GitHub...
	label 'enhancement' exists on GitHub
	label 'invalid' exists on GitHub
	label 'bug' exists on GitHub
    ===> Checking all the assignees exist on GitHub...
    ===> Adding Bugzilla reports on GitHub preserving IDs...
    Creating issue #1...
    	appending a new issue on GitHub...
    Skipping POST... (use -f to force updates)
    Creating issue #2...
    	appending a new issue on GitHub...
    Done adding with 0 issues postponed.
    ===> Appending postponed Bugzilla reports on GitHub with new IDs...
    Checking issue #3 on GitHub...
	issue #3 does not exist, breaking...
    ===> All done.

Post changes to GitHub:

    $ bugzilla2github -x export.xml -o github_owner -r github_repo -t access_token -f
    WARNING: the repo will be UPDATED! No backups, no undos!
    Press Ctrl+C within next 5 secons to cancel the update:
    ===> Converting Bugzilla reports to GitHub Issues...
    	Source XML file:    export.xml
	Dest. GitHub owner: github_owner
	Dest. GitHub repo:  github_repo
    WARNING: unable to convert email to GitHub login: email@gmail.com
    ===> Checking Bugzilla bug IDs are continuous...
    ===> Checking all the labels exist on GitHub...
    	label 'enhancement' exists on GitHub
    	label 'invalid' exists on GitHub
    	label 'bug' exists on GitHub
    ===> Checking all the assignees exist on GitHub...
    ===> Adding Bugzilla reports on GitHub preserving IDs...
    Creating issue #1...
    	appending a new issue on GitHub...
    	updating issue #1 on GitHub...
    Creating issue #2...
    	appending a new issue on GitHub...
    	updating issue #2 on GitHub...
    Done adding with 0 issues postponed.
    ===> Appending postponed Bugzilla reports on GitHub with new IDs...
    Checking issue #3 on GitHub...
    	issue #3 does not exist, breaking...
    ===> All done.


github2github
=============
github2github is a tool to copy GitHub Issues between repos/owners. Might be useful in some scenarios, when you first convert Bugzilla issues to a test repo and then copy them to a production repo.
