# INC-006 — Simulated Data Loss and Recovery via EBS Snapshot

## Environment
- AWS EC2 instance (Ubuntu 22.04 LTS, t3.micro), custom VPC, `us-east-2c`
- EBS volume (1 GiB, gp3) attached as a data disk, separate from the root volume
- AWS CLI for all volume/snapshot operations, SSH for filesystem-level work

## Problem
Simulated scenario: a file on an attached data volume is deleted and needs to be recovered without any application-level backup — only an EBS snapshot taken earlier.

## Impact
In a real incident, this represents permanent loss of the file unless a point-in-time backup (snapshot) exists and can be restored to a new volume. This exercise validates that the backup/restore path actually works before it's needed for real.

## Volume / Snapshot / Restore Relationship

```mermaid
flowchart LR
    V1[Original Volume<br/>/dev/nvme1n1<br/>file deleted in step 5]
    SNAP[(Snapshot<br/>point-in-time copy)]
    V2[New Volume<br/>/dev/nvme2n1<br/>restored from snapshot]
    EC2[Same EC2 Instance]

    V1 -- create-snapshot --> SNAP
    SNAP -- create-volume --snapshot-id --> V2
    V1 -.attached.-> EC2
    V2 -.attached.-> EC2
```

Both volumes end up attached to the same instance simultaneously, under different device names — this is exactly what caused the step-7 mistake: mounting `nvme1n1` (the original, already missing the file) instead of `nvme2n1` (the actual restored volume).

## Investigation / Procedure
1. Created and attached a 1 GiB gp3 volume to the running instance:
   ```
   aws ec2 create-volume --availability-zone us-east-2c --size 1 --volume-type gp3 --query "VolumeId" --output text --profile personal
   aws ec2 attach-volume --volume-id <volume-id> --instance-id <instance-id> --device /dev/sdf --profile personal
   ```
2. Confirmed the OS saw the new disk with `lsblk`. Note: the requested device name (`/dev/sdf`) did not match what Linux exposed it as — on this instance type (Nitro-based, NVMe), AWS renames block devices to `/dev/nvme1n1` rather than honoring the requested `sdf` name. This is expected behavior, not a failure — `lsblk` is the reliable way to confirm the actual device name rather than assuming the name passed to `attach-volume`.
3. Formatted and mounted the volume, then wrote a test file:
   ```
   sudo mkfs -t ext4 /dev/nvme1n1
   sudo mkdir /data
   sudo mount /dev/nvme1n1 /data
   echo "Esto es una prueba de EBS" | sudo tee /data/prueba.txt
   ```
4. Created a snapshot of the volume while the file existed:
   ```
   aws ec2 create-snapshot --volume-id <volume-id> --description "Snapshot con archivo de prueba" --query "SnapshotId" --output text --profile personal
   ```
   Waited for `State: completed` via `describe-snapshots` before proceeding.
5. Simulated data loss by deleting the file from the original volume:
   ```
   sudo rm /data/prueba.txt
   ```
6. Restored by creating a **new** volume from the snapshot and attaching it to the same instance on a different device:
   ```
   aws ec2 create-volume --availability-zone us-east-2c --snapshot-id <snapshot-id> --volume-type gp3 --query "VolumeId" --output text --profile personal
   aws ec2 attach-volume --volume-id <new-volume-id> --instance-id <instance-id> --device /dev/sdg --profile personal
   ```
7. Attempted to mount and verify — first attempt targeted the wrong device (`nvme1n1`, the *original* volume, which no longer had the file since it was deleted there in step 5):
   ```
   sudo mount /dev/nvme1n1 /data-restored
   cat /data-restored/prueba.txt
   # cat: /data-restored/prueba.txt: No such file or directory
   ```
   Re-checked `lsblk` output, identified the actual new device (`nvme2n1`), unmounted the wrong one and mounted the correct one:
   ```
   sudo mount /dev/nvme2n1 /data-restored
   cat /data-restored/prueba.txt
   # Esto es una prueba de EBS
   ```

## Root Cause
N/A (planned exercise, not an organic failure) — the "recovery miss" in step 7 had its own mini root cause: device naming for NVMe-based instances doesn't match the name requested at `attach-volume` time, so a new volume can easily be mounted to the wrong mount point if device names are assumed rather than confirmed via `lsblk` after each attach.

## Resolution
File successfully recovered from the snapshot-restored volume, confirmed via `cat`.

## Prevention
- Always confirm actual device names with `lsblk` after attaching a volume — never assume the `--device` value passed to `attach-volume` is what the OS will expose, especially on Nitro/NVMe instance types.
- When multiple volumes are attached to the same instance (e.g., original + restored, as in this exercise), double-check which block device corresponds to which volume before mounting — a wrong mount doesn't error out, it just silently shows stale or missing data, which can be mistaken for a failed restore when the restore actually worked fine.
- Snapshots should be taken on a schedule (not just ad hoc) for anything that can't afford to be lost — AWS Data Lifecycle Manager can automate this.

## What I'd Do Differently in Production
Tag volumes and snapshots with clear identifiers (e.g., `Name=data-original`, `Name=data-restored-from-snap-xxxx`) at creation time — relying on device names like `nvme1n1` vs `nvme2n1` to distinguish "which volume is which" doesn't scale past a quick lab exercise and is exactly the kind of ambiguity that caused the wrong-mount mistake in step 7.
