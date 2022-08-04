# UnRAID + ZFS + This = Performance Bliss! (...Hopefully)

**A collection of documentation I've written up surrounding UnRAID's use while running the ZFS filesystem**

My 9-5 is essentially "this is slow, make fast" for a data management/security company, and since basically every application out there has **some** persistent data to be managed, some OS/hardware to run it, and some way to connect to it, I end up knowing just a liiiiittle bit (enough to cause problems at least!) about a whole lot. When I started homelabbing, I tended to apply the same methodologies gained from that time at work.

Performance lives on a continuum of "This straight up doesn't work" on down to "I can't find anything to complain about", with what's actually technically possible being somewhere in the neighborhood of "It's good enough that I probably wouldn't complain"; as applications have to run on a variety of hardware, it's almost never possible to set something up as a 'one size fits all' from a performance perspective. Performance tuning, then, is simply working towards the goal of configuring that application towards achieving or exceeding that "wouldn't complain" point while running in your specific infrastructure.

This will never be exhaustive, conclusive, or even finished (lol)... But with any luck, it'll give you what you need to get started. This will grow as I find time to convert my servers' config modification log/notes to something usable by more than just myself hehehe:

### Index
* General
  * [SR-IOV on UnRAID](https://forums.unraid.net/topic/103323-how-to-using-sr-iov-in-unraid-with-1gb10gb40gb-network-interface-cards-nics/)
  * [NFS - to be documented]
  * [Virtual Machines]
  * [Common issues and questions related to ZFS on UnRAID]
    * [Hosting the Docker Image on ZFS](https://github.com/teambvd/UnRAID-Performance-Compendium/blob/main/general/DockerImageOnZFS.md)
  * [Setting up various tools and scripts for monitoring and improved server mgmt quality of life]
  * [Installed tools and apps outside the ecosystem, and integrating them into UnRAID (cleanly)]
* Container Specific
  * [Ombi](https://github.com/teambvd/unraid-zfs-docs/blob/main/containers/ombi.md)
  * [Postgres](https://github.com/teambvd/unraid-zfs-docs/blob/main/containers/postgres.md)
  * [OpenLDAP]
    * [LAM - LDAP Account Manager]
    * [Authentik]
    * [PWM]
  * [Nextcloud - (in progress)](https://github.com/teambvd/unraid_docs-ZFS_and_Containers/blob/main/containers/nextcloud.md)
  * [MariaDB]
  * [ElasticSearch]
  * [InfluxDB/Telegraf/Prometheus/Grafana]
    * [ELK stack using Elastic from above]
  * [Home Assistant]
    * [Frigate]
      * [Doube take]
      * [Compreface]
      * [Deepstack]
    * [MQTT]
  * [Sonarr/Radarr/Lidarr - Anything with SQLite](https://github.com/teambvd/unraid-zfs-docs/blob/main/containers/sonarrRadarrLidarr.md)
* Assorted stuff (that doesn't fit anywhere else)
  * [SMB on UnRAID](https://forums.unraid.net/topic/97165-smb-performance-tuning/)
  * [Compiled commands reference](https://github.com/teambvd/UnRAID-Performance-Compendium/blob/main/general/helpfulCommands.md)
  * 
  
