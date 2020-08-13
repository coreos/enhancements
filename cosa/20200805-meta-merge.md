# Merging of build meta-data
- Specification ID: cosa-20200805
- Supersedes: None
- Conflicts: None

# Background

One of the biggest blockers to performing parallel builds, distributed builds and/or re-starting failed builds is the ability to lock write access to `meta.json` in COSA.

Under the following assumptions, COSA needs to support merging of `meta.json`:
- Not all `cmd-buildextend-*` operations will happen on the same pod or Linux namespace
- S3 is the authoritative store for builds. S3 is eventually consistent meaning that the version read from S3 may be replaced after reading from S3. Implementing distributed locking on `meta.json` based on S3 is problematic.
- For RHCOS we need the ability to do partial builds and then back-fill artifacts (e.g. only build qemu, but not cloud) and later build cloud.
- Developers want to restart failed builds
- Consumers of the artifacts from COSA want to reduce the time to build and test

## State of a build is a point-of-view statement

To COSA a build is usually `ostree` + `qcow2`. The notion of a "failed" or "partial" build is a point of view. COSA does provision the build id, but COSA does not know what comprised the artifacts of the "build". Usually developers and consumers speak of a build as the base `ostree` and the _optional_ artifacts.

In COSA a "build" only starts to exist when the `meta.json` is incepted after the `ostree` has been written. The build directory may exist, but the `meta.json` that is produced immediately is valid for any consumer. In most cases a "failed" build means that an artifact in the `buildextend-*` commands has failed to publish (such as AWS) not that the artifact itself failed to build; the artifact was indeed built.

Further, a "failed build" is also used to mean that some part of a pipeline failed to complete. COSA itself does not have any concept of dependencies, much less the idea that a multiple COSA commands can be run. COSA only has implied dependencies that some artifacts requires others. In other words, COSA doesn't know when the "build" is complete until after something else considers it's complete.

## Multiple `meta.json`

Under the axiom that the state of a build is usually determined from the point-of-view of a pipeline and to support distributed and parallel builds, builds will be produced with multiple `meta.json` that will be merged together after the artifact is built.

## `cosalib.meta` library

`cmd-meta` currently uses the [`cosalib.meta`](https://github.com/coreos/coreos-assembler/blob/master/src/cosalib/meta.py) library for reading and writing `meta.json`. To support merging of `meta.json` files, the library will be updated to allow for merging in-memory updates with on-disk content.

Only `meta.json` that is validated will be merged under the following rules:
- source and destination meta must match the buildid, name and ostree commit
- on-disk `meta.json` takes precedence over the in-memory meta
- all versions must be validated before and after merging before commiting to disk.

Prior to merging `cosalib.meta` will lock the localfile.

## merge mode denoted in `meta.json`

To support production of meta meta data, `cmd-build` will have a key to support merging meta-data in a delayed fashion.

```
{
    	"coreos-assembler.delayed-meta-merge": true
}
```

`cmd-build` will have the flag of `--delay-meta-merge` added to set the merge mode. The default is `false`.

## Delayed Merging of meta-data

When `coreos-assembler.delayed-meta-merge` is set to `true`, `cmd-buildextend.<ARTIFACT>` commands will create suffixed `meta.<ARTIFACT>.json` instead of writing directly to `meta.json`. A new command called `cmd-meta-merge` will be introduced that will read `meta.<ARTIFACT>.json` and will ONLY combine the images and the cloud-build specific sections into `meta.json`.

Orchestration layers will be responsible for shuffling artifacts around, e.g. `cmd-upload-build`. `cmd-upload-build --meta <ARTIFACT>` will only upload the artifacts described in the meta.

All `meta.<ARTIFACT>.json` will be removed when their contents are merged into `meta.json`. If the artifacts are produced across compute domains, orchestration layers are responsible for calling `cmd-upload-build` to stage the artifacts.

`cmd-meta-merge` will be able to read `meta.<ARTIFACT>.json`. `cmd-meta-merge` will:
- read in `meta.json` and validate it.
- iterate over `meta.<ARTIFACT>.json` in the path or in S3.
- Validation against the schema for both source and destination meta-data before and after merging.
- the `meta.<ARTIFACT>.json` will be deleted

## Merging logic

By default, only keys not found in `meta.json` will be added from `meta.<ARTIFACT>.json`. To support rebuilds/restart, a force option will exist in `cmd-meta-merge` with a default of `false`.

Keys that will be merged:
- fields from the images section
- known cloud names such as AWS, Azure, GPC, etc.

## Default builds

When `coreos-assembler.delayed-meta-merge` is false or absent, `cmd-buildextend-*` commands will will produce a `meta.<ARTIFACT>.json` and immediately merge into `meta.json` using the `cmd-meta-merge`. To support bash commands `finalize-artifact` will use `cmd-meta-meta` for `meta.json`.

The immediate merging of metas requires:
- time sync clocks
- shared disk for `builds/`

## Validation Requirement

All merged `meta.json` must be validated against the schema.
