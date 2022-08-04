## Common Issues

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

#### Dealing with PID Whackamole?

If a new PID is spawned almost right after you've killed one for the same process, you've likely got some kind of automation running - 
* **Autosnapshot tools** - sanoid/syncoid/auto-snapshot.sh, anything that automatically handles snapshot management and is currently configured on the system can cause this. If you have it, kill *that* process (e.g. sanoid) first, then retry.
	* As an alternative, edit the config (such as your `sanoid.conf`) so it's no longer set to snapshot that fileset, restart the tool (sanoid here), and try again
* **intermittent commands** - you can try `umount -l /mnt/path` to do a `lazy` unmount; pretty commonly needed for a variety of reasons.
