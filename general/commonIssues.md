## Common Issues

**Table of Contents**
- [My zpool is busy but I cant tell why](#my-zpool-is-busy-but-i-cant-tell-why)
  * [Customizing UnRAIDs CLI/shell, and running commands natively](#customizing-unraids-cli-and-running-commands-natively)
- [Unable to modify filesets status](#unable-to-modify-filesets-status)
  * [Sledgehammer approach](#sledgehammer-approach)
- [Dataset is busy](#dataset-is-busy)
  * [Dealing with PID Whackamole](#dealing-with-pid-whackamole)

### My zpool is busy but I cant tell why

There's a tool for this now, created by the same person who created sanoid/syncoid (Jim Salter), called `ioztat` (think '[iostat for zfs](https://github.com/jimsalterjrs/ioztat)'). It's not available through the nerd/dev packs (it's a relatively recent-sh creation), and may well continue to be updated anyway, so we'll install directly from git.

You'll likely want to  a fileset which we can use to store the tool - for myself, I have all little git tools in one fileset, so did something like this:
  ```bash
  zfs create wd/utility/git -o compression=zstd-3 -o xattr=sa -o recordsize=32K
  
  # now we're going to pull down the code into our fileset
  cd /mnt/wd/utility/git
  git clone https://github.com/jimsalterjrs/ioztat
  
  # you should see it listed there now
  ls -lhaFr
  drwx------  3 root   root       8 Mar 14 11:29 ioztat/
  
  # don't forget to make it executable
  chmod +x ioztat/ioztat
  ```

Now, you can just execute the command, but only currently from the directory it's saved in. I've a bunch of random tools installed, and there's no way I'd remember all of these. One of the bigger annoyances I've had with unraid is how complicated it is to customize in a way that survives a reboot (recompile the kernel to install a tool a few KB in size? Ugh). We could just link them to our go file or something, but if you're like me, your go file is messy enough as it is...

#### Customizing UnRAIDs CLI and running commands natively

What I've done instead is to create files (placing them all inside of a folder called 'scripts', which I also put in the 'utility' fileset) that contain all my cli/shell 'customizations', then have unraid's User Scripts run through them at first array start. While they don't * Technically * persist across reboots, this method sufficiently works around it (annoying, but solvable):
  ```bash
  # create the files
  nano /mnt/wd/utility/scripts/createSymlinks.sh
  
  # create your symlinks - here's a small subset of mine - 
  
  # # Shell profiles
  ln -sf /mnt/wd/utility/git/.tmux/.tmux.conf /root/.tmux.conf
  cp /mnt/wd/utility/git/.tmux/.tmux.conf.local /root/.tmux.conf.local
  ln -sf /mnt/wd/utility/bvd-home/.bash_profile /root/.bash_profile
  # # Bad ideas
  ln -sf /mnt/wd/dock/free-ipa /var/lib/ipa-data
  # # CLI tools and Apps
  ln -sf /mnt/wd/utility/git/ioztat/ioztat /usr/bin/ioztat
  ln -sf /mnt/wd/utility/git/btop/btop /usr/bin/btop
  ```

I actually have multiple files here, each one being 'grouped' - if it's just linking a tool so it's available from the shell, that's one file, another is for linking a few privileged containers I've got running to where they 'should' be where they installed on bare metal, things like that. This way, if I screw something up and dont find out till I reboot some months later, it'll make figuring out **which** thing I screwed up (and where) far easier.

### Unable to modify filesets status

This happens when **something** is using either the dataset itself (at the filesystem level), or some of the contents within it. Common causes include:

* Application using fileset contents
* Network access (mounted via SMB/NFS/iSCSI)
* If attempting deletion: 
	* The filesystem is currently `mounted` by your host (unraid)
	* Snapshots currently exist for the fileset

#### Sledgehammer approach

For unmount issues, you can bypass the userspace restrictions by going directly to ZFS and issuing a rename; it's important to note that this could impact any applications using it, only recommended when you're either certain there's no active access, or if you plan on deleting the fileset anyway after unmounting

```bash
zfs rename tank/filesetName tank/newFilesetName
zfs destroy tank/newFilesetName

# be cognizant of potential impact on nested filesets and their mountpoints, such as tank/datasetName/datsetChileName
```

### Dataset is busy

* This usually due to a process (PID) using that fileset or some part of it's contents. You can verify whether the below example, where I have my pool `wd` mounted to `mnt` and am trying to figure out what's using my `nginx` container:
  ```bash
  root@milkyway:~# ps -auxe | grep /mnt/wd/dock/nginx
  root      63910  0.0  0.0   3912  2004 pts/0    S+   12:19   0:00 grep /mnt/wd/dock/nginx <croppedForBrevity>
  ```

* You then simply kill that PID:
  ```bash
  kill -9 63910
  ```

#### Dealing with PID Whackamole

If a new PID is spawned almost right after you've killed one for the same process, you've likely got some kind of automation running - 
* **Autosnapshot tools** - sanoid/syncoid/auto-snapshot.sh, anything that automatically handles snapshot management and is currently configured on the system can cause this. If you have it, kill *that* process (e.g. sanoid) first, then retry.
	* As an alternative, edit the config (such as your `sanoid.conf`) so it's no longer set to snapshot that fileset, restart the tool (sanoid here), and try again
* **intermittent commands** - you can try `umount -l /mnt/path` to do a `lazy` unmount; pretty commonly needed for a variety of reasons.
