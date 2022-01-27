# How to run FoundryVTT on a Raspberry Pi 4 using nginx and foundryvtt-docker

## Prerequisites:
* External SSD with USB 3.0 cable (I used this [120 GB SSD](https://www.amazon.com/dp/B073TTR8JY/ref=dp_iou_view_product?ie=UTF8&psc=1) and [this case](https://www.amazon.com/gp/product/B00OJ3UJ2S/ref=ppx_od_dt_b_asin_title_s00?ie=UTF8&psc=1))
* Raspberry Pi 4

## Format SSD so you can boot from it
1. Install Pi OS on the external SSD. I downloaded [Raspberry Pi Imager](https://www.raspberrypi.org/software/) on my adjacent Mac and used that to install the latest Raspbery Pi OS.
<img width="674" alt="Screen Shot 2020-11-17 at 10 05 03 PM" src="https://user-images.githubusercontent.com/33645693/99486155-f588af00-2920-11eb-805d-facb578fdabe.png">

2. Once the install finishes, unplug the SSD, remove the SD card from your raspberry pi (if it has one), plug in the SSD, and boot it up from the SSD.

	Note: if you have trouble getting your Raspberry Pi 4 to boot from one of the blue USB 3.0 ports, boot it from a black USB 2.0 port, then run `sudo nano /boot/cmdline.txt` and add the following to the beginning of the line in /boot/cmdline.txt: `usb-storage.quirks=152d:0578:u  `

## Setup SSH
Follow the instructions [here](https://www.raspberrypi.com/documentation/computers/remote-access.html) to setup ssh, including passwordless ssh access.

## Setup AFP (if on a mac)
Follow [this guide](https://pimylifeup.com/raspberry-pi-afp/) to setup an AFP server so you can access your Pi's files remotely.

## Setup PiTunnel
[PiTunnel](https://pitunnel.com/) makes it easy to ssh into your raspberry pi when you're outside of your local network.

## Install Docker
1. Follow the steps [here](https://docs.docker.com/engine/install/debian/#install-using-the-convenience-script) to install Docker using the convenience script.
2. Run `systemctl start docker`
3.  Install docker-compose using `sudo apt install docker-compose`. 
4.  Add a Non-Root User to the Docker Group
	By default, only users who have administrative privileges (root users) can run containers. If you are not logged in as the root, one option is to use the sudo prefix. However, you could also add your non-root user to the Docker group which will allow it to execute docker commands. The syntax for adding users to the Docker group is:
	
	```
	sudo usermod -aG docker [user_name]
	```

	To add the Pi user (the default user in Raspbian), use the command:

	```
	sudo usermod -aG docker pi
	```

	You can check the version of Docker using `docker version`. For system-wide information (including the kernel version, number of containers and images, and more extended description) run `docker info`.
  
## Setup Steps
Basically just follow the steps as indicated on the FoundryVTT [installation](https://foundryvtt.com/article/installation/) instructions for Node.js, [Nginx](https://foundryvtt.com/article/nginx/), and [foundryvtt-docker](https://github.com/felddy/foundryvtt-docker). You can get 5 free subdomains from [DuckDNS](http://www.duckdns.org/).

1. Go to www.duckdns.org, sign in, and create a subdomain of your choice. 

![Screen Shot 2020-11-17 at 7 41 23 PM](https://user-images.githubusercontent.com/33645693/99475967-e13ab700-290c-11eb-8cd6-caa0bd11e64a.png)

Then go [here](https://www.duckdns.org/install.jsp?tab=pi) and follow the installation instructions.

2. Install Node.js
	```
	sudo apt install -y libssl-dev
	curl -sL https://deb.nodesource.com/setup_12.x | sudo bash -
	sudo apt install -y nodejs
	```
3. Create a directory for the application data
	```
	cd $HOME
	mkdir foundrydata
	```
4. Install nginx: 
	```
	sudo apt-get update
	sudo apt-get install nginx
	```
5. Create an Nginx configuration file for your domain. Make sure to update the references to `your.hostname.com` in the configuration file: `sudo nano /etc/nginx/sites-available/your.hostname.com` and make sure the proxy_pass port number matches the published port in your docker-compose file.
	```
	# the filename should be "your.hostname.com"

	# Define Server
	server {

	    # Enter your fully qualified domain name or leave blank
	    server_name             your.hostname.com;

	    # Listen on port 80 without SSL certificates
	    listen                  80;

	    # Sets the Max Upload size to 300 MB
	    client_max_body_size 300M;

	    # Proxy Requests to Foundry VTT
	    location / {

		# Set proxy headers
		proxy_set_header Host $host;
		proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
		proxy_set_header X-Forwarded-Proto $scheme;

		# These are important to support WebSockets
		proxy_set_header Upgrade $http_upgrade;
		proxy_set_header Connection "Upgrade";

		# Make sure to set your Foundry VTT port number
		# This should match the published port in your docker-compose file
		proxy_pass http://localhost:30000;
	    }
	}
	```
Save and exit nano with ctrl + x, press Y, then Enter

6. Use the `service` utility to manage your Nginx server. Enable the site, test your configuration, fix any errors, and start the service. Don't forget to replace `your.hostname.com`!
	```
	# Enable new site
	sudo ln -s /etc/nginx/sites-available/your.hostname.com /etc/nginx/sites-enabled/

	# Test your configuration file
	sudo service nginx configtest

	# View configuration errors (if the configtest was not OK)
	sudo nginx -t

	# Start Nginx
	sudo service nginx start

	# Stop Nginx
	sudo service nginx stop

	# Restart Nginx
	sudo service nginx restart
	```
7. Next, create a docker-compose file to run foundryvtt-docker: `nano $HOME/docker-compose.yml`. See the [foundryvtt-docker](https://github.com/felddy/foundryvtt-docker) for instructions on using secrets, a temporary foundryvtt download url, and other options. Make sure to replace `YOUR_FOUNDRY_LICENSE`, `YOUR_ADMIN_KEY`, `YOUR_FOUNDRY_USERNAME`, and `YOUR_FOUNDRY_PASSWORD` before saving the file.
	```
	version: "3.3"

	services:
	  foundry_1:
	    container_name: foundry
	    image: felddy/foundryvtt:release
	    hostname: my_foundry_host_1
	    restart: "unless-stopped"
	    volumes:
	      - type: bind
		source: ./foundrydata
		target: /data
	    environment:
	      - CONTAINER_CACHE=/data/container_cache
	      - FOUNDRY_USERNAME=YOUR_FOUNDRY_USERNAME
	      - FOUNDRY_PASSWORD=YOUR_FOUNDRY_PASSWORD
	      - FOUNDRY_LICENSE_KEY=YOUR_FOUNDRY_LICENSE
	      - FOUNDRY_ADMIN_KEY=YOUR_ADMIN_KEY
	    ports:
	      - target: "30000"
		published: "30000"
		protocol: tcp
	```
*Note: if you need to run FoundryVTT on a different port, change the published port to the desired port. No need to change the target port. The `target` is the port inside the container, the `published` is the publicly exposed port.*

8. Now you should be able to start the container and see FoundryVTT running at `http://localhost:30000` and at `http://your.hostname.com`:
	```
	docker-compose up -d
	```
Check that your container is running using `docker container ls`, view the container logs using `docker logs foundry`. If needed you can stop the container `docker stop foundry`, remove it `docker rm foundry`, and run it again after making any necessary changes to your docker compose file `docker-compose up -d`. Alternately, `docker-compose down` will stop all containers and removes containers, networks, volumes, and images created by the previous `docker-compose up` in a single command.

9. Now it's time to setup HTTPS for your domain. Create SSL certificates using Certbot. Follow the instructions [here](https://certbot.eff.org/instructions?ws=nginx&os=debianbuster).
10. Update the nginx config file to use port 443 and the SSL certificates you created. Again, make sure to replace `your.hostname.com`: `sudo nano /etc/nginx/sites-available/your.hostname.com` and make sure the proxy_pass port number matches the published port in your docker-compose file.
	```
	# the filename should be "your.hostname.com"

	# Define Server
	server {

	    # Enter your fully qualified domain name or leave blank
	    server_name             your.hostname.com;

	    # Listen on port 443 using SSL certificates
	    listen                  443 ssl;
	    ssl_certificate         "/etc/letsencrypt/live/your.hostname.com/fullchain.pem";
	    ssl_certificate_key     "/etc/letsencrypt/live/your.hostname.com/privkey.pem";

	    # Sets the Max Upload size to 300 MB
	    client_max_body_size 300M;

	    # Proxy Requests to Foundry VTT
	    location / {

		# Set proxy headers
		proxy_set_header Host $host;
		proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
		proxy_set_header X-Forwarded-Proto $scheme;

		# These are important to support WebSockets
		proxy_set_header Upgrade $http_upgrade;
		proxy_set_header Connection "Upgrade";

		# Make sure to set your Foundry VTT port number
		# This should match the published port in your docker-compose file
		proxy_pass http://localhost:30000;
	    }
	}

	# Optional, but recommend. Redirects all HTTP requests to HTTPS for you
	server {
	    if ($host = your.hostname.com) {
		return 301 https://$host$request_uri;
	    }

	    listen 80;
		listen [::]:80;

	    server_name your.hostname.com;
	    return 404;
	}
	```
Save and exit nano with ctrl + x, press Y, then Enter

11. Don't forget to setup Port Forwarding on your router. You'll need to forward ports 80 and 443 to your raspberry pi. Every router is different, but you can find specific instructions [here](https://portforward.com/).

Now your site should be accessible at `https://your.hostname.com`!

## Enable Audio/Video Chat with Jitsi
1. Log on to the admin page of your foundryvtt instance, go to Add-on Modules and install JitsiWebRTC:
![Screen Shot 2020-11-25 at 7 30 21 PM](https://user-images.githubusercontent.com/33645693/100301155-cbed0a80-2f54-11eb-810b-8fd8033401b7.png)

2. Launch your world, join the game session as GM, go to Game Settings -> Manage Modules, enable Jitsi Web RTC client and then Save Module Settings

	![](https://i.imgur.com/Drb3eGc.png)

3. Once your world reloads, go to Game Settings -> Configure Settings -> click on Configure Audio/Video -> change Audio/Video Conferencing Mode to Audio/Video Enabled

	![Screen Shot 2020-11-25 at 7 32 54 PM](https://user-images.githubusercontent.com/33645693/100301376-43229e80-2f55-11eb-9e02-4920849df002.png)

	![Screen Shot 2020-11-25 at 7 34 28 PM](https://user-images.githubusercontent.com/33645693/100301382-44ec6200-2f55-11eb-8be4-b5e70858d652.png)

*Note: you don't need to setup your own Jitsi server*

4. Save changes and once the page reloads, you should get a popup in your browser to enable your site to use your microphone and camera. That's it!

## Sync your pi with Google Drive for automatic backups
Be aware that you shouldn't have your foundry instance referencing data in a cloud sync/backup service as it could cause data corruption. What you can do is create a tar archive of your foundry data, compress it with gzip and move it to your cloud sync/backup folder on a regular basis.
1. Follow [this guide](https://medium.com/@artur.klauser/mounting-google-drive-on-raspberry-pi-f5002c7095c2) to mount google drive using `rclone`.

*Note: install the latest version of rclone using `curl https://rclone.org/install.sh | sudo bash`*
	
a. I recommend modifying your rclone@.service file so it auto-restarts: `nano ~/.config/systemd/user/rclone@.service`. Add the `Restart` and `RestartSec` options to the 	   [Service] and `StartLimitIntervalSec` and `StartLimitBurst` in the [Unit] section prevent a failing service from being restarted every 5 seconds. This will give it 5 attempts, if it still fails, systemd will stop trying to start the service.

```
[Unit]
Description=rclone: Remote FUSE filesystem for cloud storage config %i
Documentation=man:rclone(1)
StartLimitIntervalSec=500
StartLimitBurst=5
[Service]
Type=notify
Restart=on-failure
RestartSec=5s
ExecStartPre=/bin/mkdir -p %h/mnt/%i
ExecStart= \
  /usr/bin/rclone mount \
    --fast-list \
    --vfs-cache-mode writes \
    --vfs-cache-max-size 100M \
    %i: %h/mnt/%i
[Install]
WantedBy=default.target
```

2. Follow [this guide](https://raspberrytips.com/backup-raspberry-pi/) to schedule automatic backups of your foundrydata folder (Config, Data, and Logs folders).

a. I recommend creating a local /bin folder to store the backup script and add it to your PATH, instead of the /usr/bin folder which requires sudo access:
```
mkdir ~/bin
export PATH=home/pi/bin:$PATH 
```

b. This is what my backup script looks like. I used `rclone move` to copy files to google drive and `rclone delete` to automatically delete older files.
```
#!/bin/bash
BACKUP_FOLDER='/home/pi/backups/'
BACKUP_CMD='/bin/tar -rvf'

counter=1
while [ $counter -le 4 ]
do
	SECONDS=0
	NOW=`date '+%F_%H%M'`	
	DEST_FILE="foundrydata$counter-backup-$NOW.tar"
	DEST_FOLDER="/home/pi/mnt/gdrive/foundrydata$counter/"
	rclone delete --min-age 56d $DEST_FOLDER 
	$BACKUP_CMD $BACKUP_FOLDER/$DEST_FILE -P /home/pi/foundrydata$counter
	echo compressing..
	/bin/gzip $BACKUP_FOLDER/$DEST_FILE
	echo copying..
	rclone move --update --verbose --transfers 30 --checkers 8 --contimeout 60s --timeout 300s --retries 3 --drive-chunk-size 128M --low-level-retries 10 --fast-list --stats 1s $BACKUP_FOLDER/$DEST_FILE.gz gdrive:foundrydata$counter	
	echo foundrydata$counter backup completed in $SECONDS seconds
	((counter++))
done
echo all backups complete
```

c. You can use [Crontab Generator](https://crontab-generator.org/) to find the syntax for whatever backup schedule you want in crontab. I used this to run backups each Wednesday at 2 am and output the log to a log file with the date (and the backup script above deletes backups after 8 weeks):

    0 2 * * 3 ~/bin/backup.sh > ~/backups/backup__`date +\%y\%m\%d`.log

## Run multiple FoundryVTT instances
If you have multiple FoundryVTT licenses, it's easy to modify the docker-compose file to spin up multiple foundry instances and have each instance be accessible from its own URL. 
You'll need to:
1. Create a directory for the application data.
```
cd $HOME
mkdir foundrydata2
```
2. Create another subdomain on www.duckdns.org
3. From Setup Step 5 above, follow the same process again but for your new URL. Create the Nginx config file.
4. Add the additional service to your existing docker-compose file (rather than creating another docker-compose file) and start it up to ensure it’s setup correctly. Here I have it setup for 3 foundry instances:
```
---
version: "3.3"

services:
  foundry_1:
    container_name: foundry1
    image: felddy/foundryvtt:release
    hostname: my_foundry_host_1
    restart: "unless-stopped"
    volumes:
      - type: bind
        source: ./foundrydata1
        target: /data
    environment:
      - FOUNDRY_USERNAME=YOUR_USERNAME
      - FOUNDRY_PASSWORD=YOUR_PASSWORD
      - CONTAINER_CACHE=/data/container_cache
      - FOUNDRY_LICENSE_KEY=YOUR_LICENSE_KEY
      - VIRTUAL_HOST=KEY=YOUR_URL.duckdns.org
      - FOUNDRY_ADMIN_KEY=YOUR_ADMIN_KEY
    ports:
      - target: "30000"
        published: "30000"
        protocol: tcp
  foundry_2:
    container_name: foundry2
    image: felddy/foundryvtt:release
    hostname: my_foundry_host_2
    restart: "unless-stopped"
    volumes:
      - type: bind
        source: ./foundrydata2
        target: /data
      - '/home/pi/foundrydata1/Data/maps/:/home/pi/foundrydata1/Data/maps/'
      - '/home/pi/foundrydata1/Data/modules/:/home/pi/foundrydata1/Data/modules/'
      - '/home/pi/foundrydata1/Data/systems/:/home/pi/foundrydata1/Data/systems/'
      - '/home/pi/foundrydata1/Data/audio/:/home/pi/foundrydata1/Data/audio/'
      - '/home/pi/foundrydata1/Data/tokens/:/home/pi/foundrydata1/Data/tokens/'
      - '/home/pi/foundrydata1/Data/dndbeyond/:/home/pi/foundrydata1/Data/dndbeyond/'
      - '/home/pi/foundrydata1/Data/dragupload/:/home/pi/foundrydata1/Data/dragupload/'
      - '/home/pi/foundrydata1/Data/uploaded-chat-images/:/home/pi/foundrydata1/Data/uploaded-chat-images/'
    environment:
      - FOUNDRY_USERNAME=YOUR_USERNAME
      - FOUNDRY_PASSWORD=YOUR_PASSWORD
      - CONTAINER_CACHE=/data/container_cache
      - FOUNDRY_LICENSE_KEY=YOUR_LICENSE_KEY
      - VIRTUAL_HOST=KEY=YOUR_URL.duckdns.org
      - FOUNDRY_ADMIN_KEY=YOUR_ADMIN_KEY
    ports:
      - target: "30000"
        published: "30001"
        protocol: tcp
  foundry_3:
    container_name: foundry3
    image: felddy/foundryvtt:release
    hostname: my_foundry_host_2
    restart: "unless-stopped"
    volumes:
      - type: bind
        source: ./foundrydata3
        target: /data
      - '/home/pi/foundrydata1/Data/maps/:/home/pi/foundrydata1/Data/maps/'
      - '/home/pi/foundrydata1/Data/modules/:/home/pi/foundrydata1/Data/modules/'
      - '/home/pi/foundrydata1/Data/systems/:/home/pi/foundrydata1/Data/systems/'
      - '/home/pi/foundrydata1/Data/audio/:/home/pi/foundrydata1/Data/audio/'
      - '/home/pi/foundrydata1/Data/tokens/:/home/pi/foundrydata1/Data/tokens/'
      - '/home/pi/foundrydata1/Data/dndbeyond/:/home/pi/foundrydata1/Data/dndbeyond/'
      - '/home/pi/foundrydata1/Data/dragupload/:/home/pi/foundrydata1/Data/dragupload/'
      - '/home/pi/foundrydata1/Data/uploaded-chat-images/:/home/pi/foundrydata1/Data/uploaded-chat-images/'
    environment:
      - FOUNDRY_USERNAME=YOUR_USERNAME
      - FOUNDRY_PASSWORD=YOUR_PASSWORD
      - CONTAINER_CACHE=/data/container_cache
      - FOUNDRY_LICENSE_KEY=YOUR_LICENSE_KEY
      - VIRTUAL_HOST=KEY=YOUR_URL.duckdns.org
      - FOUNDRY_ADMIN_KEY=YOUR_ADMIN_KEY
    ports:
      - target: "30000"
        published: "30002"
        protocol: tcp
```
Some things to note:
- Make sure to replace `YOUR_USERNAME`, `YOUR_PASSWORD`, `YOUR_LICENSE_KEY`, `YOUR_URL.duckdns.org`, and `YOUR_ADMIN_KEY` for each service.
- Each new Foundry instance will need its own published port number (but the target port will always be 30000)
- Foundry supports symbolic links (aka symlinks) as of [v0.5.0](https://foundryvtt.com/releases/0.5.0): "The software can now support symbolic links inside the user data location which can point to art assets, module, systems, or worlds which are stored in other locations." I've added symlinks so that all foundry instances share a bunch of folders, including modules and systems, with the foundrydata1 folder. That way I don't have to backup, maintain and update duplicate modules, worlds, and other assets. In addition to including each volume in the volumes section of the docker-compose file, you'll need to run the command `ln -s /home/pi/foundrydata1/Data/modules /home/pi/foundrydata2/Data/modules` for each folder you want to link. This command creates a link to `/home/pi/foundrydata1/Data/modules` in `/home/pi/foundrydata2/Data/`. For example, if you wanted to have the modules and systems shared between all instances, you'd need to run these commands:
	```
	ln -s /home/pi/foundrydata1/Data/modules /home/pi/foundrydata2/Data/modules
	ln -s /home/pi/foundrydata1/Data/modules /home/pi/foundrydata3/Data/modules
	ln -s /home/pi/foundrydata1/Data/systems /home/pi/foundrydata2/Data/systems
	ln -s /home/pi/foundrydata1/Data/systems /home/pi/foundrydata3/Data/systems
	```
4. Setup HTTPS using Certbot and Nginx per Setup Steps 9 and 10 above.


