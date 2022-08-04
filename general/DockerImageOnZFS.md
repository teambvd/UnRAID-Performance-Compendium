## Run your docker image on ZFS using zvols

I recommend zvols (equivalent to virtual disks) over filesets for the docker image data for a variety of reasons - it'll be more performant at reduced resource cost (more efficient), integrates more smoothly with unraid (imo - less differences from the norm at the very least), and several others. Only thing you lose by doing so is the ability to individually snapshot specific containers; but containers are supposed to be disposable, with everything needing persistence mapped to volumes anyway, so what's the loss there right?

Lets get started:
```bash
# creating a 40GB zvol named 'imageLocation' nested under wd/docker
zfs create -V 40G wd/docker/imageLocation

# now partition the disk as GPT
gdisk /dev/wd/docker/imageLocation 

# Check the physical block size with the 'p' option, then 'o' to create a new partition, then 'w' to apply and exit
Command (? for help): p
Disk /dev/wd/docker/imageLocation: 83886080 sectors, 40960.0 MiB
Sector size (logical/physical): 512/8192 bytes...
...
Command (? for help): o
This option deletes all partitions and creates a new protective MBR.
Proceed? (Y/N): Y

Command (? for help): w

Final checks complete. About to write GPT data. THIS WILL OVERWRITE EXISTING
PARTITIONS!!

Do you want to proceed? (Y/N): Y
OK; writing new GUID partition table (GPT) to /dev/wd/docker/imageLocation.
The operation has completed successfully.

# You should now see a new zvol created listed both directly in dev as zd0, as well as several other places - ex:
ls /dev/zvol/wd/portvols/
zvoltest@


# Now format as xfs, while optionally specifying the blocksize a small tunable change optimizing this filesystem for our docker image data - the default is 4K, but some SSD's may be 8k native.  While can technically choose btrfs (among others), xfs is far more performant for high IOPs workloads. The '-f' option is also available must be before '-q' to force mkfs to ignore warnings
mkfs.xfs -b size=8k -d cowextsize=64 -q /dev/zd0

# Create a place to mount this disk - recommend you use somwhere under /mnt
sudo mkdir /mnt/dockerImage

# Finally, use configure
mount /dev/zd0 /mnt/dockerImage
```

## Common issues

### Cannot Set Blocksize

Should you see an error like the below even after confirming your disks physical size:
  ```bash
  mkfs.xfs: error - cannot set blocksize 8192 on block device /dev/tank/imageLocation: Invalid argument
  ```

This means your zpool's ashift value was set to something smaller than the blocksize you're trying to configure.  The common recommendation of "everyone should use `ashift=12`" means this will be a frequent occurrence (hopefully people actually check their physical block sizes before blindly following some random guy on the internet, right? ...RIGHT!?)

If that's the case, no big deal (and not a massive loss really for our use case here), just set it to 4k instead (for `ashift=12` at least).

### mkfs fails with "appears to contain an existing filesystem"

You forgot the `-f` option. Use this carefully - if you have no other zvols, no big concern, but otherwise, triple-check before proceeding.
