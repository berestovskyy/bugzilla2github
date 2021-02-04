#!/usr/bin/env python
# -*- coding: utf-8 -*-
#
# SPDX-License-Identifier: Apache-2.0
# Copyright (c) 2018 Andriy Berestovskyy <berestovskyy@gmail.com>
#
# Bugzilla XML File to GitHub Issues Converter
#
# How to use the script:
# 1. Export Bugzilla issues into an XML file:
#    - in Bugzilla select "Search"
#    - select all statuses
#    - at the very end click an XML icon
#    - save the XML into a file
#
# 2. Generate a GitHub access token:
#    - on GitHub select "Settings"
#    - select "Developer settings"
#    - select "Personal access tokens"
#    - click "Generate new token"
#    - type a token description, i.e. "bugzilla2github"
#    - select "public_repo" to access just public repositories
#    - save the generated token
#
# 3. Run the migration script and check all the warnings:
#    bugzilla2github -x export.xml -o GITHUB_OWNER -r GITHUB_REPO -t TOKEN
#
# 4. Run the migration script again and force the updates:
#    bugzilla2github -x export.xml -o GITHUB_OWNER -r GITHUB_REPO -t TOKEN -f
#

from __future__ import absolute_import
from __future__ import print_function
import json
import getopt
import os
import pprint
import re
import requests
import sys
import datetime
import time
import xml.etree.ElementTree
from six.moves import range

if sys.version[0] == '2':
    reload(sys)
    sys.setdefaultencoding('utf-8')

########################################################################
## Global variables
########################################################################

# By default use stable GitHub APIv3.
# Use -b command line argument for GitHub beta import API.
USE_BETA_API = False
# Beta import API application
GITHUB_BETA_APP = "application/vnd.github.golden-comet-preview+json"
# Script name, i.e. bugzilla2github
NAME = os.path.basename(__file__)
# This signature is used to detect if the bug was imported from Bugzilla
# The signature follows with the bug id, i.e. an integer number.
BUG_SIGNATURE = "Bugzilla Bug "
DELETED_BUG_SIGNATURE = "Deleted Bugzilla Bug"
# Do the actual changes, i.e. post to GitHub
FORCE_UPDATES = False
# Print some debug info
DEBUG = False
# Default XML file with Bugzilla export
XML_FILE = "export.xml"
# GitHub REST API URL
# See https://developer.github.com/v3/issues/
GITHUB_API = "https://api.github.com"
# GitHub repository owner, i.e. username
GITHUB_OWNER = ""
# GitHub repository name
GITHUB_REPO = ""
# GitHub access token
GITHUB_TOKEN = ""

# Global convertion dictionaries
EMAIL2LOGIN = {
    "__name__": "email to GitHub login",
    "123@example.com": "login",
    "email@example.org": "berestovskyy",
}
STATUS2STATE = {
    "__name__": "status to GitHub state (stable APIv3)",
    "NEW": "open",
    "UNCONFIRMED": "open",
    "CONFIRMED": "open",
    "VERIFIED": "open",
    "ASSIGNED": "open",
    "IN_PROGRESS": "open",
    "RESOLVED": "closed",
    "CLOSED": "closed",
    "REOPENED": "open",
}
STATE2CLOSED_BETA = {
    "__name__": "stable API state to beta API closed",
    "open": False,
    "closed": True,
}
COMPONENT2LABELS = {
    "__name__": "component to GitHub labels",
    "Features": ["enhancement"],
    "Testing": ["tests"],
    "Documentation": ["docs"],
}
PRIORITY2LABELS = {
    "__name__": "priority to GitHub labels",
    "Lowest": ["low priority"],
    "Low": ["low priority"],
    "---": [],
    "Normal": [],
    "High": ["high priority"],
    "Highest": ["high priority"],
}
SEVERITY2LABELS = {
    "__name__": "severity to GitHub labels",
    "enhancement": ["enhancement"],
    "trivial": ["minor", "bug"],
    "minor": ["minor", "bug"],
    "normal": ["bug"],
    "major": ["major", "bug"],
    "critical": ["major", "bug"],
    "blocker": ["major", "bug"],
}
RESOLUTION2LABELS = {
    "__name__": "resolution to GitHub labels",
    "FIXED": [],
    "INVALID": ["invalid"],
    "DUPLICATE": ["duplicate"],
}
BUG_UNUSED_FIELDS = [
    "actual_time",
    "attachment.isobsolete",
    "attachment.ispatch",
    "attachment.isprivate",
    "cclist_accessible",
    "classification",
    "classification_id",
    "comment_sort_order",
    "estimated_time",
    "everconfirmed",
    "long_desc.isprivate",
    "op_sys",
    "product",
    "remaining_time",
    "reporter_accessible",
    "rep_platform",
    "target_milestone",
    "token",
    "version",
]
COMMENT_UNUSED_FIELDS = [
    "comment_count",
]
ATTACHMENT_UNUSED_FIELDS = [
    "attacher",
    "attacher.name",
    "date",
    "delta_ts",
    "token",
]


