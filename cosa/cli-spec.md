# CoreOS Assemlber Config Specification
- Subject ID: COSA.20200804
- Superceeds: None
- Conflicts: None

## Summary

The RHCOS development pipelines use a "jobspec" to describe the pipeline runs.  This enhancement proposes a file-based interface based loosely on the RHCOS jobspec.

## Goals

1. Provide a file-based interface into COSA
1. Prep for breaking up the Pipeline monoliths
1. Provide a migration path for RHCOS pipelines
1. Prepare for pipeline Nirvana (where *COS pipelines can live in peace and harmony)

## Why?

The innovation of the RHCOS JobSpec was it provided a clear seperation of configuration from the code; the logic was conflated. Further it the jobspec allowed the same logic to be run with different inputs.

To support this, a "CliSpec" is being introduced.

## YAML to EnvVars

* For Bash Pythonic CLIs, each argument will have an envVar added.
* At entry into COSA, the optional CliSpec will be checked
* Values will be parsed and emit as EnvVars by a common entry point.

## Example:

For the `coreos-assembler build` command:
```
build:
  force: false
  force_nocache: false
  skip_prune: false
  parent: <string>
  tag: <string>
  ```

  Would render the envVars:
  ```
  COSA_BUILD_FORCE=false
  COSA_BUILD_FORCE_NOCAHCE=false
  COSA_BUILD_SKIP_PRUNE=false
  ...
  ```

Then in `cmd-build` the defaults would be:
```
FORCE="${COSA_BUILD_FORCE:-}"
FORCE_IMAGE="${COSA_BUILD_FORCE_IMAGE:-}"
SKIP_PRUNE="${COSA_BUILD_SKIP_PRUNE:-0}"
...
```

## PR Standard

A thorough review of the COSA code-base, a comparison between the FCOS and RHCOS pipelines shows that each command will need to be handled seperately.

Therefore, for each command updated to modify this interface:
- The prefix `COSA_<COSA COMMAND>` will be used for envVars (e.g. `COSA_BUILDEXTEND_AZURE`)
- No calculated values. Commands may not parse the spec file itself.
- One command per PR.