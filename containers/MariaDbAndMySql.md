## MariaDB / MySQL

### THIS IS STILL UNDER CONSTRUCTION!!! I've probably only got this 30% completed; there's no harm in using what's in here, just note, it's not "done" yet.

### Assumptions

* All zpools on host included redundancy (i.e no unmirrored stripes, so raidz/mirror)
* Based on the [LSIO MariaDB](https://github.com/linuxserver/docker-mariadb) container (applies to any maria/mysql, just the location of config files may differ otherwise)
* (and of course, that you've enough system memory to allow for such additional 

#### Host changes

* Edit your go file (nano /boot/config/go) adding:

  ```bash
  echo 1 /sys/module/zfs/parameters/zfs_txg_timeout
  ``` 

(note assumptions section: all zpools should be redundant setting the above)

#### Creating ZFS dataset tuned for MySQL/Maria ####

```bash
zfs create wd/dock/mariadb -o recordsize=16K -o logbias=throughput -o primarycache=metadata -o atime=off -o xattr=sa -o compression=zstd-3
```

[**Explanation**](https://shatteredsilicon.net/blog/2020/06/05/mysql-mariadb-innodb-on-zfs/)
* Recordsize - InnoDB writes in 16KB chunks
* Logbias - recommendation from oracle re: [databases on ZFS](https://docs.oracle.com/cd/E19253-01/819-5461/givdo/index.html)
* Primarycache - the DB does it's own caching ('buffer pool'), so we should only cache filesystem metadata, not 'everything'
* atime - updating metadata every time a file is accessed is, in nearly all cases, absurd. Should be disabled as a general rule in ZFS unless you've a very specific need for it
* xattr - If you're on Linux, you should frankly have this configured at the pool level as it should be set for all (otherwise you'll likely encounter severe issues if you ever do anything like NFS or SMB access, at a minimum)
* Compression - zstd-3 is 'cheap' enough (resource-wise) that I feel it should be the standard when running any modern-ish CPU (at least for server use... there are caveats for workstations etc)

#### Creating your container ####

Example env variables:

```yaml
mariadb:
    container_name: mariadb
    image: mariadb
    ports:
      - 3306:3306
      - 3307:3307
    volumes:
      - /opt/mariadb:/var/lib/mysql
    environment:
      MYSQL_ROOT_PASSWORD: yourrootpasswordhere
      MYSQL_USER: homeassistant
      MYSQL_PASSWORD: yourstrongpasswordhere
      MYSQL_DATABASE: homeassistant
    restart: on-failure
```

#### MariaDB Tuning that's specific to use on ZFS

 **_Explain each: why we're disabling various maria data 'safety' features, etc**

```bash
innodb_doublewrite=0
innodb_checksum_algorithm=none
innodb_flush_neighbors=0
innodb_use_native_aio=0
innodb_use_atomic_writes=0
```

#### Tuning MariaDB Generally

  **_Run diff on currently running custom.cnf against base my.cnf, explain where necessary, and note that user shouldn't blindly copy/pasta from "some guy on the internet" without confirming with their own research_**
