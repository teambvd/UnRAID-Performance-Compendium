# ***Sonarr / Lidarr / Radarr on ZFS***

Reference information:

> * Sonarr - `259MB` - 31,689 Episodes 
> * Radarr - `115MB` - 2460 Movies
> * Lidarr - `422MB` - 34,581  Songs


## Filesystem settings recommendations:
* Recordsize - `64K` - This'll make more sense when we get to tuning the DB
* Compression - `zstd-3` - I'd used `lz4` since opensolaris was released, but z-std has made me a believer - 2.57 compressratio, no noticeable performance differnce (on a modern CPU at least). Standard recommendation for all non-media datasets imo.
* Primarycache - `none` - the DB does it's own caching, so this would be duplicating resources, wasting them; normally you'd set this to `metadata` for a database, but sqlite's so small, there's limited gains (if any) from doing so, especially on NVME; again, more on this later
* atime - `off` - another 'do this for everything' setting... except, actually for everything, media or otherwise
* logbias - `throughput` - again, databases
* xattr - `sa` - recommended for all ZoL datasets

Note: These settings (in general) should be set at creation time. Modifying a setting doesn't change existing data and only applies to newly created data placed within that dataset. If data already exists, it's best to migrate that data to a newly created dataset with the appropriate settings. **DO NOT** make changes to an existing dataset while the data is being actively accessed (unless you know what you're doing).

## Application tuning

The fileset's been configured this way forever, and things had been humming along swimmingly. As times gone on and my library has grown, I've noticed my Sonarr instance had been getting incrementally slower over time. It used to be that when I first loaded the UI, it'd show up within ~2-3 seconds, it'd now take twice that or more. 

The biggest tell though was when I'd go to the History tab, usually to try to sort out what'd happened with a given series (e.g. an  existing series was for some reason updated with a quality I didn't expect, or maybe an incorrect episode was pulled, etc.). It'd take ~10+ seconds to load the first set of 20 most recent activities, then close to that for each subsequent page.

### Finding the Cause

I won't bore you too much with the details, but a trace revealed that the majority of the time spent was in waiting on a query response from the database, sqlite. Looking at the database size:
```bash
259M Mar 23 06:59 sonarr.db
```

Fairly large for an SQLite DB. Vacuum brought no tangible benefit either.  Next I checked the page size:
```bash
sqlite3 sonarr.db 'pragma page_size;'
4096
```

We're only able to act on 4KB of transaction data at a time? Bingo.

### The Solution

All the 'arr's use an sqlite backend - you'll need to acquire sqlite3 from the the Dev Pack, if not already installed. First, we need to verify the data base is healthy - **MAKE SURE YOUR CONTAINER IS STOPPED BEFORE PROCEEDING FURTHER**:
```bash
sqlite3 sonarr.db 'pragma integrity_check;'
```

This can take awhile, so be patient. If this returns anything other than `OK`, then stop here and either work on restoring from backup or repairing the DB.

Without boring you to death (hopefully), SQLite essentially treats the page_size as the upper bound for it's DB cache. You can instead set the 'cache_size', but this setting is lost each time you disconnect from the DB (e.g. container reboot), so that's kinda worthless for our needs. We're setting our to the maximum, `64KB`:
```bash
sqlite3 sonarr.db 'pragma page_size=65536; pragma journal_mode=truncate; VACUUM;'
```

This might take a bit, depending on the size of your sonarr.db and transaction log files. Once it's done, we need to re-enable `WAL`:
```bash
sqlite3 sonarr.db 'pragma journal_mode=WAL;
```



***WORK IN PROGRESS / TBD - has to be done on a per connection basis and isn't persistent, which means this requires container modification*** 

Most people don't 'notice' slowness related to commits to the DB, but if you've ever seen the UI sort of spaz out whenever you've added some new series with 18 seasons at 20+ episodes per, you've encountered it:

```shell
sqlite3 sonarr.db 'pragma synchronous;'
2
```

The default setting uses both [WAL](https://sqlite.org/wal.html) (Write-Ahead Log) as well as full sync, which double-commits transactions; this helps protect the database from inconsistency in the event of an unexpected failure at the OS level and can also slow down reads. Many OS's/filesystems (cough **windows** cough) treat in-memory data as disposable, neglecting the fact that a file may be a database with in-flight writes inside.

Given ZFS already checksums and syncs data itself (whole other topic), I'm a little less concerned about the database protecting itself and simply rely on the filesystem for this - if the DB becomes corrupted on ZFS, I've got bigger problems to worry about than rebuilding sonarr/radarr/lidarr. We can set this to 'normal' mode, for single commit operation:

```shell
sqlite3 sonarr.db 'pragma synchronous = Normal;'
```



References:
* [SQLite Page Sizes](https://www.sqlite.org/pragma.html#pragma_page_size)
* [WAL vs Journal in SQLite](https://www.sqlite.org/wal.html)
* [Previously, on Sonarr Dev](https://github.com/Sonarr/Sonarr/issues/2080#issuecomment-318859070)