########################################################################
## GitHub API functions
########################################################################


def github_issues_add(issues):
    """
    Adds dictionary of converted issues to GitHub Issue tracker,
    i.e. this is the main script logic.

    Args:
        issues: Dictionary of issues converted from Bugzilla export.
    """
    id = 0
    while issues:
        id += 1
        print("Checking GitHub issue #%d..." % id)
        github_issue = github_issue_get(id)
        if github_issue:
            bug_id = bug_id_parse(github_issue["body"])
            if bug_id > 0:
                print("\tissue #%d is the Bug %d, updating..." % (id, bug_id))
                issue = issues.pop(bug_id, None)
                github_issue_update(id, issue)
            elif bug_id == 0:
                print("\tissue #%d is a deleted Bugzilla Bug, updating...")
                issue = new_issue(id, True)
                github_issue_update(id, issue)
            else:
                print("\tissue #%d already exists on GitHub, skipping..." % id)
                continue
        else:
            # Try to add Bug id to the same #id
            issue = issues.pop(id, None)
            if issue:
                print("\tadding Bugzilla Bug %d as GitHub issue #%d..."
                      % (id, id))
            else:
                # Or just add a next issue
                first_id = sorted(issues)[0]
                if first_id > id:
                    # We are not there yet, just add a dummy
                    print("\tadding a dummy (deleted) bug as #%d..." % id)
                    issue = new_issue(id, True)
                else:
                    # Otherwise, just add postponed issue
                    print("\trenumbering Bugzilla Bug %d to GitHub issue #%d..."
                          % (first_id, id))
                    issue = issues.pop(first_id)
            # Note: appending an issue does not guarantee it will become #id
            github_issue_add(id, issue)


def github_issue_get(id):
    """
    Gets issue #id from GitHub.

    Args:
        id: GitHub #id to get.

    Returns:
        A json-encoded issue or None if issue #id does not exist on GitHub.
    """
    req = github_get("issues/%d" % id)
    if req:
        return req.json()
    else:
        return None


def github_issue_update(id, issue):
    """
    Updates issue #id on GitHub.

    Args:
        id: GitHub #id to update.
        issue: Converted Bugzilla Bug to be updated.

    Returns:
        A response object on success or exits with error otherwise.
    """
    print("\tupdating issue #%d on GitHub..." % id)
    if not issue:
        print("Error updating #%d: Bugzilla Bug is missing" % id)
        exit(1)

    # TODO: There is no update functionality in beta API,
    # so we use stable API to update the issue
    r = github_post("issues/%d" % id, issue,
                    ["title", "body", "state", "labels", "assignees"])
    if not r:
        print("Error updating issue #%d on GitHub:\n%s" % (id, r.text))
        exit(1)

    orig_id = issue["id"]
    if id != orig_id:
        github_comment_update(orig_id, id)


def github_issue_add_stable(id, issue):
    # Note: this still does not guarantee the new issue will be added as #id
    r = github_post("issues", issue, ["title", "body", "labels", "assignees"])
    if not r:
        print("Error adding a new issue on GitHub:\n%s" % r.text)
        exit(1)

    if FORCE_UPDATES:
        # Make sure the issue is added as #id
        github_issue = github_issue_get(id)
        if not github_issue:
            print("Error adding issue: issue %d does not exist" % id)
            exit(1)
        bug_id = bug_id_parse(github_issue["body"])
        if bug_id and bug_id != issue["id"]:
            print("Error adding issue: issue %d is not %d"
                  % (bug_id, issue["id"]))
            exit(1)

    # Default state for the newly created issues is open, so we need to update
    # the state and the renumbering comment.
    github_issue_update(id, issue)

    return r


