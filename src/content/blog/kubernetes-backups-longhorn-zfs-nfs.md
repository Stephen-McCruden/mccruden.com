---
title: Backing Up My K8s Homelab with Longhorn, ZFS, and NFS
description: Setting up an external backup server for my Kubernetes homelab using ZFS, NFS, and Longhorn. I walk through the configuration, integration with Flux, and testing a backup and restore to verify that the data can actually be recovered.
pubDate: 2026-11-01
updatedDate: 2026-11-01
tags:
  - Kubernetes
  - "#Longhorn"
  - "#Homelab"
  - "#ZFS"
  - "#NFS"
  - "#DR"
draft: false
---
## Why I Wanted to Set This Up

One of the things I've been meaning to address in my homelab is backups.

I've been using Longhorn for persistent storage in my Kubernetes cluster, with two replicas configured for each volume. Although this provides redundancy if a node goes down, it doesn't protect me from accidentally deleting data, corrupting a volume, or losing the entire cluster.

This is particularly important because I want to be able to completely destroy and rebuild my Kubernetes infrastructure using Terraform, Ansible, and Flux without losing the data my applications depend on.

I had an HP Z440 running Ubuntu Server 24.04 with two spare 1 TB Western Digital hard drives, so I decided to use it as a dedicated backup server.

My plan was to configure the drives as a ZFS mirror, expose the storage over NFS, and configure Longhorn to back up its volumes to that server.

More importantly, I wanted to test whether I could actually restore the data.

---
## Setting Up ZFS

I decided to use ZFS with the two drives configured as a mirror. This gives me approximately 1 TB of usable storage while allowing one drive to fail without losing the pool.

Before setting everything up, I checked the SMART attributes of both drives and ran short self-tests. Both tests passed, with no obvious issues in the attributes I reviewed.

After installing ZFS on Ubuntu, I created the pool using the drives' persistent identifiers under `/dev/disk/by-id/`.

```bash
sudo zpool create \
  -o ashift=12 \
  -O compression=lz4 \
  -O atime=off \
  -O mountpoint=/srv/nas \
  tank mirror \
  /dev/disk/by-id/<disk-1> \
  /dev/disk/by-id/<disk-2>
```

I used `ashift=12` for 4 KiB sector alignment, enabled LZ4 compression, and disabled access-time updates to avoid unnecessary writes.

After creating the pool, I checked its status with `zpool status` and confirmed both drives were online with no reported errors.

I then created a separate dataset for the Longhorn backups:

```bash
sudo zfs create tank/longhorn-backups
```

This mounted the dataset at `/srv/nas/longhorn-backups`, keeping it separate from anything else I might decide to store on the server later.

---
## Configuring NFS

With the storage ready, I needed to make it accessible to my Kubernetes nodes.

I installed `nfs-kernel-server` and configured an NFS export for the backup dataset. Rather than allowing the entire network to access it, I restricted the export to the three Kubernetes node IP addresses.

I also created a dedicated `longhornbackup` user with UID/GID `2000` and configured the export to map client requests to that account.

The configuration uses the following options:

```text
rw,sync,no_subtree_check,root_squash,all_squash,anonuid=2000,anongid=2000
```

I added a systemd dependency to make sure the ZFS dataset is mounted before the NFS service starts.

Before configuring Longhorn, I SSH'd into one of my Kubernetes workers and mounted the export manually.

```bash
sudo mount -t nfs4 \
  192.168.0.115:/srv/nas/longhorn-backups \
  /mnt/longhorn-nfs-test
```

The mount succeeded using NFSv4.2. I created a test file, checked that it was owned by UID/GID `2000`, and then removed it.

With that working, I knew the Kubernetes nodes could reach and write to the NAS.

---
## Configuring Longhorn Through Flux

Since I manage my Kubernetes infrastructure through Flux, I wanted the backup target configuration in Git rather than configured manually.

I updated my existing Longhorn HelmRelease with:

```yaml
defaultBackupStore:
  backupTarget: "nfs://192.168.0.115:/srv/nas/longhorn-backups"
  pollInterval: 300
```

After committing the change and reconciling Flux, Longhorn successfully saw the path.

You can view the full configuration in my [Longhorn HelmRelease on GitHub.](https://github.com/Stephen-McCruden/homelab/blob/main/kubernetes/infrastructure/controllers/longhorn/helmrelease.yaml)

---
## Testing the Backup and Restore

Getting Longhorn connected to the NAS was only half of what I wanted to accomplish. I also wanted to know whether I could restore data from it.

Rather than risk any of my existing applications, I created a temporary namespace and a 1 GiB Longhorn PVC.

I deployed a BusyBox Pod that mounted the volume and wrote a text file along with its SHA-256 checksum.

```bash
echo "homelab-disaster-recovery-test-2026" > /data/proof.txt
sha256sum /data/proof.txt > /data/proof.sha256
```

I verified that the checksum matched, confirmed the Longhorn volume was healthy, and then deleted the writer Pod to prevent any further writes during the backup.

Next, I created a manual Longhorn backup to the NFS target.

```bash
kubectl -n longhorn-system get backups.longhorn.io
```

The backup reported a `Completed` state, and I could see data in the backup repository on the Z440.

### Restoring the Volume

I restored the backup into a separate Longhorn volume named `backup-test-restored`, leaving the original PVC untouched.

One thing I ran into was that restoring a Longhorn volume doesn't automatically mean Kubernetes has a PVC available for it.

I created a static PV and PVC, using the restored volume's name as its CSI volume handle:

```yaml
csi:
  driver: driver.longhorn.io
  volumeHandle: backup-test-restored
  fsType: ext4
```

After the PV and PVC were bound, I deployed another BusyBox Pod that mounted the restored volume and checked the original SHA-256 checksum.

The result was:

```text
/data/proof.txt: OK
RESTORE VERIFIED
```

Exactly what I wanted to see.

I'd successfully backed up data from a Kubernetes persistent volume to a separate physical server, restored it into another volume, and verified that the data hadn't changed.

---
## What's Next

I'm happy with how this turned out, but there's still more I want to do.

For now, I've tested a simple volume restore within my existing Kubernetes cluster. I haven't yet tested recovering an entire application or restoring data after rebuilding the cluster itself.

My next steps are to configure automated backup schedules and retention policies through Flux, implement application-consistent backups for databases, and test restoring a real application.

Eventually, I want to be able to destroy my Kubernetes VMs, provision them again using Terraform, configure them with Ansible, bootstrap Kubernetes and Flux, and restore the application data from my NAS.

I also plan to add offsite backups, since having two mirrored drives doesn't protect against losing the entire server.

For now, I have a working backup destination outside my Kubernetes cluster, and I've verified that I can actually restore data from it. That's a good starting point for building out the rest of my disaster-recovery process.
