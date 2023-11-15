---
title: "Oracle RMAN for Beginners – Part 5"
date: "2012-02-13"
categories: 
  - "oracle"
---

[Previously](/posts/2012/02/oracle-rman-for-beginners-part-4/ "Oracle RMAN for Beginners – Part 4") in this mini-series, I managed to dump and recover a database from a cold backup. Before I move on to similar practices with hot backups, a few terms and observations may well be in order.

## Terms and Conditions

A **restore** is the action of copying files back from a backup, prior to recovery.

**Recovery** is carried out after restoring the files and allows the database to be recovered in full or to a specific point in time, using backups of archived logs, and/or the current archived logs - which may not have been backed up yet.

A **full recovery** restores and recovers the database to an up to date point. No data will be lost or missing after the recovery.

A **partial recovery** restores and recovers the database to a point in time in the past. Data may be lost after the recovery has completed.

A **consistent backup** is a cold backup. I have been using these types of backup since the beginning of this series of articles.

A consistent backup is one which, when recovered, leaves the database ready to use with no further recovery required.

An **inconsistent backup** is a hot backup. We will be looking at these backups next as they are the ones most likely to be in popular use.

An inconsistent backup has to be restored and then rolled forward (recovered) to some specific point in time before the database can be opened for use.

An **available backup** is a backup, consistent or otherwise, which is available for use in recovering a database.

An **unavailable backup** is a backup, consistent or otherwise, which is deemed to be unavailable for use in recovering a database. In normal conditions, this is a forced action by the DBA who marks the backup as unavailable in order to prevent RMAN using it in a restoration

An **expired backup** is a backup which is registered in the catalogue and/or controlfile, but which is not found on disc or tape.

An **obsolete backup** is a backup for which the files remain on disc or tape, but the configured _retention period_ has caused the backup to become no longer required.

A **full backup** is a backup of the whole database.

An **incremental backup** is a backup which is not a full backup, but only contains blocks which have changed since the previous _level 0_ or _level 1_ incremental backup depending on whether the backup is _cumulative_ or _differential_.

## Retention and Deletion Policy

Oracle advise four (main) things when using RMAN to backup and recover your databases:

- Use a flashback recovery area
- Use a backup retention policy
- Use an archive log deletion policy
- Use a recovery catalogue.

The use of the FRA has been [covered already](/posts/2012/02/oracle-rman-for-beginners-part-1/ "Oracle RMAN for Beginners – Part 1") so no more need be said about that.

I will be discussing a recover catalogue later in this mini-series. For now, I'm sticking with the use only of the control file to record database backup details.

### Retention Policy

You may configure a retention policy for your backups to ensure that they remain available for as long as you deem necessary. You may define the retention as a recovery window - the number of days a backups should remain available for - or as a number of copies.

RMAN's default is a retention policy of one copy. This means that as each new database backup is created, the previous one becomes obsolete.

The method you use will depend on how your organisation likes to work. If the rules are that all backups should be kept for a fortnight then the command `configure retention policy to recovery window of 14 days` will cover that rule.

If, on the other hand, the rules demand that four copies of all backups should be retained then the command required will be `configure retention policy to redundancy 4`.

Whichever way that you configure your retention policy, backups that fall outside of your configured policy will automatically be flagged as obsolete by RMAN.

> **Note**: RMAN may well hold on to a backup beyond its retention period if any of the files within the backup are still required to be able to restore the database.
> 
> With backup optimisation configured on, files that have not changed since the previous backup will not be backed up. This speeds up the backup process but does mean that more backups may be required to allow everything to be restored.

### Archive log Deletion Policy

The archive log deletion policy is a similar system to specify which archived logs are _eligible_ for deleting from the FRA to recover space.

RMAN will delete those eligible logs as and when it needs to recover space in the FRA for further backups, or to avoid exceeding the setting of the `DB_RECOVERY_FILE_DEST_SIZE` for the database.

If you have mirrored copies of your archived logs stored outside the FRA, those will _not_ be deleted automatically by RMAN, it is your responsibility to delete them.

The deletion policy also controls whether or not archived logs are deleted from the FRA when the `backup archivelog all delete input` command is executed. Any archived logs that have not yet been backed up enough times, or have been backed up but for too few days, will _not_ be deleted. You can `force` the deletion though.

Examples of setting up the deletion policy are `configure archivelog deletion policy to applied on all standby` or `configure archivelog deletion policy to backed up 5 times to sbt` etc.

The default setting is `none` which means that archived logs are never deleted automatically from the FRA.
