# CoreOS Layering

This enhancement proposes:

- A fundamental new addition to ostree/rpm-ostree, which is support for directly pulling and updating the OS from container images (while keeping all existing functionality, per-node package layering and our existing support for pulling via ostree on the wire).
- Documentation for generating derived (layered) images from the pristine CoreOS base image.
- Support for switching a FCOS system to use a custom image on firstboot via Ignition
- zincati will continue to perform upgrades by inspecting the upgrade graph from the base image.

This builds on [Fedora Changes/OstreeNativeContainer](https://fedoraproject.org/wiki/Changes/OstreeNativeContainer).

# Existing work

Originally, https://github.com/coreos/fedora-coreos-tracker/issues/812 tracked native support for "encapsulating" ostree commits in containers.

Then, it was realized that when shipping the OS as a container image, it feels natural to support users deriving from it.  The bulk of this is really "ostree native container" integration glue, and is happening in https://github.com/ostreedev/ostree-rs-ext

rpm-ostree vendors the ostree-rs-ext code and also be extended to support the same interfaces as implemented by the "base" ostree-rs-ext code.

Specifically as of today, this functionality is exposed in:

- `rpm-ostree ex-container`
- `rpm-ostree rebase --experimental $containerref`

# Rationale

Since the creation of Container Linux (CoreOS) as well as Atomic Host, and continuing into the Fedora/RHEL CoreOS days, we have faced a constant tension around what we ship in the host system.

[This issue](https://github.com/coreos/fedora-coreos-tracker/issues/401) encapsulates much prior discussion.

For users who are happy with Fedora CoreOS today, not much will change.

For those who e.g. want to install custom agents or nontrivial amounts of code (such as kubelet), this "CoreOS layering" will be a powerful new mechanism to ship the code they need.

# Example via Dockerfile

One goal the current CoreOS team has in this is to still preserve the distinction we have between "base image" and user content.
This argues for a declarative input that only allows controlled mutation. See the next section about this.

However, `Dockerfile` is the lowest common denominator in the container ecosystem.
To truly illustrate the goal of supporting arbitrary inputs, we must support it.

[fcos-derivation-example](https://github.com/cgwalters/fcos-derivation-example) contains an example `Dockerfile` that builds a Go binary and injects it along with a corresponding systemd unit as a layer, building on top of Fedora CoreOS.

For ease of reference, a copy of the above is inline here:

```dockerfile
# Build a small Go program using a builder image
FROM registry.access.redhat.com/ubi8/ubi:latest as builder
WORKDIR /build
COPY . .
RUN yum -y install go-toolset
RUN go build hello-world.go

# In the future, this would be e.g. quay.io/coreos/fedora:stable
FROM quay.io/cgwalters/fcos
# Inject it into Fedora CoreOS
COPY --from=builder /build/hello-world /usr/bin
# And add our unit file
ADD hello-world.service /etc/systemd/system/hello-world.service
# Also add strace; we don't yet support `yum install` but we can
# with some work in rpm-ostree!
RUN rpm -Uvh https://kojipkgs.fedoraproject.org//packages/strace/5.14/1.fc34/x86_64/strace-5.14-1.fc34.x86_64.rpm
```

# Controlled mutation

One of the primary advantages of the `Dockerfile` layering approach is that it allows direct filesystem modifications. However, we should distinguish between layering things (e.g. `/etc` files, or third-party daemon) and modifying base content (e.g. fast-track kernel hotfix), which has a higher likelihood of invalidating our CI process. In a cluster context for example, it's possible that a cluster admin may want to permit users only certain modifications.

It is expected that control mechanisms will be integrated, though it's still not clear how that will look. It may be inside rpm-ostree (e.g. requiring override switches when first rebasing to the pullspec, and/or requiring a specific label on the image), or as part of the image build process itself (e.g. as part of [finalization](https://github.com/ostreedev/ostree-rs-ext/issues/159)). Of course, higher-level interfaces may enforce even stricter guidelines or only accept easily verifiable configs such as Butane/Ignition (see Butane example below).

Ideally, it shouldn't be difficult for an FCOS/RHCOS user to query the kinds of mutations inside a container image, and this could then be displayed in a succinct way as part of `rpm-ostree status` when rebased onto it.
# Derivation versus Ignition/Butane

This proposal does not replace Ignition.  Ignition will still play at least two key roles:

- Setting up partitions and storage, e.g. LUKS is something configured via Ignition provided on boot.
- Machine/node specific configuration, in particular bootstrap configuration: e.g. static IP addresses that are necessary to fetch container images at all.

# Butane as a declarative input format for layering

We must support `Dockerfile`, because it's the lowest common denominator for the container ecosystem, and is accepted as input for many tools.

However, one does not have to use `Dockerfile` to make containers.  Specifically, what would make a lot of sense for Fedora CoreOS is to focus
on Butane as a standard declarative interface to this process.

This could run as a "builder" container, something like this:

```
FROM quay.io/coreos/butane:release
COPY . .
RUN butane -o /build/ignition.json

FROM quay.io/fedora/coreos:stable
COPY --from=builder /build/ignition.json /tmp/
RUN ignition --write-filesystem /tmp/ignition.json && rm -f /tmp/ignition.json
```

Another option is to support being run nested inside an existing container tool, similar to
[kaniko](https://github.com/GoogleContainerTools/kaniko).  Then no
`Dockerfile` would be needed.

[More information on nesting container builds](https://www.redhat.com/sysadmin/podman-inside-kubernetes).

# Use of CoreOS disk/boot images

And more explicitly, it's expected that many if not most users would continue to use the official Fedora CoreOS "boot images" (e.g. ISO, AMI, qcow2, etc.).  This proposal does *not* currently call for exposing a way for a user to create the boot image shell around their custom container, although that is an obvious potential next step.

Hence, a user wanting to use a custom base image would provide machines with an Ignition config that performs e.g. `rpm-ostree rebase ostree-remote-image:quay.io/examplecorp/baseos:latest` as a systemd unit.  It is likely that we would provide this via [Butane](github.com/coreos/butane) as well; for example:

```
variant: fcos
version: 1.5.0
ostree_container:
  image: quay.io/mycorp/myfcos:stable
  reboot: true
```
