# The case for ZFS on UnRAID

- [Whats UnRAIDs purpose](#whats-unraids-purpose)
- [What are we running on our servers](#what-are-we-running-on-our-servers)
- [How UnRAID tries to address the performance gap](#how-unraid-tries-to-address-the-performance-gap)
- [ZFS to the Rescue](#zfs-to-the-rescue)
  * [Individualized performance characteristics](#individualized-performance-characteristics)
  * [Snapshot and backup strategies unique to the application](#snapshot-and-backup-strategies-unique-to-the-application)
  * [Clone your data for multiple instances of one container/VM](#clone-your-data-for-multiple-instances-of-one-container-vm)
- [A word of caution](#a-word-of-caution)

## Whats UnRAIDs purpose

There are many ways to think of 'storage performance', but one of the most common is described here by Nexenta:
![Nexenta](https://blogdotnexentadotcom.files.wordpress.com/2016/06/pic2.png?w=487&h=268)

The idea here being that you can place yourself at any one point within the triangle, but that means you're not getting 'the maximum of any one of the three'. You're instead receiving some sort of balanced approach, a compromise really, that matches your specific needs. I think the problem UnRAID is trying to solve isn't **quite** a match for this way of thinking. Instead, we can think of the 'Good vs Fast vs Cheap' model:

![GoodFastCheap](https://larrycuban.files.wordpress.com/2015/06/good-fast-cheap.jpg)

Any NAS's goals are to store as much data as possible. So for our purposes we can equate the above to:

* Good = Durable/Reliable
* Cheap = Running Costs
* Fast = IO performance

With the array, we've firmly chosen low running costs and durability over performance - if you want to have a hundred terabytes of data, all immediately available (so no tape), and with some semblance of redundancy, you'd be hard put to find a better option out there. 

## What are we running on our servers

Each application you run on your server will likely have a unique set of workload requirement - some (extremely limited) examples:

|   Workload	|   Example	|   IO Pattern	|
|---			|---			|---			|
|   Bulk Media	|   Movies/Music/etc	|   All sequential, large block, write once / read many	|
|   Web Apps	|   Grafana/Gaps/Dashboards	|   Writes infrequently, various sizes, random reads often	|
|   DBs	|   Postgres/MariaDB/SQLite	|   Reads and writes frequently, small block IO	|

We've got the bulk media part covered by the array, but everything else? The array's not terribly well suited for it. The array is comprised of all HDD's (assuming you're following LimeTech's recommendations - which you should!), and our IO operations per second are essentially limited to that of a single drive.

## How UnRAID tries to address the performance gap

As anyone who's ever tried to run virtual machines or containers directly on the array will likely attest, the UnRAID array just doesn't have the juice to power even a couple moderately needy containers. 

As you all are likely aware, this is where the cache pool comes in. Your only options here are a single disk XFS pool, or BTRFS if you wish to pool multiple drives (either for additional performance, or redundancy). We want our storage to have durability built in and allow for disk failures, leaving BTRFS the only option (at least as of August 2022). 

Don't get me wrong, BTRFS is a fantastic filesystem (in my opinion at least) - the internet is full of folks who'll parrot other's noting that "[BTRFS hasn't solved the write-hole problem](https://arstechnica.com/gadgets/2021/09/examining-btrfs-linuxs-perpetually-half-finished-filesystem/)", and that for that reason alone, is just completely inferior to other technologies. I disagree, but that's not really the point in this case.

The problem with this is that BTRFS just isn't as performant for the random IO loads enthusiasts (that's us) often find we need from that cache. You'll find that even [Josef Bacik](https://www.linkedin.com/in/josef-bacik-83508a7/), a Facebook employee (the largest known consumer of BTRFS) and one of the most prolific BTRFS developers out there, has essentially stated "[We'll likely never move our mysql databases to BTRFS](https://youtu.be/U7gXR2L05IU?t=3032)" - I could've swore he'd said 'just don't run databases on BTRFS', though maybe that was in a different talk (that's since been redacted?). The maximum latency values experienced with BTRFS are just too high when compared to alternatives, as much as 5x or more when compared to XFS (which is Facebook's choice).

## ZFS to the Rescue

This is where the flexibility+performance of ZFS can help us. Just like with BTRFS, each application can have it's own fileset (command line only in UnRAID), allowing you to customize how the storage behaves to the specific applications needs. But ZFS's caching system means it doesn't have the same kind of penalties (nor to the same extent) that one experiences when running those workloads on BTRFS.

### Individualized performance characteristics

Have an application that only reads once upon startup (e.g. pulls it's config and loads into memory) and doesn't do any further filesystem activity until the next restart? You can configure a higher level of compression on it so it takes up as little space as possible, setting the fileset to not cache at the filesystem level at all so those resources stay free for other applications which need them.

Maybe you're running Nextcloud, Paperless, or TandoorRecipes, applications that require a backing MariaDB/Postgres/(etc) database in order to be performant at scale? Well you can then ensure that the block/record sizes for your chosen DB's fileset match up with the page size of that database, specify what kind of data within that fileset will be cached, and how that cache operates to best squeeze every drop of performance your storage is capable of providing that database.

### Snapshot and backup strategies unique to the application

As you've each application within it's own dedicated filesystem, you're able  to be as granular as you wish (or not) with your backups. Choose to snapshot your highest change-rate data every 6 hours for a week, keep weekly snapshots for a couple months, and automatically replicate them off-site, while you only take weekly backups of those which mostly just read config data.

If you like pure command-line/cron, you can use something like [zap](https://github.com/Jehops/zap), where if you prefer a config driven option there are tools like [Sanoid/Syncoid](https://github.com/jimsalterjrs/sanoid) or [zfs-auto-snapshot](https://github.com/zfsonlinux/zfs-auto-snapshot) - any of which allows you full control over what snapshots get taken when, how often, how long they're kept, and where to send them for backup (if desired). You set it up once, and your backups are just automated from then on out (but please don't forget to test them occasionally!)

In the worst case scenario where your server goes completely down for any reason, you can simply start up the services on that backup server and limit your downtime whilst you work away on recovering whatever caused the failure to your main systems (hopefully it's not a flooded basement, ugh.)

### Clone your data for multiple instances of one container/VM

If you've ever had a home assistant upgrade go sideways, don't worry about it anymore - just create a clone of the fileset containing your VM/app data, give it it's own unique test network in UnRAID, and upgrade away. If it goes sideways? Simply destroy the clone, leaving your 'production' systems running while you research the failure.

Perhaps you've got a need to run a temporary machine for a guest in the house, but don't feel like setting up one from scratch, and **REALLY** don't want to risk their wreaking havoc on an install you've spent (years) perfecting? Just clone the VM's data, let them get their grubby paws all over it, and when they're done littering in your playground, throw it away - your source machine is still safe and sound.

## A word of caution

ZFS, while powerful, isn't for everyone - you'll need not only have the desire/drive to do some of your own background research and learning, but just as importantly, the **TIME** to invest in that learning. It's almost comically easy to have a much worse experience with zfs than alternatives by simply copy/pasting something seen online, while not fully understanding the implications of what that exact command syntax was meant to do.

The possibilities here are virtually endless - both for making your life easier, and for seeing massive performance gains. None of that, however, is worth the potential for lossing that data. As long as one's willing to put forth the time and effort into it though, the world's your oyster!
