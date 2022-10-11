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

* Download files from archiveDOTorg based on listed output (e.g. get all iso, zip, and 7z files in this case). This option is typically fine if you're not working with the Internet Archive a great deal, and only occasionally pull the odd thing down every once in a while:
```bash
wget -A iso,zip,7z -m -p -E -k -K -np https://archive.org/download/GCUSRVZ-Arquivista
```

* If you're doing this any kind of frequently though, make it easy on yourself and install the [Internet Archive's command line tool](https://archive.org/developers/internetarchive/installation.html#binaries) (direct link to download page for their binaries). Once saved onto the server (wherever it's going to live permanently), use the same method described in [this section](https://github.com/teambvd/UnRAID-Performance-Compendium/blob/main/general/commonIssues.md#customizing-unraids-cli-and-running-commands-natively) of the 'Common Issues' page to make it act as a 'native' unraid CLI command.
 	* Now that you've 'installed' (sorta... semantics!) the `ia` cli tool, you can download at your leisure. Their [reference section](https://archive.org/developers/internetarchive/cli.html#) has some good starting points, but just to give you an idea of the power you have available... Lets just say you decided you want every PS2 game ever released in the U.S. - you search the archive, and it seems like [it's in 3 parts](https://archive.org/search.php?query=title%3A%28Playstation+2%29+AND+creator%3A%28AlvRo%29&sort=titleSorter). The links look like:
	  `https://archive.org/details/ps2usaredump1`
	  
  * You *could* just type in three separate terminal windows
    ```bash
    ia download gamecollectionlinks
    # hey, that was fast!
    ia download ps2usaredump1
    # now wait a long time
    ia download ps2usaredump1_20200816_1458
    # ugh, wait some more
    ia download httpsarchive.orgdetailsps2usaredump3
    # great, now maybe my great grand-kids will get to enjoy em at least
    ```
    
  * Or instead, you could install the `parallel` utility from the `nerd pack`,  and make the server do all the work for you at once (albeit, without the pretty progress bars, if you care about that kind of thing...). **Be sure you're already in the directory you want to download the data to before executing of course!**
  ```bash
  /path/to/ia search "ps2usaredump1" --itemlist | parallel '/path/to/ia download {} --checksum --retries 10'
  ```
    * You're now downloading all three at once, and in a single window - hope you've got enough space on your cache drive(s)! Do note, you will likely need to specify the full actual path of the ia tool regardless of however you've made the tool available to be called directly (where we just called `ia download` above), as the `parallel` command doesn't run ia directly as your account/user. To explain the command (see links above for full description plz)
		  * At first we have to search for the "ps2usaredump1" collection (from the URL above), then listing its items. If you ran the command on it's own, it'd show
	     ```bash
      gamecollectionlinks
      httpsarchive.orgdetailsps2usaredump3
      ps2usaredump1
      ps2usaredump1_20200816_1458
      ```
    * Now that we have our list, we call parallel to 'do this for each' (e.g. insert each of the 4 lines output in place of the `{}`). The `--checksum` argument looks for any  existing files of the same name, compares the two, and only downloads if the local copy doesn't match the archive. Adding `--retries 10` does just what it sounds like; if the connection times out, try again (x10). These are both especially helpful if you're on a typical North American ISP that tends to like to randomly drop offline for absolutely no reason whatsoever.


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