def github_issue_add_beta(id, issue):
    # Note: this still does not guarantee the new issue will be added as #id

    # Convert issue to beta import API
    comments = issue.pop("comments", [])
    assignees = issue.pop("assignees", [])
    if len(assignees) > 0:
        issue["assignee"] = assignees[0]
    old_id = issue.pop("id", 0)
    issue_comments = {"issue": issue, "comments": comments}

    r = github_post("import/issues", issue_comments, ["issue", "comments"])
    if not r:
        print("Error importing a new issue on GitHub:\n%s" % r.text)
        exit(1)

    # Convert issue back to stable API
    issue["id"] = old_id
    issue["assignees"] = assignees

    if FORCE_UPDATES:
        # Wait until the issue is imported
        u = r.json()["url"]
        wait = 1
        r = False
        while not r or r.json()["status"] == "pending":
            time.sleep(wait)
            wait = 2 * wait
            r = github_get(u)
        if not r.json()["status"] == "imported":
            print("Error importing issue on GitHub:\n%s" % r.text)
            exit(1)

        # The issue_url field of the answer should be of the form
        # .../ISSUE_NUMBER so it's easy to get the issue number,
        # to check that it is what was expected.
        result = re.match("https://api.github.com/repos/" + GITHUB_OWNER +
                          "/" + GITHUB_REPO + "/issues/(\d+)",
                          r.json()["issue_url"])
        if not result:
            print("Error while parsing issue number:\n%s" %
                  r.json()["issue_url"])
            exit(1)
        issue_number = int(result.group(1))
        if issue_number != id:
            print("Error adding issue: issue %d is not %d"
                  % (issue_number, id))
            exit(1)

    # Need to update the renumbering comment
    github_issue_update(id, issue)

    return r


def github_issue_add(id, issue):
    """
    Adds issue #id to GitHub. GitHub does not allow direct issue id
    assignment, so prior to call this function, a previous issue (i.e. issue
    #id-1) MUST exist on GitHub and the issue to add (i.e. issue #id) MUST NOT
    exist on GitHub.

    Still, if someone in parallel adds an issue on GitHub, there is
    no guarantee the newly added issue will be added as #id.

    Please make sure no one else is modifying the issues at the same time
    you run this script!

    Args:
        is: GitHub #id to be added.
            Note: GitHub does not allow to add a specific #id, so this #id
                  MUST not exist on GitHub
        issue: Converted Bugzilla Bug to be added.
    """
    print("\tadding a new issue #%d on GitHub..." % id)

    if USE_BETA_API:
        return github_issue_add_beta(id, issue)
    else:
        return github_issue_add_stable(id, issue)


def github_get(url, params={}):
    """
    Sends GET request to GitHub or a specified URL.

    Args:
        url: A string with an URL to get. The URL might start with:
             '/' -- the request is sent to main GitHub API URL + url string
             'http://' or 'https://' -- the request is sent to the specified url
             otherwise the request is sent to:
                GitHub API URL + /repos/ + GitHub Owner + GitHub Repo + url
        params: An optional dictionary to be sent in the query.
        headers: An optional dictionary to be sent in the headers.

    Returns:
        A response object.
    """
    if url[0] == "/":
        u = GITHUB_API + url
    elif url.startswith("https://"):
        u = url
    elif url.startswith("http://"):
        u = url
    else:
        u = "%s/repos/%s/%s/%s" % (GITHUB_API, GITHUB_OWNER, GITHUB_REPO, url)

    if USE_BETA_API:
        params = {}
        headers = {
            "Accept": GITHUB_BETA_APP,
            "Authorization": "token " + GITHUB_TOKEN,
        }
    else:
        params = {"access_token": GITHUB_TOKEN}
        headers = {}

    if DEBUG:
        print("DEBUG GET: " + u)
        # It's insecure to dump the token
        # print("DEBUG PARAMS: " + json.dumps(params))
        # print("DEBUG HEADERS: " + json.dumps(headers))

    return requests.get(u, params=params, headers=headers)


