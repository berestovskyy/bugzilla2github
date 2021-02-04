#!/usr/bin/env python
# -*- coding: utf-8 -*-
#
# Copy GitHub Issues between repos/owners
# by Andriy Berestovskyy <berestovskyy@gmail.com>
#
# To generate a GitHub access token:
#    - on GitHub select "Settings"
#    - select "Personal access tokens"
#    - click "Generate new token"
#    - type a token description, i.e. "bugzilla2issues"
#    - select "public_repo" to access just public repositories
#    - save the generated token into the migration script
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
import time

if sys.version[0] == '2':
    reload(sys)
    sys.setdefaultencoding('utf-8')

force_update = False
github_url = "https://api.github.com"
src_github_owner = ""
src_github_repo = ""
src_github_token = ""

dst_github_owner = ""
dst_github_repo = ""
dst_github_token = ""


def usage():
    print("Copy GitHub Issues between repos/owners")
    print("Usage: %s [-h] [-f]\n"
          "\t[-O <src GitHub owner>] [-R <src repo>] [-T <src access token>]\n"
          "\t[-o <dst GitHub owner>] [-r <dst repo>] [-t <dst access token>]\n"
          % os.path.basename(__file__))
    print("Example:")
    print("\t%s -h" % os.path.basename(__file__))
    print("\t%s -O src_login -R src_repo -T src_token \\\n"
          "\t\t\t\t-o dst_login -r dst_repo -t dst_token" % os.path.basename(__file__))
    exit(1)


def src_github_get(url, avs={}):
    global github_url, src_github_owner, src_github_repo, src_github_token

    if url[0] == "/":
        u = "%s%s" % (github_url, url)
    elif url.startswith("https://"):
        u = url
    elif url.startswith("http://"):
        u = url
    else:
        u = "%s/repos/%s/%s/%s" % (github_url,
                                   src_github_owner, src_github_repo, url)

    # TODO: debug
    # print "SRC GET: " + u

    avs["access_token"] = src_github_token
    return requests.get(u, params=avs)


def dst_github_get(url, avs={}):
    global github_url, dst_github_owner, dst_github_repo, dst_github_token

    if url[0] == "/":
        u = "%s%s" % (github_url, url)
    elif url.startswith("https://"):
        u = url
    elif url.startswith("http://"):
        u = url
    else:
        u = "%s/repos/%s/%s/%s" % (github_url,
                                   dst_github_owner, dst_github_repo, url)

    # TODO: debug
    # print "DST GET: " + u

    avs["access_token"] = dst_github_token
    return requests.get(u, params=avs)


def dst_github_post(url, avs={}, fields=[]):
    global force_update
    global github_url, dst_github_owner, dst_github_repo, dst_github_token

    if url[0] == "/":
        u = "%s%s" % (github_url, url)
    else:
        u = "%s/repos/%s/%s/%s" % (github_url,
                                   dst_github_owner, dst_github_repo, url)

    d = {}
    # Copy fields into the data
    for field in fields:
        if field not in avs:
            print("Error posting filed %s to %s" % (field, url))
            exit(1)
        d[field] = avs[field]

    # TODO: debug
    # print "DST POST: " + u
    # print "DST DATA: " + json.dumps(d)

    if force_update:
        return requests.post(u, params={"access_token": dst_github_token},
                             data=json.dumps(d))
    else:
        if not dst_github_post.warn:
            print("Skipping POST... (use -f to force updates)")
            dst_github_post.warn = True
        return True


dst_github_post.warn = False


def dst_github_issue_exist(number):
    if dst_github_get("issues/%d" % number):
        return True
    else:
        return False


def dst_github_issue_update(issue):
    id = issue["number"]

    print("\tupdating issue #%d on GitHub..." % id)
    r = dst_github_post("issues/%d" % id, issue,
                        ["title", "body", "state", "labels", "assignees"])
    if not r:
        print("Error updating issue #%d on GitHub:\n%s" % (id, r.headers))
        exit(1)


def dst_github_issue_append(issue):
    print("\tappending a new issue on GitHub...")
    r = dst_github_post(
        "issues", issue, ["title", "body", "labels", "assignees"])
    if not r:
        print("Error appending an issue on GitHub:\n%s" % r.headers)
        exit(1)
    return r


def github_issues_copy():
    id = 0
    while True:
        id += 1
        print("Copying issue #%d..." % id)
        issue = src_github_get("issues/%d" % id)
        if issue:
            issue = issue.json()
            if dst_github_issue_exist(id):
                if force_update:
                    dst_github_issue_update(issue)
                else:
                    print("\tupdating issue #%d..." % id)
            else:
                if force_update:
                    # Make sure the previous issue already exist
                    if id > 1 and not dst_github_issue_exist(id - 1):
                        print("Error adding issue #%d: previous issue does not exists"
                              % id)
                        exit(1)
                    req = dst_github_issue_append(issue)
                    new_issue = dst_github_get(req.headers["location"]).json()
                    if new_issue["number"] != id:
                        print("Error adding issue #%d: assigned unexpected issue id #%d"
                              % (id, new_issue["number"]))
                        exit(1)
                    # Update issue state
                    if issue["state"] != "open":
                        dst_github_issue_update(issue)
                else:
                    print("\tadding new issue #%d..." % id)
        else:
            print("Done.")
            break


def args_parse(argv):
    global force_update
    global src_github_owner, src_github_repo, src_github_token
    global dst_github_owner, dst_github_repo, dst_github_token

    try:
        opts, args = getopt.getopt(argv, "hfo:r:t:O:R:T:")
    except getopt.GetoptError:
        usage()
    for opt, arg in opts:
        if opt == '-h':
            usage()
        elif opt == "-f":
            print("WARNING: the repo will be UPDATED! No backups, no undos!")
            print("Press Ctrl+C within next 5 secons to cancel the update:")
            time.sleep(5)
            force_update = True
        elif opt == "-o":
            dst_github_owner = arg
        elif opt == "-r":
            dst_github_repo = arg
        elif opt == "-t":
            dst_github_token = arg
            if not src_github_token:
                src_github_token = arg
        elif opt == "-O":
            src_github_owner = arg
        elif opt == "-R":
            src_github_repo = arg
        elif opt == "-T":
            src_github_token = arg
            if not dst_github_token:
                dst_github_token = arg

    # Check the arguments
    if (not src_github_owner or not src_github_repo or not src_github_token
            or not dst_github_owner or not dst_github_repo or not dst_github_token):
        print("Error parsing arguments: "
              "please specify source and destination GitHub owner, repo and token")
        usage()


def main(argv):
    global src_github_owner, src_github_repo
    global dst_github_owner, dst_github_repo

    # Parse command line arguments
    args_parse(argv)
    print("===> Copying GitHub Issues between repos/owners...")
    print("\tsource GitHub owner: %s" % src_github_owner)
    print("\tsource GitHub repo:  %s" % src_github_repo)
    print("\tdest.  GitHub owner: %s" % dst_github_owner)
    print("\tdest.  GitHub repo:  %s" % dst_github_repo)

    github_issues_copy()


if __name__ == "__main__":
    main(sys.argv[1:])
