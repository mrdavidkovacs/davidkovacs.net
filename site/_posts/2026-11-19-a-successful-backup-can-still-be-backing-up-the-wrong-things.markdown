---
layout: post
title: "A Successful Backup Can Still Be Backing Up the Wrong Things"
date: 2026-11-19 07:00:00 +0100
categories: systems backups operations reliability
excerpt: "A backup can run successfully and still include an old copy of the data. I traced the active library, checked the backup selection, and restored two files to see what the result actually proved."
---

A self-hosted media application stores its library on a NAS. A backup task copies selected NAS folders to a remote repository and keeps multiple recovery points.

After moving the application data to a different location, the application continued to work and the backup task continued to run. During a later review of the backup selection, I found two library trees: the location the application now used and a separate tree left behind by the earlier setup. Both had been selected for backup.

The job was protecting the paths it had been given. I needed to check whether those paths still matched the system I intended to recover.

## A larger folder is not proof of a complete copy

The two paths were separate directories, not two names for the same mounted tree. The new library was larger than the old one, which made it tempting to conclude that the old selection was redundant.

That conclusion was too quick. A larger directory can still be missing files that exist in a smaller one. Size tells me how much data is there, not whether one tree contains everything in the other.

The first useful check was to trace the application's active storage mount back to the NAS. That identified the current library. I then checked the backup task's source selection: the share containing the current library was included, while the old library path was no longer selected.

Those observations establish which path the application uses and which source the task is configured to include. They do not establish that every file in the old tree exists in the new one. The old directory itself was a separate question from the backup selection. Removing a path from the task is not the same operation as deleting its files.

## Start at the restore point

A source selection describes what a task is configured to back up. To see what was actually available, I opened the latest recovery point marked successful and browsed the protected data.

The active media tree was visible, including the library and a database-backup directory. A database backup created on the day of that recovery point was listed there. Seeing the file in the repository was useful evidence, but it was not a database restore.

I then downloaded one photo and one video from that recovery point into a separate test directory. Both decoded successfully. For each file, I calculated a SHA-256 hash and compared it with the corresponding current original. Both pairs matched.

The comparison itself can be as simple as:

```sh
sha256sum restored-file original-file
```

Matching hashes show that the two files compared are identical. They do not show that every file in the library is present. If an original has changed since the backup, a different hash may reflect that change rather than a damaged backup; the comparison has to use the expected version.

The backup was encrypted, so opening the recovery point also required the backup key. That is part of the recovery process, not an administrative detail to discover during an outage. The key needs its own secure, accessible recovery path.

## Testing the database

I also restored the database backup into an isolated PostgreSQL container and compared it with the live database. The tables, indexes, constraints and extension versions matched. The restored database contained three fewer media records; that difference is consistent with changes after the backup, but the comparison alone does not explain it.

The restore log contained no SQL errors, but an SSH timeout meant I could not check the restore process's final exit code. The database comparisons were encouraging, not a complete success receipt.

This was still not a complete application recovery. I did not start the application against the restored database and media, so I have not tested whether those pieces, the configuration and the application version work together after a restore.

External libraries need their own attention too. A media application can refer to files outside its main upload directory. A backup of the application directory alone does not prove those sources are covered.

The practical recovery checklist is therefore larger than the two samples:

- identify the application's active media and any external libraries;
- confirm that the relevant sources and a usable database backup are in the recovery point;
- recover representative files into a separate location and check them;
- keep the backup key and necessary configuration available independently;
- test the application and database together when an isolated environment is available.

## Retention is a separate decision

Changing the source selection affects future backups. It does not mean the old source immediately disappears from every existing recovery point or that its occupied space is reclaimed at once. That depends on retention, rotation and how the backup system stores shared data between versions.

The backup tool should manage that lifecycle. Deleting files directly inside its repository is not a shortcut to safe cleanup: it can damage the backup. Before shortening retention or removing versions, I want to know which recovery points remain and what they contain.

## The useful question

A successful backup report is valuable. It tells me the task completed, not that its source list still reflects the application after a migration.

The more useful question starts at the other end: **if the service disappeared today, what would I need to bring it back, and which recovery point contains each piece?**

In this case, checking the mount and backup selection clarified which library was active. Two recovered files matched their originals, and the isolated database test provided additional evidence, with the exit-code limitation noted above. The full application restore remains untested. That is a narrower conclusion than “the backup works”, but it is one I can support.

## References

- [Immich: Backup and Restore](https://docs.immich.app/administration/backup-and-restore/) explains the database and media components needed for recovery.
- [Synology: Restore individual files from a Hyper Backup backup](https://kb.synology.com/en-ca/DSM/tutorial/How_to_browse_and_restore_Hyper_Backup_backup_data_in_hbk_file_format) describes browsing a recovery point and retrieving selected files.
