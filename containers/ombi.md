# ***Ombi on ZFS***

## "Ombi's UI just doesn't seem  to work anymore..."

Many have noticed that over time and as your media library continues to grow, you eventually get to the point that it just seems Ombi's no longer fit for purpose. Browsing the UI is fine, but then when your users try to make any new requests, they just never get saved, or perhaps parts of it get saved (e.g. your manually added sonarr 'custom parameters', but not the request approval itself), and so on.

It's this that starts people searching for either an alternative application, or if they're up for troubleshooting, to finding [this page](https://docs.ombi.app/guides/migrating-databases/) which walks them through converting everything over to MariaDB... 

Don't get me wrong, MySQL is great and all, but that's like proposing the use of a telescoping crane usually reserved for mining work to move a box that's just a little too heavy to lift by hand.

## Figuring out the default config to get a baseline

Let's take a look at our persistent data directory (usually appdata for UnRAID users) to see what we have on hand:

```bash
root@server:/ombi# ll
total 1.8M
drwxrw-rw-  7 nobody users   10 Apr 17 10:58 ./
drwxrw-rw- 48 nobody users   48 Apr 16 21:38 ../
drwxr-xr-x  3 nobody users    3 Nov 17 10:20 .aspnet/
drwxr-xr-x  3 nobody users    3 Nov 17 13:39 .dotnet/
drwxr-xr-x  2 nobody users   33 Apr 17 04:23 Logs/
-rw-r--r--  1 nobody users 1.3M Apr 17 06:00 Ombi.db
-rw-r--r--  1 nobody users 3.4M Apr 17 10:15 OmbiExternal.db
-rw-r--r--  1 nobody users  24K Mar  1 13:21 OmbiSettings.db
drwxr-xr-x  2 root   root     2 Jun 24  2021 custom-cont-init.d/
drwxr-xr-x  2 root   root     2 Jun 24  2021 custom-services.d/
```

Three separate DB's (information on their individual uses can be found [here](https://docs.ombi.app/info/faq/#database-uses)) - checking their config:

```sql
root@server:/ombi# sqlite3 Ombi.db 'pragma journal_mode;'
delete
root@server:/ombi# sqlite3 Ombi.db 'pragma synchronous;'
2
root@server:/ombi# sqlite3 Ombi.db 'pragma page_size;'
4096
```

Delete is the absolute slowest of the options, and we're doing this as a fully synchronous operation, each action waiting on the outcome of the last to be committed fully, each time we do *anything* in the application other than browsing pages. For reference, here's the application's workflow for newly added requests:

<img src="https://docs.ombi.app/assets/images/request_flow.png" />

Now imagine you're requesting an entire series of episodes, 10 seasons with 20 episodes each. Each one of those episodes has to be tracked individually in the DB, and we've only got 4KB of memory to work with -  looking at the above workflow, you can imagine how many database actions this could add up to... Each time that 4KB fills, we have to commit, each time we have an episode to add to the db we have to commit, and every time we complete, we have to delete the 'log' that said we had a commit to do.

It's no wonder the database completely locks up once you've got multiple users and thousands of items in your media library!

## You don't need a crane, just a dolly

SQLite's more than capable of handling this as long as we give it the proper resources it needs to do so.

We'll use the WAL (Write-Ahead Log) logging mechanism instead of TRUNCATE as it's more performant than both `truncate` and `delete` and we don't really care about the additional files being generated additional files generated as we would with WAL, as well as bumping the page size to max to minimize our number of unnecessary writes:

```sql
# Do this for each of the three .db files
sqlite3 Ombi.db 'pragma page_size=65536; pragma journal_mode=wal; VACUUM;'
```

(this setting doesn't stick and can be ignored for now) - Just like with Sonarr, we change sync to 'most' as we depend on ZFS for db consistency - helps lower the number of blocking ops on the database:
```shell
sqlite3 Ombi.db 'pragma synchronous = NORMAL;'
```

## Filesystem settings recommendations
See [Arr's on ZFS](https://github.com/teambvd/unraid-zfs-docs/main/containers/sonarrRadarrLidarr.md)