def github_post(url, dict={}, fields=[]):
    """
    Sends POST request to a specified URL. If global flag FORCE_UPDATES
    is not set, the function prints a warning and just returns True.

    Args:
        url: A string with an URL to post. The URL might start with:
             '/' -- the request is sent to main GitHub API URL + url string
             otherwise the request is sent to:
                GitHub API URL + /repos/ + GitHub Owner + GitHub Repo + url
        dict: An optional dictionary to be posted.
        fields: An optional list of fields to send from the dictionary.

    Returns:
        A response object. If the global flag FORCE_UPDATES is not set, always
        returns True.
    """
    if url[0] == "/":
        u = "%s%s" % (GITHUB_API, url)
    else:
        u = "%s/repos/%s/%s/%s" % (GITHUB_API, GITHUB_OWNER, GITHUB_REPO, url)

    d = {}
    # Copy fields into the data
    for field in fields:
        if field not in dict:
            print("Error posting field '%s' to %s" % (field, url))
            exit(1)
        d[field] = dict[field]

    if USE_BETA_API:
        params = {}
        headers = {
            "Accept": GITHUB_BETA_APP,
            "Authorization": "token " + GITHUB_TOKEN,
        }
    else:
        params = {"access_token": GITHUB_TOKEN}
        headers = {}
    data = json.dumps(d)
    if DEBUG:
        print("DEBUG POST: " + u)
        # It's insecure to dump the token
        # print("DEBUG PARAMS: " + json.dumps(params))
        # print("DEBUG HEADERS: " + json.dumps(headers))
        print("DEBUG DATA: " + json.dumps(d))

    if FORCE_UPDATES:
        return requests.post(u, params=params, headers=headers, data=data)
    else:
        if not github_post.POST_WARNING_PRINTED:
            print("Skipping POST... (use -f to force updates)")
            github_post.POST_WARNING_PRINTED = True
        return True


# A flag if the post warning has been printed
github_post.POST_WARNING_PRINTED = False


def github_comment_add(orig_id, new_id):
    """
    Adds a comment to Github #orig_id that Bugzilla Bug is now at #new_id.

    Args:
        orig_id: Original Bugzilla Bug id.
        new_id: Newly assigned GitHub Issue #id.
    """
    print("\tadding a comment to issue #%d on GitHub..." % orig_id)
    comment = new_renumber_comment(orig_id, new_id)
    r = github_post("issues/%d/comments" % orig_id, comment, ["body"])
    if not r:
        print("Error adding new comment to issue #%d on GitHub:\n%s"
              % (orig_id, r.text))
        exit(1)


def github_comment_update(orig_id, new_id):
    """
    Updates a comment on Github #orig_id that Bugzilla Bug is now at #new_id.

    Args:
        orig_id: Original Bugzilla Bug id.
        new_id: Newly assigned GitHub Issue #id.
    """
    print("\tupdating comment on GitHub issue #%d..." % orig_id)
    r = github_get("issues/%d/comments" % orig_id)
    if r:
        comments = r.json()
        for comment in comments:
            bug_id = bug_id_parse(comment["body"])
            if not bug_id:
                continue
            comment_id = int(comment["id"])
            print("\tupdating comment %d on GitHub issue #%d..."
                  % (comment_id, orig_id))
            comment = new_renumber_comment(orig_id, new_id)
            r = github_post("issues/comments/%d" %
                            comment_id, comment, ["body"])
            if not r:
                print("Error updating comment %d on GitHub issue #%d:\n%s"
                      % (comment_id, orig_id, r.text))
                exit(1)
            return
    print("\tno comment found on #%d, adding a new one..." % orig_id)
    github_comment_add(orig_id, new_id)


def github_label_create(label):
    """
    Creates a new label on GitHub.

    Args:
        label: String with new label name.
    """
    print("\tcreating label '%s' on GitHub..." % label)
    r = github_post("labels", {
        "name": label,
        "color": "0"*6,
    }, ["name", "color"])
    if not r:
        print("Error creating label %s: %s" % (label, r.text))
        exit(1)


def github_labels_create(issues):
    """
    Checks if all the labels assigned to all the issues are on GitHub.
    If FORCE_UPDATES flag is set, creates the missing labels.

    Args:
        issues: A dictionary of converted Bugzilla reports.
    """
    labels_set = set()
    for id in issues:
        for label in issues[id]["labels"]:
            labels_set.add(label)

    for label in labels_set:
        if github_get("labels/" + label):
            print("\tlabel '%s' exists on GitHub" % label)
        else:
            print("WARNING: label '%s' does not exist on GitHub" % label)
            github_label_create(label)


