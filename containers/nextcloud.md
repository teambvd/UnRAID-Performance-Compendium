# Nextcloud - Performance Optimized

This doc goes through the various performance-related tips and tweaks I've compiled for my own implementation over the course of the last several years. It's not intended as a one-stop-shop for 'how to install nextcloud', but instead more of a 'what to do once you've gotten your nextcloud up and running' to make it run more efficiently. 

## Table of Contents

  + [Assumes use of the following containers](#assumes-use-of-the-following-containers)
    - [Additional Notes](#additional-notes)
  + [Databases](#databases)
  + [Tuning the Nextcloud container](#tuning-the-nextcloud-container)
    - [Global PHP parameters](#global-php-parameters)
    - [Preview generation related](#preview-generation-related)
      + [Making sure pregeneration serves its purpose in UnRAID](#making-sure-pregeneration-serves-its-purpose-in-unraid)
  + [Generally helpful configuration options](#generally-helpful-configuration-options)
    - [SSL on LAN for secured local access](#ssl-on-lan-for-secured-local-access)
    - [The Deck app is dumb](#the-deck-app-is-dumb)
    - [Create previews for numerous additional filetypes](#create-previews-for-numerous-additional-filetypes)
    - [HTTP2 in NginxProxyManager](#http2-enablement)
    - [Some notes on Cloudflare](#some-notes-on-cloudflare)
       + [Bypassing site rules](#bypassing-site-rules)

### Assumes use of the following containers
* LSIO nextcloud with php8
* xternet onlyoffice image (manually updated from within container - I know, terrible, I'm lazy I guess :-\ )
* redis (`official alpine`)
* postgres 15 (`official alpine`) - However pg16 should be sufficiently stable at current if deploying from scratch
* and NginxProxyManager (`official`) as double proxy
(if already using rabbitmq, for something like homeassistant or w/e, you can use that same instance for onlyoffice here too. Extra credit or some crap, brown noser.)

#### Additional Notes

* All containers on same custom network, including NginxProxyManager (can have multiple networks attached if necessary - easiest using something like portainer for managing more complex docker network deployments)
* Container names are used for DNS within that custom network, and are named: nextcloud, redis, postgres, onlyoffice, nginx

### Databases

* Postgres user and db creation:
  ```sql
  CREATE USER nc WITH PASSWORD 'MakeThisOneGoodAndSTRONK';
  CREATE DATABASE nc TEMPLATE template0 ENCODING 'UNICODE';
  ALTER DATABASE nc OWNER TO nc;
  GRANT ALL PRIVILEGES ON DATABASE nc TO nc;
  ```
  * Create one for `onlyoffice` as well, so the docs container doesn't have to spin up some postgres 12 container or some such nonsense, same for `redis` (but no additional redis config necessary). This helps to reduce excess CPU consumption. See [here](https://github.com/ONLYOFFICE/Docker-DocumentServer#available-configuration-parameters) for variables to add to the onlyoffice container to allow use of existing postgres/redis containers.

* Performance recommendations related to DB's - Add the below to 'go' file - see [here](https://docs.actian.com/vector/4.2/index.html#page/User%2FOS_Settings.htm%23) for details/calculations - 
  ```bash
  ### (see Vector documentation) - re DB performance
  ## allow overcommit of mem for virtualization
  echo 1 > /proc/sys/vm/overcommit_memory
  ## required for Elasticsearch
  sysctl -w vm.max_map_count=393216
  ```

* [Redis](https://github.com/redis/redis) - needed for high performace backend as well as onlyoffice, and hugely helpful for systems with either high user counts, or where users may be syncing a massive number of files; this assumes you've created a custom docker network which both nextcloud and redis share, and that the redis container is simply named 'redis' - edit config.php to look like:
  ```php
  $CONFIG = array (
  'memcache.local' => '\\OC\\Memcache\\APCu',
  'memcache.distributed' => '\\OC\\Memcache\\Redis',
  'memcache.locking' => '\\OC\\Memcache\\Redis',
  'redis' =>
  array (
    'host' => 'redis',
    'port' => 6379,
  ), 
  ```

### Tuning the Nextcloud container

Nearly all our nextcloud 'application-specific' tuning options are PHP related, though some are to the 'global' PHP instance, while others are for directed to applications coded in PHP which use that PHP instance.

#### Global PHP parameters

You can find explanations of these tunable options (PHP calls them 'directives') and what their impacts are can be found in the [official PHP documentation](https://www.php.net/manual/en/ini.core.php):

* Set your nextcloud's limits - input and execution time must be increased as well (as this is the max time it'll allow an action to take, and 60G can't be downloaded in 60 seconds, set to 2 hours here) "nano /mnt/wd/dock/nextcloud/php/php-local.ini"
  ```php
  memory_limit = 8192M
  upload_max_filesize = 60G
  post_max_size = 60G
  max_input_time = 7200
  max_execution_time = 7200
  ```
  * For the execution and input times here, you should try to do some rough math to deduce 'with my max file size, and my available upload bandwidth, how long would it realistically take to complete the upload?' Taking the above as an example - for a `60GB` file to upload using a 30Mb/s pipe takes roughly 4 hours and 20 mins. Building in some buffer room for other small sync operations in between and rounding up, 5 hours is chosen (our `7200` above, converting to time in seconds). You can use a [bandwidth calculator](https://www.calculator.net/bandwidth-calculator.html) to make things easier on you.

* Increase the number of PHP processes allowed, adding the below to to the bottom of - /mnt/wd/dock/nextcloud/php/www2.conf
  ```php
  pm = dynamic
  pm.max_children = 120
  pm.start_servers = 12
  pm.min_spare_servers = 6
  pm.max_spare_servers = 24
  ```
  * Be aware that these settings are reliant upon your [DB's configuration](https://github.com/teambvd/UnRAID-Performance-Compendium/blob/main/containers/postgres.md) for proper function; for instance, if you've set `pm.max_children` to `100`, but your database's max connections allowed is 50, you're going to see php reporting errors spinning up another server/child process if it tries to request a connection beyond the DB's allowed settings.

* install pdlib at startup for facial recognition plugin - "nano /mnt/wd/dock/nextcloud/custom-cont-init.d/pdlib"
  ```bash
  #!/bin/bash
  echo "**** installing pdlib ***"
  apk add -X http://dl-cdn.alpinelinux.org/alpine/edge/testing php8-pdlib
  apk add -X http://dl-cdn.alpinelinux.org/alpine/edge/testing dlib
  ```

#### Preview generation related

Highly customizable; the more options you add here, the more preview types you'll generate, though this is at the cost of more storage for preview. One should note, whether and to what degree this helps page load performance will be highly dependent upon the storage which contains these preview files, and as such, recommend they're kept on low latency / high performance data storage. 

* Below pre-gen's thumbnails, but not the 'clicked' images (when you open the image, this preview is generated on demand), and limits the on demand generated size to 2048x2048
    * From the cli, configure the pre-generator
      ```bash
      s6-setuidgid abc php8 -f /config/www/nextcloud/occ config:app:set --value="32 64 1024"  previewgenerator squareSizes
      s6-setuidgid abc php8 -f /config/www/nextcloud/occ config:app:set --value="64 128 1024" previewgenerator widthSizes
      s6-setuidgid abc php8 -f /config/www/nextcloud/occ config:app:set --value="64 256 1024" previewgenerator heightSizes
      ```
	* Then limit on demand generation size, adding below to the bottom - "nano /mnt/wd/dock/nextcloud/config/config.php"
	  ```php
	  'preview_max_x' => '2048',
	  'preview_max_y' => '2048',
	  'jpeg_quality' => '60',
	  ```
		* Or alternatively, from the cli
		  ```python
		  s6-setuidgid abc php8 -f /config/www/nextcloud/occ config:app:set preview preview_max_y --value="2048"
		  s6-setuidgid abc php8 -f /config/www/nextcloud/occ config:app:set preview preview_max_x --value="2048"
		  s6-setuidgid abc php8 -f /config/www/nextcloud/occ config:app:set preview jpeg_quality --value="60"
		  ```
	* Set up automated preview generation and facial recognition hourly for face, 3 times/hour for previews - "nano /mnt/wd/dock/nextcloud/crontabs/root"
      ```bash
      */20	*	*	*	*	s6-setuidgid abc php8 -f /config/www/nextcloud/occ preview:pre-generate -vvv
      */20	*	*	*	*	s6-setuidgid abc php8 -f /config/www/nextcloud/occ documentserver:flush
      0	*	*	*	*	s6-setuidgid abc php8 -f /config/www/nextcloud/occ face:background_job -t 35
      ```

<div class="warning" style='padding:1em; background-color:#3396ff; color:#0033cc'>
<span>
<p style='margin-top:0.5em; text-align:left'>
<b>NOTE</b></p>
<p style='margin-left:0.5em;'>
Using 's6-setuidgid' is typically preferred as this pretty well 'always works', as it sets the config as the specific (occ) user would from the command line, 'impersonating' them, but transparently doing so. Modifying the config files directly may not always work, depending on the install, as one config file may be overridden by another during init. Just depends on the specific container. 
<p style='margin-bottom:1em; margin-right:0.5em; text-align:right; font-family:Georgia'>
</p></span>
</div>

*Refer to the [s6 Overlay Github](https://github.com/just-containers/s6-overlay) for further information on how s6 works*

##### **Making sure pregeneration serves its purpose in UnRAID**

Preview pre-generation's entire goal is to ensure that, when you click to load a page, there's no waiting around for that pages contents to load other than that which is caused by the upload bandwidth of your server. If, like me, you've a boatload of files in your nextcloud, you may well be using the UnRAID array to host your files (if so, recommend the Integrity plugin from community apps), likely with the share you've mapped to nextcloud's "/data" folder set to `cache=yes`.

The problem here is that nextcloud's appdata (where it puts all data related to items installed from the nextcloud Hub) is stored in this same share - with `cache=yes`, all your previews will inevitably end up on the (slow) array once mover runs. We want **at least** the pre-generated images on the cache; we don't really care about the longevity of these files, so even if you've only a single disk cache pool, we can keep them there. Should the worst occur and your cache disk dies, we can always re-generate the previews.

* First ensure the [CA mover tuning](https://forums.unraid.net/topic/70783-plugin-mover-tuning/) plugin is installed. This can then be configured at `https://UnraidIpAddress/Plugins/Scheduler`, then the `Mover Tuning` tab.
* Now we need to create a file noting what to ignore - on your flash boot device is fine, I've chosen to put it on my zpool with the rest of my scripts and such:
  ```bash
  nano /mnt/user/scriptLocation/moverTuningExclusions.txt
  
  # now we're telling Mover Tuning what to leave alone - put just this line into the file and save/exit:
  /mnt/user/nxstore/appdata_ocbgah4z0nhr/preview
  ```
* We'll now set up mover tuning to ignore moving anything within our preview dir using `Ignore files listed inside of a text file` and specifying the file we just created:
  ![MoverTuning](https://github.com/teambvd/UnRAID-Performance-Compendium/blob/main/containers/img/moverTuning.png)

**IMPORTANT**
You can set mover tuning to leave the entire appdata directory on the cache, but should **ONLY** do this if you've a redundant cache pool (mirrored, raid5/6) as not all data within the appdata dir is 'expendable' in the same way the pre-generated previews are. 

### Generally helpful configuration options

There are some 'quality of life' customizations I've found over my time running owncloud, then eventually nextcloud after the split, which have been significantly useful:

#### SSL on LAN for secured local access

* Set up user script to ensure that SSL certs are copied from NginxProxyManager to nextcloud, I've mine kicking off daily at 0305 early each morning (`custom sched, '5 3 * * *'`)
  ```bash
  ## copying key and cert for nextcloud
  cp /mnt/wd/compose/nginx/letsencrypt/live/npm-30/privkey.pem /mnt/wd/compose/nextcloud/keys/privkey.pem
  cp /mnt/wd/compose/nginx/letsencrypt/live/npm-30/cert.pem /mnt/wd/compose/nextcloud/keys/cert.pem
  ```

#### The Deck app is dumb

* Make your login actually go to your files instead of the dashboard - for home users utilizing nextcloud mainly for it's initial design purpose (cloud storage / file sync and share), the dashboard 'Deck' app is... Sorta worthless. You can instead make nextcloud default to showing your files (or whatever you prefer) immediately after logging in to save you that extra click each time! Add the following to config.php - this can be set to any app (e.g. 'photos', 'circles', 'calendar', etc):

  ```php
  'defaultapp' => 'files',
  ```

#### Create previews for numerous additional filetypes

This can be resource intensive at first when setting up, but once it's churned through all your existing files, the additional load should be minimal (if even noticeable):

* Allow previews to be generated for files other than pictures -
	* List of potential preview candidates are as follows: 
      ```php
      OC\Preview\BMP
      OC\Preview\GIF
      OC\Preview\JPEG
      OC\Preview\MarkDown
      OC\Preview\MP3
      OC\Preview\PNG
      OC\Preview\TXT
      OC\Preview\XBitmap
      OC\Preview\OpenDocument
      OC\Preview\Krita
      OC\Preview\Illustrator
      OC\Preview\HEIC
      OC\Preview\Movie
      OC\Preview\MSOffice2003
      OC\Preview\MSOffice2007
      OC\Preview\MSOfficeDoc
      OC\Preview\PDF
      OC\Preview\Photoshop
      OC\Preview\Postscript
      OC\Preview\StarOffice
      OC\Preview\SVG
      OC\Preview\TIFF
      OC\Preview\Font
      ```
    * Defaults only to a select few (images mostly, but notably excluding HEIC, which is super annoying if you've any iOS users)... But you can modify the included providers. Note, if you specify ANY providers, then none of the defaults are selected, meaning f you're going to add one, then you need to specify EACH of the ones you want previews for (e.g. setting just the HEIC provider means that nothing other than HEIC will get previews, so ensure you've listed each one you need).
      
      However, if you specify any providers, only those explicitly specified will be used (so you can't add just one then expect the defaults to be called as well); example (add to config.php):
      ```php
      'enable_previews' => true,
      'enabledPreviewProviders' =>
       array (
         1 => 'OC\\Preview\\HEIC',
         2 => 'OC\\Preview\\PDF',
         3 => 'OC\\Preview\\XBitmap',
         4 => 'OC\\Preview\\PNG',
         5 => 'OC\\Preview\\Image',
         6 => 'OC\\Preview\\Photoshop',
         7 => 'OC\\Preview\\TIFF',
         8 => 'OC\\Preview\\SVG',
         9 => 'OC\\Preview\\BMP',
         10 => 'OC\\Preview\\GIF',
         11 => 'OC\\Preview\\JPEG',
      ),
      ```
    * Lastly, finally enable your configured stuff from above (docker terminal for nextcloud):
      ```php
      s6-setuidgid abc php8 -f /config/www/nextcloud/occ preview:generate-all -vvv
      ```

#### HTTP2 Enablement

This is enabled by default in NginxProxyManager, but worth it to confirm the HTTP/2 switch is enabled, just in case - external connectivity on modern browsers will likely throw a warning if this isn't set to enabled regardless though, so you've probably already got this covered.

#### Some notes on Cloudflare

While I won't go in to the full setup/config here (tons of great guides on that available elsewhere), and it's definitely a fantastic tool for both protecting yourself, as well as speeding up (most) websites, there are various features and functionality CF enables / makes available which can cause some severe issues with Nextcloud. Some examples include the Auto-Minify functions for CSS and Javascript, caching content (counterintuitive, I know!), and some compression types (which don't always behave properly with PHP).

We don't want CF caching anything from Nextcloud for a couple reasons:
* We already generate lower resolution previews ourselves (or should at least - see above). Having CF cache already tiny files has limited (if any) benefit, with the possible down side of rendering out-of-date content (which can result in weird rendering issues)
* From a security perspective, it's usually not a great idea to have someone else hosting your images, which is effectively what's happening here, except in this case, you're not able to encrypt them from view as you would with a backup. Using CF caching for your private and possibly sensitive files increases your datas surface area.

##### Bypassing site rules

In order to keep the benefits of various CF tools / components (brotli compression, caching for static asset sites, IP protection, etc) while still hosting Nextcloud under the same domain, we can set up a page rule in order to have the site we're hosting Nextcloud under bypass those potentially problematic settings.

Under "Rules" -> "Page Rules", we'll create a new rule specifying our Nextcloud location (e.g. nextcloud.mydomain.com), and choosing to both bypass caching as well as "Disable Performance" (more like "Disable Suffering"):
![SiteRules](https://github.com/teambvd/UnRAID-Performance-Compendium/blob/main/containers/img/site-rules.png?raw=true)

*******

*******

*******


***** EXTRA / BONUS *****

* Rename and set all your photos to the datetime that you actually took them (makes them show up in the order you took them in web interface) - requires installation of 'exiftool' (see Nerd Pack):

	* Create a script dir and script (as the 'nobody' user to avoid perms issues):

		sudo -u nobody mkdir /mnt/user/appdata/nextcloud/scripts
		sudo -u nobody nano /mnt/user/appdata/nextcloud/scripts/photo-name-to-create-date.sh

	* Paste in the following:

		#!/bin/sh
		
		albumdir=$1
		
		# comment out the next two lines if running outside of nextcloud
		nextclouddir="/config/var/www/nextcloud"
		nextclouduser=""
		
		echo "Changing modified date to date photo was taken..."
		exiftool "-filemodifydate<datetimeoriginal" -r "$albumdir"
		
		echo "Renaming files to shot date..."
		exiftool '-FileName<DateTimeOriginal' -r -d "%Y-%m-%d_%H.%M.%S%%-c.%%e" "$albumdir"
		
		# comment out the next two lines if running outside of nextcloud
		echo "Re-scanning your Nextcloud data directory..."
		s6-setuidgid abc php8 -f "$nextclouddir"/occ files:scan "$nextclouduser"
		
		exit 0

	* Test run your script:

		s6-setuidgid abc php8 -f /config/scripts/photo-name-to-create-date.sh /data/USERNAME/files/Photos




<NOT DONE, further documentation needed!!!>


* config.php

'tempdirectory' => '/tmp/nextcloudtemp',
'localstorage.allowsymlinks' => false,
'filesystem_check_changes' => 0,
'simpleSignUpLink.shown' => false,

* Nginx Proxy Manager - Advanced / nextcloud -

location ^~ /push/ {
  proxy_pass http://<SERVER_IP>:7867/;
  proxy_http_version 1.1;
  proxy_set_header Upgrade $http_upgrade;
  proxy_set_header Connection "Upgrade";
  proxy_set_header Host $host;
  proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
  }

 proxy_max_temp_file_size 16384m;
 client_max_body_size 0;




* Allow for implementation of high performance backend for nextcloud - duplicate your container

# Rule added in order to allow implementation of HPB
location = /push/ {
    proxy_pass http://<SERVER-IP>:7867/;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "Upgrade";
    proxy_set_header Host $host;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

