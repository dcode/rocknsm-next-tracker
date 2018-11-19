# Design Document

This document follows the style used in the [fedora-coreos-tracker](https://github.com/coreos/fedora-coreos-tracker/blob/master/Design.md)

- [OSTree Delivery Format](#ostree-delivery-format)
- [Release Streams](#release-streams)
- [Disk Layout](#disk-layout)
- [Approach towards shipping Python](#approach-towards-shipping-Python)
- [Identification in `/etc/os-release`](#identification-in-etcos-release)
- [Cloud Agents](#cloud-agents)
- [Supported Ignition Versions](#supported-ignition-versions)

### Summary:

RockNSM Next will be a container-based platform, allowing lightweight deployment on a single host using local container runtime (docker or podman) or scaled up deployment across multiple hosts using Kubernetes orchestration.

Eventually, RockNSM will be a customized distribution based upon Fedora/Redhat CoreOS. This means the operating system will be read-only, have SELinux enabled,
and will perform atomic updates with a recovery partition.

Fedora CoreOS isn't ready yet, however. In the meantime, we will implement the features on CentOS 7, as we do today, but with a very stripped down installation. Any additional tools will be deployed via the container toolbox approach.

This middle stage will have the core operating system delivered via RPM yum repositories. All RockNSM services and additional tooling will be delivered via OCI container repositories.

> ## Discussion
> We can possibly "defend" using yum through aliases as a simple reminder that
> users shouldn't use it.
> ~~~
> alias yum='echo "Instead of adding software via yum, please stick to the toolbox container pattern. If you really need yum, try \yum"'
> ~~~
> {: .source}
{: .callout}


## Release Streams

TODO

### Production Refs

TODO

## Disk Layout

RockNSM will migrate towards [FHS](https://en.wikipedia.org/wiki/Filesystem_Hierarchy_Standard) compliant filesystem layout. This will make it easier to migrate towards a read-only operating system. As a convenience, we can possibly bind mount new directory paths to their existing paths under `/data`.

The original rational of placing everything under `/data` was to make it easy to purge the sensor. However, currently in CoreOS, it is possible to purge the system to a brand new clean state with the following:

```
# Remove all user data in the root filesystem
rm -rf --one-file-system --no-preserve-root /
# Trigger firstboot
touch /boot/coreos/first_boot
```

Now, for sensor data itself (e.g. PCAP, kafka logs, etc), it is still best practice to place those on either another filesystem or some quota limited container volume. This approach will easily facilitate two scripts:

- Purge Data
- Factory Reset Sensor (as if it's never booted)

Proposed partition layout on GPT formatted disk:

|Part Num | Type | Mount Path | Purpose
| ---- | ---- | ---- | ---- |
| 1 | FAT32 | - | Boot Partition/EFI |
| 2 | XFS | /boot | BIOS boot partition |
| 3 | XFS | / | root OS partition
| 4 | XFS | /var | Variable OS data |
| 5 | XFS | /var/lib/stenographer | _(optional)_ Stenographer data files |
| 6 | XFS | /var/log/fsf | _(optional)_ FSF log data |
| 7 | XFS | /var/log/suricata | _(optional)_ Suricata log data |
| 8 | XFS | /var/log/bro | _(optional)_ Rotated Bro logs |
| 9 | XFS | /var/spool/bro | _(optional)_ Current Bro logs before rotation |
| 10 | XFS | /var/lib/elasticsearch | _(optional)_ Elasticsearch state directory |
| 11 | XFS | /var/lib/kafka | _(optional)_ Kafka state directory |
| 12 | XFS | /var/spool/docket | _(optional)_ Docket spool directory for temporary PCAP storage |

The optional partitions will be labelled application volumes inside the containers and either bind mounted into the container or attached as external volumes. Other state is relatively minimal and will be retained within the application container as optional container volumes.

**NOTE**: Open to changing this partition layout, but want to align with upstream. Another approach would be some sort of quota-oriented container volumes.

### Summary:

 - The bare metal image and cloud images have the same layout.
 - The `/var` and root (`/`) filesystems will be XFS by default.
 - Ignition will be used to customize disk layouts.

FCOS should have a fixed partition layout that Ignition can modify on first boot. The installer will be similar to the
Container Linux installer; the core of it will be dd'ing an image to the disk.

The partition layout is still undecided, but initial proposals look something like:

    Number	Type	Purpose
    -----------------------------------
    1	fat32	Boot partition/ESP
    2	N/A	Bios Boot partition
    3	XFS	Root
    4	XFS	/var
    ... potentially others, see **NOTE** above.

### Open Questions:

 - Do we use LVM for now (as is the CentOS 7 default) or switch to GPT partitions (as is the FCOS default)? I think switching from an RPM system to OSTree system is going to be destructive anyway, so the point may be moot.

## Identification in `/etc/os-release`

### Summary:

We will identify a RockNSM platform using the `VARIANT_ID=rocknsm`
field in the `/etc/os-release` file.

## Cloud Agents


### Summary:

 - RockNSM needs to support cloud agents for better deployment in offline
 environments. This is contrary to the FCOS approach, however FCOS does intend to support certain aspects of agents.
 - We could feasibly extend the installer to add-on cloud agents at install time via container. (vmtoolsd runs fine in a system container, for example).
 - For the short term, if we need to include an agent we will install it via the RPM in the kickstart %post (as we do now).

## Supported Ignition Versions

### Summary:

 - RockNSM will only support Ignition spec 3.0.0 and up.

Ignition will provide several desirable features:

- the ability to auto-grow storage on boot, if added to the sensor or VM
- ability to generate systemd units to start RockNSM services upon reconfiguration

## Services

The primary point of configuration will still be the RockNSM YAML file. Ansible will be responsible for:

- staging container images into a local registry
- populating service configuration values into `etcd`
- (single node) creating systemd service files to instantiate services
- (single node) populating service configuration templates (preferably using `confd`, allowing dynamic reconfig via `etcd`)
- (single node) allocating service volumes
- (multi node) create kubernetes pod configs for each service
- (multi node) generate service configuration templates
- (multi node) allocate volumes