def github_assignees_check(issues):
    """
    Checks if all the users assigned to all the issues are on GitHub.

    Args:
        issues: A dictionary of converted Bugzilla reports.
    """
    a_set = set()
    for id in issues:
        for assignee in issues[id]["assignees"]:
            a_set.add(assignee)

    for assignee in a_set:
        if not github_get("/users/" + assignee):
            print("Error checking user '%s' on GitHub" % assignee)
            exit(1)
        else:
            print("\tassignee '%s' exists on GitHub" % assignee)


########################################################################
## XML parsing and conversion functions
########################################################################


def XML2bug(parent):
    """
    Converts XML nodes to a new Bugzilla bug report dictionary.

    Args:
        parent: A parent XML node to start the conversion.

    Returns:
        A new dictionary containing Bugzilla bug report.
    """
    issue = {}

    if DEBUG:
        print("DEBUG XML:")
    for key in parent:
        if len(key) > 0:
            if DEBUG:
                print("\t> %s" % key.tag)
            val = XML2bug(key)
        else:
            val = key.text
        if key.text:
            if key.tag not in issue:
                issue[key.tag] = val
                if DEBUG:
                    print("\tissue['%s'] = '%s'" % (key.tag, val))
            else:
                if isinstance(issue[key.tag], list):
                    issue[key.tag].append(val)
                    if DEBUG:
                        print("\tissue['%s'].append('%s')" % (key.tag, val))
                else:
                    issue[key.tag] = [issue[key.tag], val]
                    if DEBUG:
                        print("\tissue['%s'] = [issue['%s'], '%s']" %
                              (key.tag, key.tag, val))
        # Parse attributes
        for name, val in key.items():
            issue["%s.%s" % (key.tag, name)] = val
            if DEBUG:
                print("\tissue['%s.%s'] = '%s'" % (key.tag, name, val))

    return issue


def str2list(map, str):
    """Lookups in the map specific string and returns it as a list."""
    if str not in map:
        print("WARNING: unable to convert %s: %s" % (map["__name__"], str))
        # Suppress further reports
        map[str] = []

    return map[str]


def ids2ids(ids):
    """Converts Bugzilla bug report ids to issues #ids."""
    issue = []

    if not ids:
        return ""
    if isinstance(ids, list):
        for id in ids:
            issue.append("#" + id)
    else:
        issue.append("#" + ids)

    return ", ".join(issue)


def email2login(email, name):
    """Converts email to a GitHub user login string."""
    issue = str2list(EMAIL2LOGIN, email)
    if issue:
        return "@" + issue
    else:
        if name and not name.find("@") >= 0:
            return "%s &lt;<%s>&gt;" % (name, email)
        else:
            return email


def emails2logins(emails):
    """Converts an email or a list of emails to a string."""
    issue = []
    if isinstance(emails, list):
        for email in emails:
            issue.append(email2login(email, None))
    else:
        issue.append(email2login(emails, None))
    return ", ".join(issue)


def fields_ignore(obj, fields):
    """Ignores (removes) all the fields in the objects."""
    # Ignore some Bugzilla fields
    for field in fields:
        obj.pop(field, None)


def fields_dump(obj):
    """Dumps all the fields in the object."""
    # Make sure we have converted all the fields
    for key, val in obj.items():
        print("\t%s[%d] = %s" % (key, len(val), val))


def attachment2str(idx, attach):
    """Converts Bugzilla attachment to a string."""
    issue = []

    id = attach.pop("attachid")
    issue.append("> Attached file: %s (%s, %s bytes)"
                 % (attach.pop("filename"),
                    attach.pop("type"),
                    attach.pop("size")))
    issue.append("> Description:   " + attach.pop("desc"))

    # Ignore some fields
    fields_ignore(attach, ATTACHMENT_UNUSED_FIELDS)
    # Make sure we have converted all the fields
    if attach:
        print("WARNING: unconverted attachment fields:")
        fields_dump(attach)

    idx[id] = "\n".join(issue)


def attachments2dict(attachments):
    """Converts Bugzilla attachments to a dictionary of strings."""
    issue = {}
    if isinstance(attachments, list):
        for attachment in attachments:
            attachment2str(issue, attachment)
    else:
        attachment2str(issue, attachments)

    return issue


