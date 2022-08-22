## Table of Contents:

- [Plex related](#plex-related)
- [Unraid host specific](#unraid-host-specific)
  * [Restart the UnRAID UI from cli](#restart-the-unraid-ui-from-cli)
  * [Get total number of inotify user watches used](#get-total-number-of-inotify-user-watches-used)
    + [List all current inotify users and totals](#list-all-current-inotify-users-and-totals)
  * [Start or stop the array from the CLI](#start-or-stop-the-array-from-the-cli)
  * [Spin down all drives in the array](#spin-down-all-drives-in-the-array)
- [General Linux stuff](#general-linux-stuff)
  * [List top 10 biggest files in all sub-dirs of current directory](#list-top-10-biggest-files-in-all-sub-dirs-of-current-directory)
  * [Bulk Downloading from Archive.org](#bulk-downloading-from-archivedotorg)
  * [Remove the first 7 characters from all files in a dir](#remove-the-first-7-characters-from-all-files-in-a-dir)
  * [Get a breakdown of file sizes for a given directory including subdirs](#get-a-breakdown-of-file-sizes-for-a-given-directory-including-subdirs)
- [Personal tools stuff](#personal-tools-stuff)
  * [Chrome to table generation](#make-a-table-showing-links-displayed-on-page-)
  * [Fix time machine stuck in stopping](#kill-bvdmbp-time-machine-locks-so-it-can-pick-back-up)


## Plex related

- Increase memory allocation to the container (docker extra params)
``--shm-size=2048m`
- Location of database
``/mnt/wd/dock/plex/Library/Application Support/Plex Media Server/Plug-in Support/Databases/com.plexapp.plugins.library.db`
- Stopping plex service (LSIO container)
  ```bash
  cd /var/run/s6/services
  s6-svc -d plex
  ```
- Run repair on the DB
  ```bash
  cd "/config/Library/Application Support/Plex Media Server/Plug-in Support/Databases" 
  "/usr/lib/plexmediaserver/Plex\ SQLite" com.plexapp.plugins.library.db ".output recover.out" ".recover"
  ```

## Unraid host specific

Stuff that's specific to unraid as opposed to generalized linux stuff

### Restart the UnRAID UI from cli

* If the GUI fails to load in your browser, becomes slow over time, etc
  ```bash
  /etc/rc.d/rc.nginx reload
  ```

### Get total number of inotify user watches used

* Figure out if you need to increase the max watchers settings in tips and tweaks
  ```bash
  find /proc/*/fd -lname anon_inode:inotify | cut -d/ -f3 | awk '{s+=$1} END {print s}'
  ```

#### List all current inotify users and totals

* This is a more detailed version of the above, listing what application/process is using watchers, how many for each individual process individually, so you can scope in more specifically when troubleshooting
  ```bash
  find /proc/*/fd -lname anon_inode:inotify | cut -d/ -f3 | xargs -I '{}' -- ps --no-headers -o '%p %U %c' -p '{}' | uniq -c | sort -nr
  ```

### Start or stop the array from the CLI

```bash
#start
wget -qO /dev/null http://localhost:$(lsof -nPc emhttp | grep -Po 'TCP[^\d]*\K\d+')/update.htm?cmdStart=Start

#stop
wget -qO /dev/null http://localhost:$(lsof -nPc emhttp | grep -Po 'TCP[^\d]*\K\d+')/update.htm?cmdStop=Stop
```

### Spin up a drive manually
* Manually spinning down a drive
  ```bash
  #where 'X' equals your drive letter
  sdspin sdX up
  ```

### Spin down all drives in the array

* You can choose to forcefully spin down all drives. If there's an active session, no worries - they may experience a temporary interruption, but the drive needed will automatically spin back up:
  ```bash
  for dev in /dev/sd?; do /usr/local/sbin/emcmd cmdSpindown="$(grep -zoP "(?<=name=\")[a-z0-9]+(?=\"\ndevice=\"${dev: -3})" /var/local/emhttp/disks.ini | tr -d '\0')"; done
   ```

## General Linux stuff

(that works just the same on UnRAID)

### List top 10 biggest files in all sub-dirs of current directory

```bash
find $PWD -type f -printf '%s %p\n' | sort -nr | head -10
```

### Bulk Downloading from archiveDOTorg

* Download files from archiveDOTorg based on listed output (e.g. get all iso, zip, and 7z files in this case)
```bash
wget -A iso,zip,7z -m -p -E -k -K -np https://archive.org/download/GCUSRVZ-Arquivista
```

### Remove the first 7 characters from all files in a dir

`for f in *; do mv "$f" "${f:7}"; done`

### Get a breakdown of file sizes for a given directory including subdirs

  note - this can take a **very** long time to complete if you've got a ton of tiny little files
  ```bash
  find /mnt/whatever/directory -type f -print0 | xargs -0 ls -l | awk '{ n=int(log($5)/log(2)); if (n<10) { n=10; } size[n]++ } END { for (i in size) printf("%d %d\n", 2^i, size[i]) }' | sort -n | awk 'function human(x) { x[1]/=1024; if (x[1]>=1024) { x[2]++; human(x) } } { a[1]=$1; a[2]=0; human(a); printf("%3d%s: %6d\n", a[1],substr("kMGTEPYZ",a[2]+1,1),$2) }'
  ```

## Personal tools stuff

Probably not useful to anyone else

### Make a table showing links displayed on page:
```html
var x = document.querySelectorAll("a");
var myarray = []
for (var i=0; i<x.length; i++){
var nametext = x[i].textContent;
var cleantext = nametext.replace(/\s+/g, ' ').trim();
var cleanlink = x[i].href;
myarray.push([cleantext,cleanlink]);
};
function make_table() {
    var table = '<table><thead><th>Name</th><th>Links</th></thead><tbody>';
   for (var i=0; i<myarray.length; i++) {
            table += '<tr><td>'+ myarray[i][0] + '</td><td>'+myarray[i][1]+'</td></tr>';
    };
 
    var w = window.open("");
w.document.write(table); 
}
make_table()
```

### Kill BVDMBP time machine locks so it can pick back up

```bash
kill -9 $(smbstatus | grep BVDMBP | grep timemac | cut -d' ' -f1 | uniq)
```
