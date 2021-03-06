# Releases

Deis uses a [continuous delivery][] approach for creating releases. Every merged commit that passes
testing results in a deliverable that can be given a [semantic version][] tag and shipped.

The master `git` branch of a project should always work. Only changes considered ready to be
released publicly are merged.


## Components Release as Needed

Deis components release new versions as often as needed. Fixing a high priority bug requires the
project maintainer to create a new patch release. Merging a backward-compatible feature implies
a minor release.

By releasing often, each component release becomes a safe and routine event. This makes it faster
and easier for users to obtain specific fixes. Continuous delivery also reduces the work
necessary to release a product such as Deis Workflow, which integrates several components.

"Components" applies not just to Deis Workflow projects, but also to development and release
tools, to Docker base images, and to other Deis projects that do [semantic version][] releases.

See "[How to Release a Component](#how-to-release-a-component)" for more detail.


## Workflow Releases Every Two Weeks

Deis Workflow has a regular, public release cadence. From v2.0.0 onward, the Deis team releases
every 2 weeks, with patch versions as needed. GitHub milestones are used to communicate the
content and timing of major and minor releases.

Workflow release timing is not linked to specific features. If a feature is merged before the
release date, it is included in the next release.

See "[How to Release Workflow](#how-to-release-workflow)" for more detail.


## Semantic Versioning

Deis releases comply with [semantic versioning][semantic version], with the "public API" broadly
defined as:

- REST, gRPC, or other API that is network-accessible
- Library or framework API intended for public use
- "Pluggable" socket-level protocols users can redirect
- CLI commands and output formats

In general, changes to anything a user might reasonably link to, customize, or integrate with
should be backward-compatible, or else require a major release. Deis users can be confident that
upgrading to a patch or to a minor release will not break anything.


## How to Release a Component

Most Deis projects are "components" which produce a Docker image or binary executable as a
deliverable. This section leads a maintainer through creating a component release.

### Step 1: Update Code and Set Environment variables

In the component repository, update from the GitHub remote and ensure `HEAD` is the commit
intended for release. Major or minor releases should happen on the master branch. Patch releases
should check out the previous release tag and cherry-pick specific commits from master.

Double-check that `git log` looks correct, then set some environment variables:
```bash
export COMPONENT=${PWD##*/}
export OLD_RELEASE=$(git describe --abbrev=0 --tags)
export NEW_SHA=$(git rev-parse --short HEAD)
deisrel changelog individual $COMPONENT $OLD_RELEASE $NEW_SHA unknown
export NEW_RELEASE=v2.2.1  # changelog agrees it's a patch release
```

### Step 2: Push the Release Tag

Push the new release tag to the GitHub repository:

```bash
git tag $NEW_RELEASE && git push upstream $NEW_RELEASE
```

### Step 3: Put CHANGELOG in GitHub Release Notes