def bugzilladate2githubdate(date):
	    result = re.match(
	        r'(\d\d\d\d-\d\d-\d\d) (\d\d:\d\d:\d\d) \+(\d\d)(\d\d)', date)
	    if not result:
	        print("Date %s was not converted!" % date)
	        exit(1)
	    return "{a}T{b}+{c}:{d}".format(a=result.group(1), b=result.group(2),
	                                    c=result.group(3), d=result.group(4))


def comment2dict(comment, attachments):
    """
    Converts Bugzilla comment to a string (stable APIv3)
    or a dictionary (beta import API).
    """
    body = []

    bug_when = comment.pop("bug_when")
    body.extend([
        "## Comment " + comment.pop("commentid"),
        "Date: " + bug_when,
        "From: " + email2login(comment.pop("who"),
                               comment.pop("who.name", None)),
        "",
        comment.pop("thetext", ""),
        "",
    ])
    # Convert attachments if any
    if "attachid" in comment:
        body.extend([
            attachments.pop(comment.pop("attachid")),
            ""
        ])
    body.append("")

    # Syntax: convert "bug id" to "bug #id"
    for i, val in enumerate(body):
        body[i] = re.sub(r"(?i)(bug)\s+([0-9]+)", r"\1 #\2", val)

    # Ignore some comment fields
    fields_ignore(comment, COMMENT_UNUSED_FIELDS)
    # Make sure we have converted all the fields
    if comment:
        print("WARNING: unconverted comment fields:")
        fields_dump(comment)

    if USE_BETA_API:
        created_at = bugzilladate2githubdate(bug_when)
        # Return a dictionary (beta import API)
        return {"body": "\n".join(body), "created_at": created_at}
    else:
        # Return a string (stable APIv3)
        return "\n".join(body)


def comments2list(comments, attachments):
    """
    Converts Bugzilla comments to a list of
    strings (stable APIv3) or dictionaries (beta import API).
    """
    issue = []
    if isinstance(comments, list):
        for comment in comments:
            issue.append(comment2dict(comment, attachments))
    else:
        issue.append(comment2dict(comments, attachments))

    return issue


def bug2issue(bug):
    """
    Converts all Bugzilla report fields to a new issue dictionary.
    Dumps any unconverted fields, so all the fields must be either converted
    or listed as unused.

    Args:
        bug: A Bugzilla bug report dictionary.

    Returns:
        A new issue dictionary.
    """
    issue = new_issue(bug.pop("bug_id"), False)
    attachments = {}

    # Convert attachments if any
    if "attachment" in bug:
        attachments = attachments2dict(bug.pop("attachment"))
    # Convert long_desc and attachment to comments
    issue["comments"].extend(comments2list(bug.pop("long_desc"), attachments))
    # Convert short_desc to title
    issue["title"] = ("%s (" % bug.pop("short_desc") + BUG_SIGNATURE
                      + "%d)" % issue["id"])
    # Convert reporter to user login
    user_login = email2login(bug.pop("reporter"),
                             bug.pop("reporter.name", None))
    # Convert assigned_to to assignees
    issue["assignees"].append(
        email2login(bug.pop("assigned_to"), bug.pop("assigned_to.name", None)))
    # Convert component to labels
    issue["labels"].extend(str2list(COMPONENT2LABELS, bug.pop("component")))
    # Convert bug_status to state
    issue["state"] = str2list(STATUS2STATE, bug.pop("bug_status"))
    if USE_BETA_API:
        # Convert creation_ts to created_at
        issue["created_at"] = bugzilladate2githubdate(bug.pop("creation_ts"))
        # Convert delta_ts to updated_at
        issue["updated_at"] = bugzilladate2githubdate(bug.pop("delta_ts"))
        # Convert bug_status to closed
        issue["closed"] = str2list(STATE2CLOSED_BETA, issue["state"])
        if issue["closed"]:
            issue["closed_at"] = issue["updated_at"]
    else:
        # Convert creation_ts to created_at
        issue["created_at"] = bug.pop("creation_ts")
        # Convert delta_ts to updated_at
        issue["updated_at"] = bug.pop("delta_ts")
        issue["user.login"] = user_login

    # Convert priority to labels
    issue["labels"].extend(str2list(PRIORITY2LABELS, bug.pop("priority")))
    # Convert severity to labels
    issue["labels"].extend(str2list(SEVERITY2LABELS, bug.pop("bug_severity")))
    # Convert resolution to labels
    if "resolution" in bug:
        issue["labels"].extend(
            str2list(RESOLUTION2LABELS, bug.pop("resolution")))

    # Create the bug description
    body = []
    body.extend([
        "# " + BUG_SIGNATURE + "%d" % issue["id"],
        "Date: " + issue["created_at"],
        "From: " + user_login,
        "To:   " + ", ".join(issue["assignees"]),
    ])
    if "cc" in bug:
        body.append("CC:   " + emails2logins(bug.pop("cc")))
    # Extra information
    body.append("")
    if "dup_id" in bug:
        body.append("Duplicates:   " + ids2ids(bug.pop("dup_id")))
    if "dependson" in bug:
        body.append("Depends on:   " + ids2ids(bug.pop("dependson")))
    if "blocked" in bug:
        body.append("Blocker for:  " + ids2ids(bug.pop("blocked")))
    if "see_also" in bug:
        if isinstance(bug["see_also"], list):
            for see_also in bug.pop("see_also"):
                body.append("See also:     " + see_also)
        else:
            body.append("See also:     " + bug.pop("see_also"))
    body.append("Last updated: " + issue["updated_at"])
    body.append("")

    # Put everything together
    if USE_BETA_API:
        issue["body"] += "\n".join(body)
    else:
        issue["body"] += "\n".join(body) + "\n\n" + \
            "\n".join(issue["comments"])
    issue["assignees"] = [a[1:] for a in issue["assignees"] if a[0] == "@"]
    # Get unique labels
    labels_set = set(issue["labels"])
    issue["labels"] = list(labels_set)

    # Ignore some bug fields
    fields_ignore(bug, BUG_UNUSED_FIELDS)
    # Make sure we have converted all the fields
    if bug:
        print("WARNING: unconverted bug fields:")
        fields_dump(bug)
    # Make sure we have converted all the attachments
    if attachments:
        print("WARNING: unused attachments:")
        fields_dump(attachments)

    return issue


