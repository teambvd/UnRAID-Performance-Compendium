# ***Postgres on ZFS***

## **Table of Contents**

- [Primer](#primer)
  * [My usecase](#my-usecase)
- [General Recommendations](#general-recommendations)
- [Configuring Dataset and OS](#configuring-dataset-and-os)
- [Configuring Postgres](#configuring-postgres)
  * [A word on Autovacuum](#a-word-on-autovacuum)
- [Backups](#backups)
  * [Via pgAdmin](#via-pgadmin)
  * [Via CLI - psql](#via-cli---psql)
  * [Taking advantage of some ZFS](#taking-advantage-of-some-zfs)
    + [Sanoid Automated Snapshots](#sanoid-automated-snapshots)
    + [Pre and Post scripts](#pre-and-post-scripts)
- [For when youre ready to go pro](#for-when-youre-ready-to-go-pro)
- [References](#references)

## Primer

I got started with Postgres long after ZFS, with my introduction being [this talk](https://www.youtube.com/watch?v=T_1Zo4m4v_M) from one of the engineers over at Joyent. If you really wanna nerd out, and have 40 minutes to burn, it's worth a watch - some of the below comes directly from that talk. Another good place to look is [this article](https://www.2ndquadrant.com/en/blog/pg-phriday-postgres-zfs/), which walks you through initially poking around with both ZFS and Postgres. 

I've tried to give what I feel are some sane config values, along with just enough justification on them to explain without going **too** deep into it; this should hopefully provide the minimum detail necessary to allow for your own additional testing/tuning to best match your specific scenario. Please understand, every person's deployment is unique, and simply copying my config values and running with them may have unexpected results!

### My usecase

What I use postgres for, the DB sizes, and any workload information I can think of that might allow someone to equate my settings and usage to their own. DB Size equals DB + Temp files (as reported by pgadmin):


> * Nextcloud -  `6` users - `1.1m` files - `1991MB` DB size 
> * Paperless - `2` users - `1438` files - `14MB` DB size
> * OnlyOffice - Nextcloud connected only - `8MB` DB size
> * NextcloudTest - `2` users - `2.35m` files - `3.05GB` DB size - a random local test instance for perf tuning tests prior to putting into 'production'


## General Recommendations

There are several broad strokes I take with regards to any database implementation

* If optional (e.g. built in sqlite, json, whatever), **do I really need a full production database** (PostgreSQL, MariaDB), or will the built in option suit the need? For example, I run vaultwarden (~bitwarden), ombi, all the 'arr's, NginxProxyManager, and several others... but none of them really *need* anything more than sqlite provides, at least for the vast majority of users; with proper tuning of sqlite, it can be made to handle more than most give it credit for.
	* The thing to think about here is "how many actions do I foresee occurring at once?". For a password manager, I'm mostly just reading data, with the occasional add or change.
	* NginxProxyManager will rarely have a configuration change, and most of it's 'db activities' will simply be for (bi-monthly) cert updates or the occasional new proxy host.
	* An exception to this **might** be Lidarr (once it [eventually gets psql support](https://github.com/Lidarr/Lidarr/pull/2625)), namely due to how many unique records there end up being (at one per song, even a medium sized collection can get 'metadata heavy').
*  Simply "SSD" storage isn't really good enough for the best level of performance. **SATA introduces significant latency penalties that don't exist for PCIe based storage** (NVME/Optane). Always put your databases on storage that's the most performant you can afford for the best possible outcome. 
	* The amount of latency that can be added simply at the protocol layer (SATA) is more than enough to turn a snappy, responsive application UI into one that feels sluggish/dated
* **Snapshots alone are not enough to 'always' ensure the DB is up to date at that time** - you need to take special precautions when backing up DBs, as they operate much like ZFS does. For ZFS, you have the ZIL; for DBs, this is called a WAL (write ahead log), another sort of 'abstraction layer' meant to improve performance (though designed without the robustness of ZFS in mind).
* **Mo Memory = MO BETTAH** - but you're running ZFS right? So you already knew that.
* Specific to Postgres, **Get familiar with [pgAdmin](https://www.pgadmin.org/)** - while I typically only use it for reporting, it can be helpful for TONS of situations. It'll report statistics for individual databases (size, number of current connections, etc), allow you to take backups via GUI if you prefer, and is just a fantastic way to visualize what's happening 'beneath the covers' of your application.

## Configuring Dataset and OS

* In your BIOS, disable NUMA, and ensure memory interleaving is enabled - [ref](http://rhaas.blogspot.com/2014/06/linux-disables-vmzonereclaimmode-by.html) - (NOT transparent huge pages)

* Refrain from using swap (on unraid, just don't install the swap plugin)

* Configure the dataset where your postgres data will reside with properties tuned specifically for pg's read/write behavior:
 
  ```bash
  zfs set compression=zstd-3 recordsize=16K atime=off logbias=throughput primarycache=metadata redundantmetadata=most zpool/dataset
  ```

As with SQLite, we use the same compression and logbias settings, but this time we're setting the primarycache to `metadata`. This means we're caching the filesystem pointers that say 'this is where the data is', but not the data itself. 

**Note**, the `redundantmetadata` setting should only be modified in a pool with redundancy (mirror, raidz, etc), and where `checksums` have not been disabled (it's on by default).

* Edit your go file (nano /boot/config/go) adding:

  ```bash
  echo 1 /sys/module/zfs/parameters/zfs_txg_timeout
  ``` 

  The above is a global parameter (see [github docs](https://openzfs.github.io/openzfs-docs/Performance%20and%20Tuning/Module%20Parameters.html#zfs-txg-timeout))- even if the pool you plan to put your DB on is redundant, should you have any other non-redundant pools, recommend avoiding this setting. While it'll likely help performance, in combination with the other settings we're using (both above and below), there's an increased risk (however small) of data loss in certain circumestances.

<div class="warning" style='padding:0.2em; background-color:#3396ff; color:#0033cc'>
<span>
<p style='margin-top:0.2em; text-align:left'>
<b>NOTE</b></p>
<p style='margin-left:0.5em;'>
All of the settings mentioned your ZFS dataset configuration should be set prior to putting any data on them. If data already exists, you'll need to create a new dataset and migrate the data to that dataset in order to fully enjoy their benefits.
<p style='margin-bottom:0.5em; margin-right:0.5em; text-align:right; font-family:Georgia'>
</p></span>
</div>

## Configuring Postgres
* ZFS (and BTRFS for that matter) are both copy-on-write filesystems, while postgres does a version of this itself as well by default (to allow for the typically journaled filesystems it's installed on); disabling this behavior removes excessive writes, increasing performance:
  ```sql
  ALTER SYSTEM SET full_page_writes=off;
  ALTER SYSTEM SET synchronous_commit=off;
  CHECKPOINT
  ```

* Edit `postgresql.conf` to better take advantage of that TB of memory you bought - some of this is (strictly speaking) unnecessary for most of us, as if you have compression enabled, the databases tend to be pretty small (more on that below):

  ```sql
  shared_buffers = 2048MB
  huge_pages = try
  work_mem = 64MB
  maintenance_work_mem = 64MB
  max_stack_depth = 16MB
  max_files_per_process = 8000
  effective_io_concurrency = 200
  maintenance_io_concurrency = 200
  max_worker_processes = 24
  max_parallel_maintenance_workers = 8
  max_parallel_workers = 24
  random_page_cost = 1.1
  max_wal_size = 1GB
  min_wal_size = 80MB
  checkpoint_completion_target = 0.9
  ```

Shared buffers is effectively the 'total memory allocation' for PG (among all it's processes). Work mem is the max memory that may be used by an individual worker, so it's important to keep that shared buffers value in mind when setting this. In the above, I've allocated `2GB` to PG; we have `24` workers and `8` maint workers, giving us `32 * 64MB = 2048MB`.  With autovacuum disabled, maint workers effectively just carry out indexing tasks, so we don't need a great many of them (more on autovacuum below). Keep an eye on your DB sizes (as reported by postgres, NOT the filesystem) - if you're allocating memory beyond what your DB's total sizes are, you're simply wasting resources!

The IO concurrency setting of 200 allows for us to take better advantage of all those IOPs of our NVME; we could theoretically increase this even further, but for home use, there's a point of diminishing returns... And if you're using SATA SSD's, there's a greater chance that increasing this too high would result in excess CPU utilization due to iowait. The `200` value should be safe for even the most basic NVME, while SATA might be better off with a `100` setting, depending on the device's performance characteristics.

For reference, the above settings have proven highly performant for myself, and I currently have DBs for Nextcloud, OnlyOffice, Paperless, and Gitea (there's vaultwarden as well, but it's needs can be suitably met by SQLite)

* Now restart postgres:

  ```bash
  psql restart
  ```

<div class="warning" style='padding:0.2em; background-color:#3396ff; color:#0033cc'>
<span>
<p style='margin-top:0.6em; text-align:left'>
<b>NOTE</b></p>
<p style='margin-left:0.6em;'>
Unlike the ZFS settings, all of the settings mentioned for postgresql.conf can be changed even after the DB has been created and still reap the benefits. If you're not getting the performance you desire, you can test and tweak at will; simply stop whatever applications are using PG, make the change to the config file, then restart the PG container (and start the application container back up as well following). There is unfortunately no 'one size fits all', but the above can be a healthy starting point. <b>Tune, test, evaluate - rinse and repeat!</b>
<p style='margin-bottom:0.2em; margin-right:0.2em; text-align:right; font-family:Georgia'>
</p></span>
</div>

### A word on Autovacuum

Many will recommend also uncommenting the `autovacuum = on` line within  `postgresql.conf`. For truly "production" deployments, this is a must. However, in smaller scale deployments (such as my own), I prefer to manually vacuum the databases myself. This has two benefits:

1. I know exactly when the DBs are going to be vacuumed, and can plan accordingly. Vacuum's can have a significant performance impact, even if properly tuned (more on that in a moment), and my overly cautious nature means I always take a backup (pgdump) prior to manually running vacuum. If 'something goes wrong', I have a known good point in time to restore from.
2. Autovacuum, like everything else, has yet more tuning variables to be concerned with. Improper tuning can mean trying to track down intermittent performance issues indefinitely, only to (hopefully) eventually find that 'oh... it was running a vacuum'. 

I do all my maintenance at once on databases, usually as part of a scheduled outage where I'm already undertaking other operations (e.g. nextcloud upgrade, disk replacement, or quarterly {if I remember...} server reboot). It's just 'cleaner', at least for me.

## Backups

There are two 'standard' options when it comes to backups; using pgAdmin, and via commandline using 'psql'. In addition, there are countless other methods/tools as well, and we'll touch on < however many I do >:

### Via pgAdmin
1. Stop the application which is using the DB
2. Log into pgAdmin and connect to the database instance (your server's container)
3. Right-click on the DB to be backed up and select... wait for it... 'Backup'
4. Give it a memorable name, selecting `Tar` as the type, and (typically) `UTF-8` for the encoding type
5. Under the Data/Objects table, ensure all three options are selected (`Pre`, `Data`, and `Post`), and click `Backup`.

By default these are saved in pgAdmin's appdata directory under your user account, similar to:

```bash
/mnt/user/appdata/pgadmin/storage/email.address_domain/yourBackupName.tar
```

For a more detailed description (including pretty pitchaz!), see pgAdmin's [official documentation](https://www.pgadmin.org/docs/pgadmin4/development/backup_dialog.html)

### Via CLI - psql
1. Once again, stop the application using the DB
2. Open your postgres container's console by either right-clicking and selecting `console`, or alternatively from your terminal:
    ```bash
    docker exec -ti containerName bash
    ```
3. Create a backup directory in your persistent storage location, and log in with a user that has rights to the DB(s)
    ```bash
    / # mkdir /var/lib/postgresql/data/backups
    / # psql -U  usernameA
    usernameA=# <blinking cursor here>
    ```
4. Create your backup using `pg_dump`:
    ```bash
    usernameA=> pg_dump -E UTF-8 -F t myDatabaseName > /var/lib/postgresql/data/backups/myDatabaseNameAndDate.tar
    ```

The dumps are placed wherever your appdata is located at for postgres, and available for restore/testing. If all of your applications are down for something of a 'global maintenance', you can use pg_dumpall (though I typically don't recommend it for it's lack of granularity as it tends to cause confusion for some):

```bash
 usernameA=> pg_dumpall > allDBs-date.tar
```

### Taking advantage of some ZFS

***WORK IN PROGRESS / TBD - I really need to finish documenting this!*** 

We're going to use ZFS snapshots for our backups. Since we're using WAL in combination with zfs, this *could* be considered optional by some - the ZIL keeps the ZFS filesystem layer consistent, and the WAL keeps the DB application layer consistent. Paranoia, however, is healthy when it comes to data loss prevention.

The only difference between the above and below is that they're not necessarily 'both consistent at the same time'. Because of this, when the DB starts up, it may start in recovery mode - it replays the logs from WAL as needed.

 #### Sanoid Automated Snapshots

 #### Pre and Post scripts
 
 In sanoid.conf, use the below to let pg know you intend to start and stop a backup - recoveries are faster this way (not implemented in my setup, felt too much like 'work'):
 
* **Pre-script** - 

  ``` sql
  pg_start_backup('backup_name')
  ``` 

* **Post-script** - 

  ``` sql
  pg_stop_backup
  ``` 

## For when youre ready to go pro

You almost certainly don't need these, but at least from my work experience, these are the best in the business - allowing for point-in-time recovery (e.g. your backup was taken at noon, you can recover to any individual second prior, such as 11:54 and 32 seconds), highly configurable backups from multiple local/cloud locations, and so on:

* [PMM](https://www.percona.com/software/database-tools/percona-monitoring-and-management) -  If there were a product that was "Grafana + GoAccess, but for databases", they'd call it Percona Monitoring and Management. This is the kind of tool folks like the Nextcloud dev team would use to optimize the queries used by the application.
* [PGHero](https://github.com/ankane/pghero) - Think "PMM, but if I don't run a fortune 500 company". I run it [at home](https://github.com/ankane/pghero/blob/master/guides/Docker.md).
* [pg_probackup](https://github.com/wal-g/wal-g) - Dedicated highly configurable and excellent cataloging of backups, making efficient work of understanding what your backup history and recoverability points are. Believe it or not, sometimes just *finding* the right backup to recover from (or even knowing what your options are) is the hardest part, especially when managing thousands of databases across 10s/100s of instances, and this makes it wickid simple.
* [WAL-G](https://github.com/wal-g/wal-g) - Biggest win for wal-g is that it's the one tool I'm aware of that does (those things mentioned above), **AND** can handle Postgres, Maria/MySQL, Microsoft's SQLServer, and more recently even Mongo and Redis (crazy eh!?). Really, if you're in the devops world and haven't tried WAL-G, you owe it to yourself to give it a shot.
* [Metabase](https://github.com/metabase/metabase) - asdfa
* [Chartbrew](https://github.com/chartbrew/chartbrew) - Like metabase, just 'a bit less'. Fewer crazy features, fewer options, resulting in fewer resources used by the application. Great for the small/medium business market!

## References
[PSQL tuner](https://github.com/jfcoz/postgresqltuner) - Dockerfile designed around finding issues and some areas for optimization

[pgTune](https://pgtune.leopard.in.ua/#/) - Input your containers resource allocations (memory, CPU, storage type) and use case, helps provide a starting point for tuning `postgresql.conf`

[pgConfig](https://www.pgconfig.org/) - More tunable output than pgTune, but based on the same underlying code/work. The real value here lies not in the 'recommended tuning' output it provides (which is just average, as all 'general' recommendations must be without having a deeper understanding of the workload), but instead the explanations given for what the config variables specific impacts are when you select their dropdown

