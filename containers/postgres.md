# ***Postgres on ZFS***

I got started with Postgres long after ZFS, with my introduction being [this talk](https://www.youtube.com/watch?v=T_1Zo4m4v_M) from one of the engineers over at Joyent. If you really wanna nerd out, and have 40 minutes to burn, it's worth a watch - some of the below comes directly from that talk.

Another good place to look is [this article](https://www.2ndquadrant.com/en/blog/pg-phriday-postgres-zfs/), which walks you through initially poking around with both ZFS and Postgres.

## Configuring Dataset/OS

* Configure the dataset where your postgres data will reside with properties tuned specifically for pg's read/write behavior:
 
  ```bash
  zfs set compression=zstd-3 recordsize=16K atime=off logbias=throughput primarycache=metadata redundantmetadata=most zpool/dataset
  ```

As with SQLite, we use the same compression and logbias settings, but this time we're setting the primarycache to `metadata`. This means we're caching the filesystem pointers that say 'this is where the data is', but not the data itself. 

**Note**, the `redundantmetadata` setting should only be modified in a pool with redundancy (mirror, raidz, etc).

* Edit your go file (nano /boot/config/go) adding:

  ```bash
  echo 1 /sys/module/zfs/parameters/zfs_txg_timeout
  ``` 

**Another Note**: this is a global parameter - even if the pool you plan to put your DB on is redundant, should you have any other non-redundant pools, I'd avoid this.

## Configuring Postgres
* ZFS (and BTRFS for that matter) are both copy-on-write filesystems, while postgres does a version of this itself as well by default (to allow for the typically journaled filesystems it's installed on); disabling this behavior removes excessive writes, increasing performance:
  ```sql
  ALTER SYSTEM SET full_page_writes=off;
  ALTER SYSTEM SET synchronous_commit=off;
  CHECKPOINT
  ```

* Now restart postgres:

  ``` bash
  psql restart
    ```

 ### Backups

***WORK IN PROGRESS / TBD - has to be done on a per connection basis and isn't persistent, which means this requires container modification*** 

 We're going to use ZFS snapshots for our backups. Since we're using WAL in combination with zfs, this is pretty much completely optional - the ZIL keeps the ZFS filesystem layer consistent, and the WAL keeps the DB application layer consistent. 

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

*References*
* [PSQL tuner](https://github.com/jfcoz/postgresqltuner) - Dockerfile designed around finding issues and some areas for optimization
* [PGTune](https://pgtune.leopard.in.ua/#/) - Input your containers resource allocations (memory, CPU, storage type) and use case, helps provide a starting point for tuning `pg_hba.conf`