def bugs2dict(xml_root):
    """
    Converts XML root node to a dictionary of issues.

    Args:
        xml_root: The root XML element.

    Returns:
        A new dictionary of converted Bugzilla reports.
    """
    issues = {}
    for xml_bug in xml_root.iter("bug"):
        bug = XML2bug(xml_bug)
        issue = bug2issue(bug)
        # Check for duplicates
        id = issue["id"]
        if id in issues:
            print("Error checking duplicates: #%d is duplicated in '%s'"
                  % (id, XML_FILE))
        issues[id] = issue

    return issues


########################################################################
## Helper functions
########################################################################


def new_issue(id, dummy):
    """
    Creates a new issue.

    Args:
        id: The id of a new issue.
        dummy: Set True to create a new dummy issue.

    Returns:
        A dictionary representing an issue with title, body, labels, assignees,
        comments, id etc.
    """
    issue = {}
    body = []
    issue["labels"] = []
    issue["assignees"] = []
    issue["comments"] = []
    issue["id"] = int(id)
    issue["title"] = DELETED_BUG_SIGNATURE
    issue["created_at"] = datetime.datetime.utcnow().replace(
        microsecond=0).isoformat() + "Z"
    issue["updated_at"] = issue["created_at"]
    issue["state"] = "closed"
    if USE_BETA_API:
        issue["closed"] = True
    else:
        issue["user.login"] = "bugzilla2github"

    body.extend([
        "This issue was created automatically with %s" % NAME,
        "",
    ])
    if (dummy):
        body.extend([
            "# " + DELETED_BUG_SIGNATURE,
            "Date: " + issue["created_at"],
            "",
            "This bug was created during the import from Bugzilla.",
            "GitHub Issues Tracker allows to assign consecutive",
            "ids only. So this dummy issue was created in place of",
            "a deleted Bugzilla Bug.",
            "",
            "Please ignore this report.",
            "",
        ])
    issue["body"] = "\n".join(body)

    return issue


def new_renumber_comment(orig_id, new_id):
    """
    Creates a comment to renumber Bugzilla Bug orig_id to an issue #new_id.

    Args:
        orig_id: Original Bugzilla bug id.
        new_id: New issue #id.

    Returns:
        A dictionary with comment in the "body", i.e. { "body": "comment" }
    """
    body = []
    body.extend([
        "This comment was created automatically with %s tool" % NAME,
        "## NOTE: " + BUG_SIGNATURE + "%d" % orig_id
        + " is renumbered to issue #%d" % new_id,
        "If you are looking for a Bugzilla Bug %d," % orig_id
        + " please follow issue #%d instead." % new_id,
    ])
    comment = {
        "body": "\n".join(body)
    }
    return comment


