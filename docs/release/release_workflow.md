# How to create a DSP release

This doc outlines the steps required for manually preparing and performing a DSP release.

Versioning for DSP follows [semver]:

```txt
Given a version number MAJOR.MINOR.PATCH, increment the:

    MAJOR version when you make incompatible API changes
    MINOR version when you add functionality in a backward compatible manner
    PATCH version when you make backward compatible bug fixes
```

DSPO and DSP versioning is tied together, and DSP `MAJOR` versions are tied to [kfp-tekton] upstream.

> Note: In main branch all images should point to `latest` and not any specific versions, as `main` is rapidly moving,
> it is likely to quickly become incompatible with any specific tags/shas that are hardcoded.

## Pre-requisites
Need GitHub repo admin permissions for DSPO and DSP repos.

## Release workflow
Steps required for performing releases for `MAJOR`, `MINOR`, or `PATCH` vary depending on type.

### MAJOR Releases
Given that `MAJOR` releases often contain large scale, api breaking, changes. It is likely the release process will vary
between each `MAJOR` release. As such, each `MAJOR` release should have a specifically catered strategy.

### MINOR Releases
Let `x.y.z` be the `latest` release that is highest DSPO/DSP version.

Steps on how to release `x.y+1.z`

1. Ensure `compatibility.yaml` is upto date, and generate a new `compatibility.md`
   * Use [release-tools] to accomplish this
2. Cut branch `vx.y+1.x` from `main/master`, the trailing `.x` remains unchanged (e.g. `v1.2.x`, `v1.1.x`, etc.)
   * Do this for DSPO and DSP repos
3. Build images. Use the [build-tags] workflow
4. Retrieve the sha images from the resulting workflow (check quay.io for the digests)
   * Using [release-tools] generate a `params.env` and submit a new pr to vx.y+1.**x** branch
   * For images pulled from registry, ensure latest images are upto date
5. Perform any tests on the branch, confirm stability
   * If issues are found, they should be corrected in `main/master` and be cherry-picked into this branch.
6. Create a tag release for `x.y+1.z` in DSPO and DSP (e.g. `v1.3.0`)
7. Add any manifest changes to ODH manifests repo using the [ODH sync workflow]

**Downstream Specifics**

Downstream maintainers of DSP should forward any manifest changes to their odh-manifests downstream

### PATCH Releases
DSP supports bug/security fixes for versions that are at most 1 `MINOR` versions behind the latest `MINOR` release.
For example, if `v1.2` is the `latest` DSP release, DSP will backport bugs/security fixes to `v1.1` as `PATCH` (z) releases.

Let `x.y.z` be the `latest` release that is the highest version.\
Let `x.y-1.a` be the highest version release that is one `MINOR` version behind `x.y.z`

**Example**:
If the latest release that is the highest version is `v1.2.0`\
Then:
```txt
x.y.z = v1.2.0
x.y-1.a = v1.1.0
vx.y.z+1 = v1.2.1
vx.y-1.a+1 = v1.1.1
```

> Note `a` value in `x.y-1.a` is arbitrarily picked here. It is not always the case `z == a`, though it will likely
> be the case most of the time.

Following along our example, suppose a security bug was found in `main`, `x.y.z`, and `x.y-1.a`.
And suppose that commit `08eb98d` in `main` has resolved this issue.

Then the commit `08eb98d` needs to trickle to `vx.y.z` and `vx.y-1.a` as `PATCH` (z) releases: `vx.y.z+1` and `vx.y-1.a+1`

1. Cherry-pick commit `08eb98d` onto relevant minor branches `vx.y.x` and `vx.y-1.x`
   * The trailing `.x` in branch names remains unchanged (e.g. `v1.2.x`, `v1.1.x`, etc.)
2. Build images for `vx.y.z+1` and `vx.y-1.a+1` (e.g. `v1.2.1` and `v1.1.1`) DSPO and DSP
   * Images should be built off the `vx.y.x` and `vx.y-1.x` branches respectively
   * Use the [build-tags] workflow
3. Retrieve the sha image digests from the resulting workflow
   * Using [release-tools] generate a params.env and submit a new pr to `vx.y.x` and `vx.y-1.x` branches
4. Cut `vx.y.z+1` and `vx.y-1.a+1` in DSP and DSPO

**Downstream Specifics**

Downstream maintainers of DSP should:
* forward any manifest changes to their odh-manifests downstream
* ensure `odh-stable` branches in DSP/DSPO are upto date with bug/security fixes for the appropriate DSPO/DSP versions,
  and forward any changes from `odh-stable` to their downstream DSPO/DSP repos


## Release Automation Repo Notes

In addition to manual steps outlined above, the process is also automated via gh actions and can be triggered via the
`Release Prep` GH workflow found within this repository.

Some notes to consider before using this automation:
* ensure that a label named `release-automation` exists in the workflow directory (DSPO)
* tests will execute on any prs made to branches starting with `v`, e.g. `v1.3.x`
* once a pr with the label `release-automation` is merged, a release is created for the target release described in the PR body.
  * Additional checks are made to ensure a release:
    * if params.env was updated
    * if pr was merged instead of closed
    * if pr haas `release-automation` label
    * if the pr body has appropriate info needed for creating release (e.g. tag, previous tag, etc.)
* PRs need to be manually merged before continuing the automation (automation will not auto merge).

[semver]: https://semver.org/
[build-tags]: https://github.com/opendatahub-io/data-science-pipelines-operator/actions/workflows/build-tags.yml
[kfp-tekton]: https://github.com/kubeflow/kfp-tekton
[ODH sync workflow]: https://github.com/opendatahub-io/data-science-pipelines-operator/actions/workflows/odh-manifests-PR-sync.yml
[release-tools]: ../../scripts/release/README.md
