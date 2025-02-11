title: Install Singularity

Install Singularity
===================

[Singularity](http://singularity.hpcng.org/) is a tool that creates
docker-like process containers but without giving extra privileges to
unprivileged users.  It is used by grid pilot jobs (which are
submitted by per-VO grid workload management systems) to isolate user
jobs from the pilot's files and processes and from other users' files
and processes.  It also supplies a chroot environment in order to run
user jobs in different operating system images under one Linux kernel.

Kernels with a version 3.10.0-957 or newer include a feature called
unprivileged user namespaces that allows singularity to run completely
unprivileged.  This kernel version is the default for
RHEL/CentOS/Scientific Linux 7.6 and is available for previous 7.x
releases.  Although the feature is available, it needs to be enabled to
be usable (instructions below) on RHEL 7.  The feature is enabled by
default on RHEL 8.

Without unprivileged user namespaces, singularity must be installed and run
with setuid-root executables. Singularity keeps the privileged code to a
[minimum](https://sylabs.io/guides/3.6/user-guide/security.html#singularity-runtime-user-privilege)
in order to reduce the potential for vulnerabilities.

The OSG has installed singularity in [OASIS](install-cvmfs.md),
so most sites will not need to install singularity locally if they
enable it to run unprivileged.  An RPM installation can be configured to
be unprivileged or privileged.

!!! danger "Kernel vs. Userspace Security"
    Enabling unprivileged user namespaces increases the risk to the
    kernel. However, the kernel is much more widely reviewed than singularity
    and the additional capability given to users is more limited.
    OSG Security considers the non-setuid, kernel-based method to have a
    lower security risk.

This document is intended for system administrators that wish to enable,
install, and/or configure singularity.

Before Starting
---------------

As with all OSG software installations, there are some one-time (per host)
steps to prepare in advance:

- Ensure the host has [a supported operating system](../release/supported_platforms.md)
- Obtain root access to the host
- Prepare the [required Yum repositories](../common/yum.md). 
  Note that with OSG 3.5 and later, the singularity RPM comes from the
  EPEL yum repository.  OSG validates that distribution, and detailed
  instructions are still here.  With unprivileged singularity, no yum
  repository is needed.

In addition, this is highly recommended for image distribution:

- Install [CVMFS](install-cvmfs.md)

Choosing Unprivileged vs Privileged Singularity
-----------------------------------------------

There are two sets of instructions on this page:

- [Enabling Unprivileged Singularity](#enabling-unprivileged-singularity)
- [Singularity via RPM](#singularity-via-rpm)

OSG VOs all support running singularity directly from CVMFS when unprivileged
singularity is enabled.  Unprivileged singularity is enabled by default
on RHEL 8, and OSG recommends that system administrators enable it on
RHEL 7 worker nodes.  When unprivileged singularity is enabled, OSG
recommends that sites not install the singularity RPM unless they have
non-OSG users that require it.  Sites that do install the RPM may choose
to configure their RHEL 7 or later RPM installations to run unprivileged.

In addition to improved security, unprivileged singularity enables
`condor_ssh_to_job` to enter a container namespace without itself
needing privileges. 

On the other hand, there are some rare use cases that require
singularity to run privileged:

1. **Using single-file container images.**  Some systems, especially
    High Performance Computing (HPC) systems, deal poorly with
    collections of small files.  In this case, container images stored
    in a single file (as opposed to an unpacked directory) may be
    needed.

    However, OSG VO container images are usually directory-based in
    CVMFS, and when possible we [recommend disabling this
    feature](#limiting-image-types) on privileged installations in order
    to avoid potential kernel exploits.

1. **The overlay feature.**  The "overlay" feature of singularity uses
    the kernel overlayfs module to add bind mounts where mount points
    don't exist in the underlying image.

    However, singularity has an "underlay" feature that is
    equivalent which does not require
    privileges, and the overlay feature has been a source of security
    vulnerabilities in the past.  For these reasons,
    [we recommend replacing overlay with underlay](#configuring-singularity-rpm)
    even on privileged installations.

    overlayfs is also used to make the appearance of writable images
    when building containers, so it may be needed on some systems for
    that purpose.


Enabling Unprivileged Singularity
---------------------------------

The instructions in this section are for enabling singularity to run
unprivileged.

1. Enable user namespaces via `sysctl` on EL 7:

    If the operating system is an EL 7 variant and has been updated to
    the EL 7.6 kernel or later, enable unprivileged singularity with the
    following steps.  This step is not needed on EL 8 because it is
    enabled by default.

        :::console
        root@host # echo "user.max_user_namespaces = 15000" \
            > /etc/sysctl.d/90-max_user_namespaces.conf
        root@host # sysctl -p /etc/sysctl.d/90-max_user_namespaces.conf

1. (Optional) Disable network namespaces:

        :::console
        root@host # echo "user.max_net_namespaces = 0" \
            > /etc/sysctl.d/90-max_net_namespaces.conf
        root@host # sysctl -p /etc/sysctl.d/90-max_net_namespaces.conf

    OSG VOs do not need network namespaces with singularity, and
    disabling them reduces the risk profile of enabling user
    namespaces.

    Network namespaces are, however, utilized by other software,
    such as Docker.  Disabling network namespaces may break other
    software, or limit its capabilities (such as requiring the
    `--net=host` option in Docker).
    
    Disabling network namespaces blocks the systemd PrivateNetwork
    feature, which is a feature that is used by some EL 8 services.
    It is also configured for some EL 7 services but they are all
    disabled by default.  To check them all, look for PrivateNetwork in
    `/lib/systemd/system/*.service` and see which of those services are
    enabled but failed to start.  The only default such service on EL 8
    is systemd-hostnamed, and a popular non-default such service is
    mlocate-updatedb.  The PrivateNetwork feature can be turned off for
    a service without modifying an RPM-installed file through a
    `<service>.d/*.conf` file, for example for systemd-hostnamed:

        :::console
        root@host # cd /etc/systemd/system
        root@host # mkdir -p systemd-hostnamed.service.d
        root@host # (echo "[Service]"; echo "PrivateNetwork=no") \
                >systemd-hostnamed.service.d/no-private-network.conf
        root@host # systemctl daemon-reload
        root@host # systemctl start systemd-hostnamed
        root@host # systemctl status systemd-hostnamed

1. If docker is being used to run jobs, the following options are 
    recommended to allow unprivileged singularity to run (it does not
    need `--privileged` or any added capabilities):

        ::console
        --security-opt seccomp=unconfined --security-opt systempaths=unconfined

    `--security-opt seccomp=unconfined` enables unshare to be called (which is needed to
    create namespaces), and `--security-opt systempaths=unconfined` allows /proc to be mounted
    in an unprivileged process namespace (as done by singularity exec -p).
    `--security-opt systempaths=unconfined` requires Docker 19.03 or later.
    The options are secure as long as the system administrator controls
    the images and does not allow user code to run as root, and are
    generally more secure than adding capabilities.  If at this point no
    setuid programs needs to be run within the container, adding the
    following option will add security by preventing any privilege
    escalation (singularity uses the same feature on its containers):

        ::console
        --security-opt no-new-privileges

    In addition, the following option is recommended for allowing
    unprivileged fuse mounts on kernels that support that (RHEL >= 7.8):

        ::console
        --device=/dev/fuse

### Configuring Unprivileged Singularity ###

When unprivileged singularity is enabled and VOs run singularity from
CVMFS, the singularity configuration file also comes from CVMFS so local
sites have no control over changing the configuration.  However, the
most common local configuration change to the singularity RPM is to add
additional local "bind path" options to map extra local file paths into
containers.  This can instead be accomplished by setting the
`SINGULARITY_BINDPATH` variable in the environment of jobs, for
example through
[configuration](../other/configuration-with-osg-configure.md#local-settings)
on your compute entrypoint.
This is a comma-separated list of paths to bind, following the syntax of the
`singularity exec --bind` option.

There are also other environment variables that can affect singularity
operation; see the
[singularity documentation](https://sylabs.io/guides/3.6/user-guide/appendix.html)
for details.

### Validating Unprivileged Singularity ###

If you haven't yet installed [CVMFS](install-cvmfs.md), please do so.
Alternatively, use the
[cvmfsexec package](https://github.com/cvmfs-contrib/cvmfsexec)
configured for osg as an unprivileged user and mount the
oasis.opensciencegrid.org and singularity.opensciencegrid.org
repositories.

As an unprivileged user verify that singularity in OASIS works with this
command:

```console
user@host $ /cvmfs/oasis.opensciencegrid.org/mis/singularity/bin/singularity \
                exec --contain --ipc --pid --bind /cvmfs \
                /cvmfs/singularity.opensciencegrid.org/opensciencegrid/osgvo-el7:latest \
                ps -ef
UID        PID  PPID  C STIME TTY          TIME CMD
user         1     0  2 21:27 ?        00:00:00 shim-init
user         2     1  0 21:27 ?        00:00:00 ps -ef
```


Singularity via RPM
-------------------

The instructions in this section are for the singularity RPM, which 
includes setuid-root executables.  The setuid-root executables can
however be disabled by configuration, details below.

### Installing Singularity via RPM ###

To install the singularity RPM, make sure that your host is up to date
before installing the required packages:

1. Clean yum cache:

        ::console
        root@host # yum clean all --enablerepo=*

1. Update software:

        :::console
        root@host # yum update

    This command will update **all** packages

1. Install Singularity

        :::console
        root@host # yum install singularity

### Configuring Singularity RPM ###

singularity includes an option called `underlay` that enables using bind
mount points that do not exist in the container image.
By default it is enabled, but only if the similar `overlay` option cannot
be used, such as when running in unprivileged mode. 
It is recommended to completely disable `overlay`,
because it is more vulnerable to security problems than `underlay`.

Set this option in `/etc/singularity/singularity.conf`:

        enable overlay = no

!!! warning
    If you modify `/etc/singularity/singularity.conf`, be careful with
    your upgrade procedures.
    RPM will not automatically merge your changes with new upstream
    configuration keys, which may cause a broken install or inadvertently
    change the site configuration.  Singularity changes its default
    configuration file more frequently than typical OSG software.

    Look for `singularity.conf.rpmnew` after upgrades and merge in any
    changes to the defaults.

#### Configuring the RPM to be Unprivileged ####

If you choose to run the RPM unprivileged, after
[enabling unprivileged singularity](#enabling-unprivileged-singularity),
change the line in `/etc/singularity/singularity.conf` that says
`allow setuid = yes` to

        allow setuid = no

Note that the setuid-root executables stay installed, but they will exit
very early if invoked when the configuration file disallows setuid, so
the risk is very low.  There are non-setuid equivalent executables that
are used instead when setuid is disallowed.

#### Limiting Image Types ####

A side effect of disabling privileged singularity is that loopback
mounts are disallowed.  If the installation is privileged, also consider
the following.

Images based on loopback devices carry an inherently higher exposure to
unknown kernel exploits compared to directory-based images distributed via
CVMFS.  See [this article](https://lwn.net/Articles/652468/) for further
discussion.

The loopback-based images are the default image type produced by singularity
users and are common at sites with direct user logins.  However (as of April
2019) we are only aware of directory-based images being used by OSG VOs.
Hence, it is reasonable to disable the loopback-based images by setting
the following option in `/etc/singularity/singularity.conf`:

        max loop devices = 0

While reasonable for some sites, this is not required as there are currently
no public kernel exploits for this issue; any exploits are patched by
Red Hat when they are discovered.

### Validating Singularity RPM ###

After singularity is installed, as an ordinary user run the following
command to verify it:

```console
user@host $ singularity exec --contain --ipc --pid --bind /cvmfs \
                /cvmfs/singularity.opensciencegrid.org/opensciencegrid/osgvo-el7:latest \
                ps -ef
WARNING: Container does not have an exec helper script, calling 'ps' directly
UID        PID  PPID  C STIME TTY          TIME CMD
user         1     0  1 21:41 ?        00:00:00 shim-init
user         2     1  0 21:41 ?        00:00:00 ps -ef
```

Starting and Stopping Services
------------------------------

singularity has no services to start or stop.

References
----------
- [Singularity Documentation](https://www.sylabs.io/docs/)
- [Singularity Support](https://sylabs.io/resources/support)
- [Additional guidance for CMS sites](https://twiki.cern.ch/twiki/bin/view/Main/CmsSingularity)
