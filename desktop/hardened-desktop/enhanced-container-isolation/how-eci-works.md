---
description: Instructions on how to set up enhanced container isolation
title: How does it work?
keywords: set up, enhanced container isolation, rootless, security
---

>**Note**
>
>Enhance Container Isolation is available to Docker Business customers only.

Enhanced Container Isolation hardens container isolation using the [Sysbox
container runtime](https://github.com/nestybox/sysbox). Sysbox is a fork of the
standard OCI runc runtime that was modified to enhance container isolation and
workloads. For more details see [Under the covers](under-the-covers).

Starting with version 4.13, Docker Desktop includes a customized version of
Sysbox. This is available to Docker Business customers only.

When [Enhanced Container Isolation is enabled](index.md#how-do-i-enable-enhanced-container-isolation), containers
created by users through `docker run` or `docker create` are automatically
launched using Sysbox instead of the standard OCI runc runtime. Users need not
do anything else and can continue to use containers as usual. For exceptions,
see [FAQs and known issues](faq.md).

Even containers that use the insecure `--privileged` flag can now be run
securely with Enhanced Container Isolation, such that they can no longer be used
to breach the Docker Desktop Virtual Machine (VM) or other containers.

>Note
>
> When Enhanced Container Isolation is enabled in Docker Desktop, the Docker CLI
> "--runtime" flag is ignored. Docker's default runtime continues to be "runc",
> but all user containers are implicitly launched with Sysbox.

Enhanced Container Isolation is not the same as Docker Engine's userns-remap
mode or Rootless Docker. See [here](#enhanced-container-isolation-vs-docker-userns--remap-mode) and
[here](#enhanced-container-isolation-vs-rootless-docker) for more.

### Under the Covers

Sysbox enhances container isolation by using techniques such as:

* Enabling the Linux user-namespace on all containers (root user in the container maps to an unprivileged user in the Linux VM).
* Restricting the container from mounting sensitive VM directories
* Vetting sensitive system-calls between the container and the Linux kernel
* Mapping filesystem user/group IDs between the container's user-namespace and the Linux VM
* Emulating portions of the procfs and sysfs filesystems inside the container

Some of these are made possible by recent advances in the Linux kernel which
Docker Desktop now incorporates. Sysbox applies these techniques with minimal
functional or performance impact to containers.

These techniques complement Docker's traditional container security mechanisms
such as using other Linux namespaces, cgroups, restricted Linux capabilities,
seccomp, and AppArmor. They add a strong layer of isolation between the
container and the Linux kernel inside the Docker Desktop VM.

The following sections describe these features in more detail and provide
examples.

### Features and Benefits of Enhanced Container Isolation

#### Linux User Namespace on All Containers

With Enhanced Container Isolation, all user containers leverage the [Linux user-namespace][linux-user-ns]
for extra isolation. This means that the root user in the container maps to an unprivileged
user in the Docker Desktop Linux VM.

For example:

```
$ docker run -it --rm --name=first alpine
/ # cat /proc/self/uid_map
         0     100000      65536
```

The output `0 100000 65536` is the signature of the Linux user-namespace. It
means that the root user (0) in the container is mapped to unprivileged user
100000 in the Docker Desktop Linux VM, and the mapping extends for a continuous
range of 64K user IDs. The same applies to group IDs.

Each container gets an exclusive range of mappings, managed by Sysbox. For
example, if a second container is launched the mapping range is different:

```
$ docker run -it --rm --name=second alpine
/ # cat /proc/self/uid_map
         0     165536      65536
```

In contrast, without Enhanced Container Isolation, the container's root user is
in fact root on the host (aka "true root") and this applies to all containers:

```
$ docker run -it --rm alpine
/ # cat /proc/self/uid_map
         0       0     4294967295
```

By virtue of using the Linux user-namespace, Enhanced Container Isolation
ensures the container processes never run as user ID 0 (true root) in the Linux
VM. In fact they never run with any valid user-ID in the Linux VM. Thus, their
Linux capabilities are constrained to resources within the container only,
increasing isolation significantly compared to regular containers, both
container-to-host and cross-container isolation.

#### Privileged Containers Are Also Secured

Privileged containers `docker run --privileged ...` are insecure because they
give the container full access to the Linux kernel. That is, the container runs
as true root with all capabilities enabled, seccomp and AppArmor restrictions
are disabled, all hardware devices are exposed, etc.

For organizations that wish to secure Docker Desktop on their developer's
machines, privileged containers are problematic as they allow container
workloads whether benign or malicious to gain control of the Linux kernel
inside the Docker Desktop VM and thus modify security related settings (registry
access management, network proxies, etc.)

With Enhanced Container Isolation, privileged containers can no longer do this:
the combination of the Linux user-namespace and other security techniques used
by Sysbox ensures that processes inside a privileged container can only access
resources assigned to the container.

> Note
>
> Enhanced Container Isolation does not prevent users from launching privileged
> containers, but rather runs them securely by ensuring that they can only
> modify resources associated with the container. Privileged workloads that
> modify global kernel settings (e.g., loading a kernel module, changing BPF
> settings, etc.) will not work properly as they will receive "permission
> denied" error when attempting such operations.

For example, Enhanced Container Isolation ensures privileged containers can't
access Docker Desktop network settings in the Linux VM configured via Berkeley
Packet Filters (BPF):

```
$ docker run --privileged djs55/bpftool map show
Error: can't get next map: Operation not permitted
```

In contrast, without Enhanced Container Isolation, privileged containers
can easily do this:

```
$ docker run --privileged djs55/bpftool map show
17: ringbuf  name blocked_packets  flags 0x0
        key 0B  value 0B  max_entries 16777216  memlock 0B
18: hash  name allowed_map  flags 0x0
        key 4B  value 4B  max_entries 10000  memlock 81920B
20: lpm_trie  name allowed_trie  flags 0x1
        key 8B  value 8B  max_entries 1024  memlock 16384B
```

Note that some advanced container workloads require privileged containers, for
example Docker-in-Docker, Kubernetes-in-Docker, etc. With Enhanced Container
Isolation you can still run such workloads but do so much more securely than
before.

#### Containers can't share namespaces with the Linux VM

When Enhanced Container Isolation is enabled, containers can't share Linux
namespaces with the host (e.g., pid, network, uts, etc.) as that essentially
breaks isolation.

For example, sharing the pid namespace will fail:

```
$ docker run -it --rm --pid=host alpine
docker: Error response from daemon: failed to create shim task: OCI runtime create failed: error in the container spec: invalid or unsupported container spec: sysbox containers can't share namespaces [pid] with the host (because they use the linux user-namespace for isolation): unknown.
```

Similarly sharing the network namespace will fail:

```
docker run -it --rm --network=host alpine
docker: Error response from daemon: failed to create shim task: OCI runtime create failed: error in the container spec: invalid or unsupported container spec: sysbox containers can't share a network namespace with the host (because they use the linux user-namespace for isolation): unknown.
```

In addition, the `--userns=host` flag (to disable the user-namespace on the
container) is ignored:

```
$ docker run -it --rm --userns=host alpine
/ # cat /proc/self/uid_map
         0     100000      65536
```

Finally, Docker build `--network=host` and Docker buildx entitlements
(`network.host`, `security.insecure`) are not allowed. Builds that require these
will not work properly.

#### Bind-Mount Restrictions

When Enhanced Container Isolation is enabled, Docker Desktop users can continue
to bind-mount host directories into containers as configured via **Settings ->
Resources -> File sharing**, but they are no longer allowed to bind-mount
arbitrary Linux VM directories into containers.

This prevents containers from modifying sensitive files inside the Docker
Desktop Linux VM, files that can hold configurations for registry access
management, proxies, docker engine configurations, and more.

For example, the following bind-mount of the Docker Engine's configuration file
(`/etc/docker/daemon.json` inside the Linux VM) into a container is restricted
and will therefore fail:

```
$ docker run -it --rm -v /etc/docker/daemon.json:/mnt/daemon.json alpine
docker: Error response from daemon: failed to create shim task: OCI runtime create failed: error in the container spec: can't mount /etc/docker/daemon.json because it's configured as a restricted host mount: unknown
```

In contrast, without Enhanced Container Isolation this mount works and gives the
container full read and write access to the Docker Engine's configuration.

Of course, bind-mounts of host files continue to work as usual. For example,
assuming a user configures Docker Desktop to file share her $HOME directory,
she can bind-mount it into the container:

```
$ docker run -it --rm -v $HOME:/mnt alpine
/ #
```

> Note
>
> Enhanced Container Isolation won't allow bind-mounting the Docker socket
> (/var/run/docker.sock) into a container, as doing so essentially grants the
> container control of Docker, thus breaking container isolation. Containers
> that rely on this will not work with Enhanced Container Isolation enabled.

#### Vetting Sensitive System Calls

Another feature of Enhanced Container Isolation is that it intercepts and vets a
few highly sensitive system calls inside containers, such as `mount` and
`umount`.  This ensures that processes that have capabilities to execute these
system calls can't use them to breach the container.

For example, a container that has `CAP_SYS_ADMIN` (required to execute the
`mount` system call) can't use that capability to change a read-only bind-mount
into a read-write mount:

```
$ docker run -it --rm --cap-add SYS_ADMIN -v $HOME:/mnt:ro alpine
/ # mount -o remount,rw /mnt /mnt
mount: permission denied (are you root?)
```

Since the `$HOME` directory was mounted into the container's `/mnt` directory as
read-only, it can't be changed from within the container to read-write. This
ensures container processes use `mount` (or `umount`) to breach the container's
root filesystem.

Note however that in the example above the container can still create mounts
within the container, and mount them read-only or read-write as needed. Those
mounts are allowed since they occur within the container, and therefore don't
breach it's root filesystem:

```
/ # mkdir /root/tmpfs
/ # mount -t tmpfs tmpfs /root/tmpfs
/ # mount -o remount,ro /root/tmpfs /root/tmpfs

/ # findmnt | grep tmpfs
├─/root/tmpfs    tmpfs      tmpfs    ro,relatime,uid=100000,gid=100000

/ # mount -o remount,rw /root/tmpfs /root/tmpfs
/ # findmnt | grep tmpfs
├─/root/tmpfs    tmpfs      tmpfs    rw,relatime,uid=100000,gid=100000
```

This feature, together with the user-namespace, ensures that even if a container
process has all Linux capabilities they can't be used to breach the container.

Finally, Enhanced Container Isolation does system call vetting in such a way
that it does not affect the performance of containers in the great majority of
cases. It intercepts control-path system calls that are rarely used in most
container workloads but data-path system calls are not intercepted.

#### Filesystem user-ID mappings

As mentioned above, Enhanced Container Isolation enables the Linux
user-namespace on all containers and this ensures that the container's user-ID
range (0->64K) maps to an unprivileged range of "real" user-IDs in the Docker
Desktop Linux VM (e.g., 100000->165535).

Moreover, each container gets an exclusive range of real user-IDs in the Linux
VM (e.g., container 0 could get mapped to 100000->165535, container 2 to
165536->231071, container 3 to 231072->296607, and so on). Same applies to
group-IDs. In addition, if a container is stopped and restarted, there is no
guarantee it will receive the same mapping as before. This by design and further
improves security.

However the above presents a problem when mounting Docker volumes into
containers, as the files written to such volumes will have the real
user/group-IDs and will therefore won't be accessible across a container's
start/stop/restart, or between containers due to the different real
user-ID/group-ID of each container.

To solve this problem, Sysbox uses "filesystem user-ID remapping" via the Linux
Kernel's ID-mapped mounts feature (added in 2021) or an alternative module
called shiftfs. These technologies map filesystem accesses from the container's
real user-ID (e.g., range 100000->165535) to the range (0->65535) inside Docker
Desktop's Linux VM. This way, volumes can now be mounted or shared across
containers, even if each container uses an exclusive range of user-IDs. Users
need not worry about the container's real user-IDs.

Note that although filesystem user-ID remapping may cause containers to access
Linux VM files mounted into the container with real user-ID 0 (i.e., root), the
[restricted mounts feature](#bind-mount-restrictions) described above ensures
that no Linux VM sensitive files can be mounted into the container.

#### Procfs & Sysfs Emulation

Another feature of Enhanced Container Isolation is that inside each container,
the procfs ("/proc") and sysfs ("/sys") filesystems are partially emulated. This
serves several purposes, such as hiding sensitive host information inside the
container and namespacing host kernel resources that are not yet namespaced by
the Linux kernel itself.

As a simple example, when Enhanced Container Isolation is enabled the
`/proc/uptime` file shows the uptime of the container itself, not that of the
Docker Desktop Linux VM:

```
$ docker run -it --rm alpine
/ # cat /proc/uptime
5.86 5.86
```

In contrast, without Enhanced Container Isolation you would see the uptime of
the Docker Desktop Linux VM. Though this is a trivial example, it shows how
Enhanced Container Isolation aims to prevent the Linux VM's configuration and
information from leaking into the container so as to make it more difficult to
breach the VM.

In addition several other resources under "/proc/sys" that are not namespaced by
the Linux Kernel are also emulated inside the container. That is, each container
sees a separate view of each such resource and Sysbox reconciles the values
across the containers when programming the corresponding Linux kernel setting.

This has the advantage of enabling container workloads that would otherwise
require truly privileged containers to access such non-namespaced kernel
resources to run with Enhanced Container Isolation enabled, thereby improving
security.

### Limitations of Enhanced Container Isolation

Please refer to the [Limitations and Known Issues](faq.md) section.

### Enhanced Container Isolation vs Docker Userns-Remap Mode

The Docker Engine includes a feature called [userns-remap mode][docker-userns-remap]
that enables the user-namespace in all containers. However it suffers from a few
[limitations](https://docs.docker.com/engine/security/userns-remap/) and it's
not supported within Docker Desktop.

Userns-remap mode is similar to Enhanced Container Isolation in that both improve
container isolation by leveraging the Linux user-namespace.

However, Enhanced Container Isolation is much more advanced since it assigns
exclusive user-namespace mappings per container automatically and add several
other [container isolation features](#under-the-covers) meant to secure Docker
Desktop in organizations with stringent security requirements.

### Enhanced Container Isolation vs Rootless Docker

[Rootless Docker][rootless-docker] allows running the Docker Engine (and by
extension the containers) without root privileges natively a Linux host. This
allows non-root users install and run Docker natively on Linux.

Rootless Docker is not supported within Docker Desktop. While it's a valuable
feature when running Docker natively on Linux, its value within Docker Desktop
is reduced since Docker Desktop runs the Docker Engine within a Linux VM. That
is, Docker Desktop already allows non-root host users to run Docker and
isolates the Docker Engine from the host using a virtual machine.

Unlike Rootless Docker, Enhanced Container Isolation does not run Docker Engine
within a Linux user-namespace. Rather it runs the containers generated by that
engine within a user-namespace. This has the advantage of bypassing [the
limitations](/engine/security/rootless/#known-limitations) of Rootless Docker
and creates a stronger boundary between the containers and the Docker Engine.

Enhanced Container Isolation is meant to ensure containers launched with Docker
Desktop can't easily breach the Docker Desktop Linux VM and therefore modify
security settings within it.

[linux-user-ns]: https://man7.org/linux/man-pages/man7/user_namespaces.7.html
[docker-userns-remap]: https://docs.docker.com/engine/security/userns-remap/
[rootless-docker]: https://docs.docker.com/engine/security/rootless/