Generate the CHANGELOG with the [`deisrel`](https://github.com/deis/deisrel.git) tool and paste
it into the body of release notes for the component in GitHub. In the "Release Title" field, use
the project & component with its release, such as "Deis Controller v2.2.1":

```bash
deisrel changelog individual $COMPONENT $OLD_RELEASE $NEW_SHA $NEW_RELEASE | pbcopy
open https://github.com/deis/$COMPONENT/releases/new?tag=$NEW_RELEASE
```

### Step 4: Publish Release to Versions Server

Finally, use the Workflow Manager API publishing tool to add the new component release to the
versions server. This step is described in the workflow-manager-api-publish docs and must be done
by a Deis maintainer.


## How to Release Workflow

Deis Workflow integrates multiple component releases together with a [Helm Classic][] chart
deliverable. This section leads a maintainer through creating a Workflow release.

### Step 1: Update Code and Set Environment Variables

In the [deis/charts][] repository, update from the GitHub remote. Major or minor releases start
from the master branch. Patch releases should check out the previous release tag and cherry-pick
specific commits from master.

Export two environment variables that will be used in later steps:

```bash
export WORKFLOW_RELEASE=v2.3.0 WORKFLOW_PREV_RELEASE=v2.2.0  # for example
```

### Step 2: Update Jenkins Jobs

Update the `WORKFLOW_RELEASE` value in the
[common.groovy](https://github.com/deis/jenkins-jobs/blob/master/common.groovy) file so the [workflow-test-release](https://ci.deis.io/job/workflow-test-release/) job will kick off
automatically when the `release-${WORKFLOW_RELEASE}` branch is pushed:

```bash
git clone git@github.com:deis/jenkins-jobs.git
# update WORKFLOW_RELEASE value
git commit -a -m "chore(workflow-$WORKFLOW_RELEASE): update WORKFLOW_RELEASE"
git push upstream HEAD:master
```

### Step 3: Create Helm Charts

For a patch release, check out the previous tag and cherry-pick commits onto it:

```bash
git checkout -b release-$WORKFLOW_RELEASE $WORKFLOW_PREV_RELEASE
git cherry-pick 143ac41  # and so on...
```

For a major or minor release, copy and modify the current development charts:

```bash
git checkout -b release-$WORKFLOW_RELEASE master
cp -r workflow-dev workflow-$WORKFLOW_RELEASE
cp -r workflow-dev-e2e workflow-$WORKFLOW_RELEASE-e2e
cp -r router-dev router-$WORKFLOW_RELEASE
```

Change the `generate_params.toml` file in **each** new chart as follows:

  1. Set all `imagePullPolicy` values to `IfNotPresent`
  1. Set all `org` values to `"deis"`
  1. Set all `dockerTag` values to latest releases for each component
  1. If there's a `[workflowManager]` section, change `versionsApiURL` to
     `"https://versions.deis.com"` and `doctorApiURL` to `"https://doctor.deis.com"`

Change the `workflow-$WORKFLOW_RELEASE/tpl/deis-controller-rc.yaml` file:

  1. Remove the `KUBERNETES_POD_TERMINATION_GRACE_PERIOD_SECONDS` env var

Commit and push your changes:

```bash
git commit -a -m "chore(workflow-$WORKFLOW_RELEASE): releasing workflow-$WORKFLOW_RELEASE(-e2e)"
git push origin HEAD:release-$WORKFLOW_RELEASE
```

Open a pull request at [deis/charts][] to merge this branch into master.

### Step 4: Manual Testing

Now it's time to go above and beyond current CI tests. Create a testing matrix spreadsheet (copying
from the previous document is a good start) and sign up testers to cover all permutations.

Testers should pay special attention to the overall user experience, make sure upgrading from
earlier versions is smooth, and cover various storage configurations and Kubernetes versions and
infrastructure providers.

When showstopper-level bugs are found, the process is as follows:

1. Create a component PR that fixes the bug.
1. Once the PR passes and is reviewed, merge it and do a new
  [component release](#how-to-release-a-component)
1. Update that component's `dockerTag` value in the release chart(s) to the new semver tag
1. Commit and push the chart changes to the release branch and restart testing

### Step 5: Merge and Put CHANGELOG in GitHub Release Notes

When testing has completed without uncovering any new showstopper bugs and the charts PR has been
reviewed successfully, merge it to master. Then update your local master branch, tag it with the
release, and push the tag:

```bash
git checkout master && git fetch --tags upstream master && git merge upstream/master
git tag $WORKFLOW_RELEASE
git push upstream $WORKFLOW_RELEASE
```

Generate the CHANGELOG with the [`deisrel`](https://github.com/deis/deisrel.git) tool and paste
it into the body of release notes for [deis/charts][] in GitHub. In the "Release Title" field, use
the project & component with its release, such as "Deis Workflow v2.3.0":

```bash
deisrel changelog individual workflow $WORKFLOW_PREV_RELEASE HEAD $WORKFLOW_RELEASE | pbcopy
open https://github.com/deis/workflow/releases/new?tag=$WORKFLOW_RELEASE
```

### Step 6: Close GitHub Milestone

Close the GitHub milestone for [deis/charts][] and for [deis/workflow][] and create new ones for
the next planned release.

### Step 7: Update Documentation

Create a new pull request at [deis/workflow][] that updates version references to the new release.
Use `git grep $WORKFLOW_PREV_RELEASE` to find any references, but be careful not to change
`CHANGELOG.md`. This PR should also change `upgrading-workflow-md` by updating references to
older releases to `$WORKFLOW_PREV_RELEASE`, so the documentation always describes upgrading
between recent versions.

### Step 8: Assemble Master Changelog

Each component already updated its release notes on GitHub with CHANGELOG content. The
bodies of each component's release notes should be concatenated into a single gist. Note that there
may be more than one release per component--and more than one set of release notes--included in the
Workflow release.

### Step 9: Let Everyone Know

Let the rest of the team know they can start blogging and tweeting about the new Workflow release.
Post a message to the #company channel on Slack. Include a link to the released chart and to the
master CHANGELOG:

```
@here Deis Workflow v2.3.0 is now live!
Release notes: https://github.com/deis/charts/releases/tag/v2.3.0
Master CHANGELOG: https://gist.github.com/mboersma/dcb300c1530552dc612b19f418731234
```

You're done with the release. Nice job!

[continuous delivery]: https://en.wikipedia.org/wiki/Continuous_delivery
[deis/charts]: https://github.com/deis/charts
[deis/workflow]: https://github.com/deis/workflow
[Helm classic]: https://github.com/helm/helm-classic
[semantic version]: http://semver.org