def bug_id_parse(str):
    """
    Parses Bugzilla Bug id in a string.

    Args:
        str: String to parse Bugzilla Bug id (i.e. issue body or title)

    Returns:
        Integer id, 0 for the deleted bug or None if Bug id is not found.
    """
    idx = str.find(NAME)
    if idx == -1:
        return None
    bug_id = re.search(r"(?ims)" + BUG_SIGNATURE + "([0-9]+)", str)
    if bug_id:
        return int(bug_id.group(1))
    else:
        idx = str.find(DELETED_BUG_SIGNATURE)
        if idx != -1:
            return 0  # bug_id is zero for the deleted bugs
        else:
            return None


def usage():
    """
    Displays usage information and exits.
    """
    print("Bugzilla XML export file to GitHub Issues Converter")
    print("Usage: %s [-b] [-h] [-f] [-d]" % NAME)
    print("\t[-x <src XML file>]")
    print("\t-o <GitHub owner> -r <GitHub repo> -t <GitHub access token>")
    print("Where:")
    print("\t-b  use GitHub beta import API")
    print("\t-h  usage information")
    print("\t-f  force updates (actually post the changes to GitHub)")
    print("\t-d  print debug information")
    print("Example:")
    print("\t%s -h" % NAME)
    print("\t%s -x export.xml -o github_login -r GITHUB_REPO -t token" % NAME)
    exit(1)


def args_parse(argv):
    """
    Parses command line options and sets global variables accordingly.
    """
    global FORCE_UPDATES, DEBUG, USE_BETA_API
    global XML_FILE, GITHUB_OWNER, GITHUB_REPO, GITHUB_TOKEN

    try:
        opts, args = getopt.getopt(argv, "ehfdo:r:t:x:")
    except getopt.GetoptError:
        usage()
    for opt, arg in opts:
        if opt == '-e':
            print(
                "WARNING: Using GitHub beta import API. "
                "Please report issues here:\n"
                "\t https://github.com/berestovskyy/bugzilla2github/issues")
            USE_BETA_API = True
        if opt == '-h':
            usage()
        elif opt == "-f":
            print(
                "WARNING: The repo will be UPDATED! No backups, no undos!\n"
                "Press Ctrl+C within next 5 seconds to cancel the update:")
            time.sleep(5)
            FORCE_UPDATES = True
        elif opt == "-d":
            DEBUG = True
        elif opt == "-o":
            GITHUB_OWNER = arg
        elif opt == "-r":
            GITHUB_REPO = arg
        elif opt == "-t":
            GITHUB_TOKEN = arg
        elif opt == "-x":
            XML_FILE = arg

    # Check the arguments
    if (not XML_FILE or not GITHUB_OWNER or not GITHUB_REPO or
            not GITHUB_TOKEN):
        print("Error parsing arguments: "
              "please specify XML file, GitHub owner, repo and token")
        print()
        usage()


def main(argv):
    """
    Steps to convert Bugzilla exported bugs and post them on GitHub.
    """
    args_parse(argv)
    print("===> Converting Bugzilla Bugs to GitHub Issues...")
    print("\tSource XML file:          %s" % XML_FILE)
    print("\tDestination GitHub owner: %s" % GITHUB_OWNER)
    print("\tDestination GitHub repo:  %s" % GITHUB_REPO)
    if DEBUG:
        print("\tDebug information:        on")
    if USE_BETA_API:
        print("\tUsing beta import API:    on")

    xml_tree = xml.etree.ElementTree.parse(XML_FILE)
    xml_root = xml_tree.getroot()
    issues = bugs2dict(xml_root)

    print("===> Checking if all the labels exist on GitHub...")
    github_labels_create(issues)
    print("===> Checking if all the assignees exist on GitHub...")
    github_assignees_check(issues)

    print("===> Posting Bugzilla reports to GitHub Issue Tracker...")
    github_issues_add(issues)
    print("===> All done.")


if __name__ == "__main__":
    main(sys.argv[1:])
