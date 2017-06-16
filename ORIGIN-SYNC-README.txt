This is a how-to for syncing a given tag from service-catalog and merging
it into the openshift/origin repository.


Prerequisite setup:
- git clone of service-catalog repo from https://github.com/kubernetes-incubator/service-catalog.git
- $ git remote add openshift git@github.com:openshift/service-catalog.git
- ensure there aren't any patches in the openshift/origin repo that need to be
  put in the openshift/service-catalog:origin-patches branch (git log
  cmd/service-catalog)

Because git allows you to set up your remotes as you wish, we'll define the
remotes by name and URL here:
(in service-catalog repo)
$ git remote -v
openshift	git@github.com:openshift/service-catalog.git (fetch)
openshift	git@github.com:openshift/service-catalog.git (push)
origin	https://github.com/kubernetes-incubator/service-catalog.git (fetch)
origin	https://github.com/kubernetes-incubator/service-catalog.git (push)


# download updates
$ git fetch origin
$ git fetch openshift

# syncs the openshift/service-catalog repo with the upstream tag
# (in service-catalog repo)
$ TAG=v0.0.10
$ git pull origin
$ git push openshift $TAG

# updates master branch in service-catalog repo (unused, but the master branch
# is the first thing people see.)
$ git rebase openshift/master $TAG
$ git push openshift openshift/master

# rebases origin-patches branch
$ git rebase openshift/origin-patches $TAG
$ git push openshift openshift/origin-patches --force-with-lease

[At this point, the service catalog origin remote is no longer needed.]


# If git log cmd/service-catalog in the origin repo is showing commits not in
# origin-patches, they must be brought over to the service-catalog
# origin-patches branch. Otherwise, skip this section.
(origin repo)
$ git pull
$ git log cmd/service-catalog
# If there are commits since the last rebase, not the earliest commit.
# Next descend in the catalog directories to the point that matches upstream,
# which is:
$ cd go/src/github.com/kubernetes-incubator/service-catalog
# Now generate the patches for all the commits (caret is important):
$ git format-patch 0d55d15c8a1b6dc16db77d176da9651e13382a48^ --relative

# EXAMPLE:
$ git log --oneline cmd/service-catalog/
d09188f636 (HEAD -> catalog-ldflags) a test
0d55d15c8a catalog: add build time version info
dc92bcfd23 Merge version v0.0.17 of Service Catalog from https://github.com/openshift/service-catalog:v0.0.17+origin
4cf8f8e04e Rename pkg/deploy -> pkg/apps
2617823364 Merge pull request #15202 from adelton/drop-oadm

# move generated patches to directory of service-catalog repo checkout
$ mv *.patch to /path/to/catalog/repo
# in origin-patches branch of service catalog
$ git checkout origin-patches
$ git am *.patch
$ git push openshift origin-patches

[At this point, the origin-patches branch requires no further changes.]


# Create a new branch based off the rebased origin-patches branch
$ git checkout -b $TAG+origin
# Squash all origin patches into one commit (search for earliest origin tooling
# commit, "origin build: add origin tooling").
$ git rebase -i <SHA before origin tooling>
# toggle all the origin commits to be squashed
$ git push openshift

[Now the openshift/service-catalog repo is fully up to date.]


# Pulls in code from openshift/service-catalog repo into OpenShift.
# Do not change the wording of the merge commit! The wording is used to set
# version information during the build.
$ git pull
$ git subtree pull --prefix cmd/service-catalog/go/src/github.com/kubernetes-incubator/service-catalog https://github.com/openshift/service-catalog $TAG+origin --squash -m "Merge version $TAG of Service Catalog from https://github.com/openshift/service-catalog:$TAG+origin"


===
The following shows the general flow for rebasing, along with how the
image is set within "oc cluster up". The info may be useful for testing
a rebase.
===

 Start here when there's a new upstream release.
                     +
                     |
+--------------------v--------------------+
| GIT:kubernetes-incubator/service-catalog|
| 1                                       |
+--------------------+--------------------+
                     |
                     |  rebase and push new tag
                     |
                     |
     +---------------v---------------+         +--------------------+
     |GIT:openshift/service-catalog  <---------+GIT:openshift/origin|  commits made in cmd/service-catalog
     |2                              |         |3                   |  are put in origin-patches branch in (2)
     +---------------+---------------+         +--------------------+
                     |
                     |  vendor into origin via subtree merge
                     |
                     |
         +-----------v---------+
         | GIT:openshift/origin|
         | 4                   |
         +---------------------+


Upon code landing in the openshift/origin repo, hack/build-release.sh is
called. This script calls hack/build-images.sh, which contains a list of
images and the directories containing the required Dockerfiles. Eventually,
hack/push-release.sh is called which pushes the origin-service-catalog image
to dockerhub.

When "oc cluster up --service-catalog" is executed, the template in
examples/service-catalog/service-catalog.yaml is used. (Technically it's not
used directly, and is updated via hack/update-generated-bindata.sh) The
template has a variable for the service catalog image to use named
SERVICE_CATALOG_IMAGE. The variable is currently set to
openshift/origin-service-catalog:latest.

In order to best test the rebase, rebuilding everything is best to ensure that
openshift is in sync with service catalog.
# builds origin
$ make

# builds new origin images with latest code
$ hack/build-local-images.py

$ builds new service catalog image with latest code
$ hack/build-local-images.py service-catalog

# run oc cluster up with latest images
$ oc cluster up --version=latest --service-catalog
