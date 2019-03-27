# RichiHomeServer
This is my collection of Docker Compose files I use to run my simple home server. It runs off of traefik, Nextcloud, and Jellyfin at the moment, but I (or you,) can add simply enough by just adding to the services docker-compose file.

## Description
I'll go through some quick details about why I'm running what I am. Traefik is used to handle the reverse-proxy side of things, which means you can connect to different services by using different sub-domains (nc.domain.com will go to a diffent service than media.domain.com). Traefik is easy to install with docker-compose and, so far, has kept up with what I've installed in docker. I'm still new to Docker, so if I'm being a bit inefficient, please let me know.

I'm running [Nextcloud](https://nextcloud.com/) as our file hosting service (using MariaDB for our database, though if it gives you trouble I'll recommend SQLite for it's ease). Nextcloud can be used as a desktop sync client, file storage, and more if you're willing to install apps.  

I also recommend [Jellyfin](https://jellyfin.github.io/) for a media hosting and viewing solution, as it's FOSS (unlike emby and Plex) and has handled all media I've thrown at it so far.  

This is all run on Ubuntu Server 19.04, so I can't gaurantee it'll work on different version. You must also have docker and docker-compose installed, which will vary across Distro's (though I used Snap).

#### 1: Clone the repository, and other commands
First, type this command in your terminal: 
`git clone https://github.com/Richiachu/RichiHomeServer.git`
This will copy the repository to the folder you're in (likely your home/username folder).
We also need to run some necessary docker commands. We need to create a network that traefik will use to talk and see other docker images. In this case, it's called web. The command for this is:
 `docker network create web`

#### 2.1: Edit the necessary files - Traefik
To begin we're going to edit our docker-compose files. I'll explain as we go along. First, we'll edit traefik.  
Once we're in the right folder (RichiHomeServer/traefik), we'll edit our docker-compose file. I will be using nano, but you can use whatever you want (obligatory emacs vs vim comment aside).
`nano docker-compose.yml`
Let's analyze this file.
    version: '3'
    
    services:
      traefik:
        image: traefik
        restart: always
        ports:
          - "80:80"
          - "443:443"
        networks:
          - web
        volumes:
          - /var/run/docker.sock:/var/run/docker.sock
          - /home/username/RichiHomeServer/traefik/traefik.toml:/traefik.toml
          - /home/username/RichiHomeServer/traefik/acme.json:/acme.json
        container_name: traefik
        labels:
          - "traefik.enable=true"
          - "traefik.docker.network=web"
          - "traefik.frontend.rule=Host:traefik.domain.com"
          - "traefik.port=8080"
          - "traefik.frontend.auth.basic=Username:$$2y$$05$$ztcHlOeJs.4JbPziBrG5MuP1lf3ddNP/qm06ByT9Fd4/xzldxYw/2"

    networks:
      web:
        external: true

This is our Traefik docker compose file. We'll be using this to create our reverse-proxy that will connect us to our services.

We'll go through this section by section. The # is used to represent my comments, and shouldn't be included in your files (duh).

    services:
      traefik:
        image: traefik #Names the docker container and the image it will be building, in this case, traefik.
        restart: always
        ports: # You will need to port forward these ports on your router (but only these). Port 80 and 443 are used by your web browser to access web pages.
          - "80:80"
          - "443:443"
        networks:
          - web

You're probably not gonna need to change any of these commands. The only thing you'll need to do is port forward ports 80 and 443, but that's not done in the compose file. Next, we have:

        volumes:   # Volumes are used to create link data from your actual PC to the docker container, in this case we use the /var/run one to let traefik see what docker is running, and the other two to connect to the traefik.toml & acme.json file.
          - /var/run/docker.sock:/var/run/docker.sock
          - /home/username/RichiHomeServer/traefik/traefik.toml:/traefik.toml
          - /home/username/RichiHomeServer/traefik/acme.json:/acme.json
        container_name: traefik # Names the container

This is the first chance you'll have to actually do some real editing. Change /home/username/ to your actual home folder location (in my case /home/richlab) and if you rename the cloned repository, you'll change RichiHomeServer to something else as well. We'll do some editing to traefik.toml later, but let's finish this docker compose first. Last but not least is:

        labels:
          - "traefik.enable=true"
          - "traefik.docker.network=web"
          - "traefik.frontend.rule=Host:traefik.domain.xyz" # You will change this to match your domain, an example would be traefik.richi.live. You can change the traefik part to whatever you want to use as a subdomain to show the traefik page.
          - "traefik.port=8080"
          - "traefik.frontend.auth.basic=Username:$$2y$$05$$ztcHlOeJs.4JbPziBrG5MuP1lf3ddNP/qm06ByT9Fd4/xzldxYw/2" # This part will require some more explaining, so please look below.
    
    networks:  
      web:
        external: true # This establishes that the web network traefik will be using is one established through docker already, and should not be created when you run the compose file.

I'll dictate what needs to be changed, but first we need to install something. To do this, run: 
`sudo apt install apache2-utils`
We'll use this to generate our auth password for the traefik page. If you want to access it, you'll need a password. Luckily, instead of keeping this password visible in plaintext, we can create a hashed password in our terminal. In the examples case, the username is Username nd the password is HardDabOnAllOfThesePeople. To change this we need to run a new command:
`echo $(htpasswd -nbB Username YourPassword) | sed -e s/\\$/\\$\\$/g`
You will change the Username part to something else I imagine, and the YourPassword bit to a (hopefully secure) password. I recommend using a password manager (might I suggest KeePass?). Once this command is run you will get something looking like this: `Username:$$Stringofnonsense$$etc.$$nonsense.` You will place that into the compose file in place of `"traefik.frontend.auth.basic=Username:$$2y$$05$$ztcHlOeJs.4JbPziBrG5MuP1lf3ddNP/qm06ByT9Fd4/xzldxYw/2"`

And that's the end of our traefik docker-compose.yml file! Let's edit our traefik.toml file next.
`nano traefik.toml`

We should see this: 

    debug = false
    logLevel = "ERROR"
    
    defaultEntryPoints = ["http", "https"]
    
    [web]
    # Port for the status page
    address = ":8080"
    
    [entryPoints]
        [entryPoints.http]
        address = ":80"
            [entryPoints.http.redirect]
            entryPoint = "https"
    
        [entryPoints.https]
        address = ":443"
        [entryPoints.https.tls]
    
    [retry]
    
    [docker]
    endpoint = "unix:///var/run/docker.sock"
    domain = "whatever.com"
    watch = true
    exposedByDefault = false
    
    [acme]
    email = "whatever@whatever.com"
    storage = "acme.json"
    entryPoint = "https"
    # onDemand = true
    onHostRule = true
    
    [acme.httpChallenge]
    entryPoint = "http"

    I'll only explain one bit of this, since that's all we'll edit:

        [docker]
    endpoint = "unix:///var/run/docker.sock"
    domain = "whatever.com"
    watch = true
    exposedByDefault = false
    
    [acme]
    email = "whatever@whatever.com"
    storage = "acme.json"
    entryPoint = "https"
    # onDemand = true
    onHostRule = true

We'll edit the domain (whatever.com) and the email (whatever@whatever.com) to whatever we want it to be. Once we do that, we're good to go.

We can test this now by running our docker-compose file. We'll do that using this command:
`docker-compose up -d`
It should start up without a hitch. You can now connect to your traefik instance by heading to the domain you used in the compose file, likely traefik.domain.com. After typing in the username and password you set, you'll be able to connect to the traefik dashboard. This will update as you add more containers in docker, but that'll be done without you editing and of traefiks config.

#### 2.1: Edit the necessary files - Services

Let's head into our services folder now. We'll use the nano command to edit our docker-compose file first.
`nano docker-compose.yml`

You should see the following:

    version: '3'
    
    services:
      db:
        image: mariadb
        restart: always
        volumes:
          - ./db:/var/lib/mysql
        environment:
          - MYSQL_ROOT_PASSWORD=StrongSecurePassword
        env_file:
          - db.env
        networks:
          - web
          - internal

      app:
        image: nextcloud:latest
        restart: always
        volumes:
          - ./nextcloud/data:/var/www/html
        env_file:
          - db.env
        environment:
          - MYSQL_HOST=db
        depends_on:
          - db
        networks:
          - web
          - internal
        labels:
          - "traefik.backend=nextcloud"
          - "traefik.docker.network=web"
          - "traefik.enable=true"
          - "traefik.frontend.rule=Host:nextcloud.domain.xyz"
          - "traefik.port=80"
    
      jellyfin:
        image: jellyfin/jellyfin
        restart: always
        volumes:
          - /home/username/media/:/media
        environment:
          - PUID=${PUID}
          - PGID=${PGID}
          - TZ=${TZ}
        networks:
          - web
          - internal
        labels:
          - "traefik.enable=true"
          - "traefik.backend=jellyfin"
          - "traefik.frontend.rule=Host:media.domain.xyz"
          - "traefik.port=8096"
          - "traefik.protocol=http"
          - "traefik.docker.network=web"
          - "traefik.frontend.headers.SSLRedirect=true"
          - "traefik.frontend.headers.STSSeconds=315360000"
          - "traefik.frontend.headers.browserXSSFilter=true"
          - "traefik.frontend.headers.contentTypeNosniff=true"
          - "traefik.frontend.headers.forceSTSHeader=true"
          - "traefik.frontend.headers.SSLHost=domain.xyz"
          - "traefik.frontend.headers.STSIncludeSubdomains=true"
          - "traefik.frontend.headers.STSPreload=true"
          - "traefik.frontend.headers.frameDeny=true"
    
    networks:
      internal:
      web:
        external: true
    
    volumes:
      db:
      nextcloud:


This is a much larger group of text than the last one, but it'll be just as easy to edit it. Here are the sections you'll need to tinker with:

        environment:
          - MYSQL_ROOT_PASSWORD=StrongSecurePassword
Change StrongSecurePassword to something else. 
That's all we have to do in the db container. Let's move down to the nextcloud section:

        labels:
          - "traefik.backend=nextcloud"
          - "traefik.docker.network=web"
          - "traefik.enable=true"
          - "traefik.frontend.rule=Host:nextcloud.domain.xyz"
          - "traefik.port=80"
You'll be changing nextcloud.domain.xyz to whatever you want to use to access your nextcloud. For example: cloud.richi.live. You might also note this:

        volumes:
          - ./nextcloud/data:/var/www/html
This will create a nextcloud/data folder in the services folder which will house all of your data. If you want to use a different folder (or a different drive), you just need to change it to the path of that folder. That's all there is to do in the nextcloud section. Let's move on to the jellyfin one:

        volumes:
          - /home/username/media/:/media
You'll change the /home/username/media to the path of your movies and TV shows and whatever other media you have.  
Continuing:

        labels:
          - "traefik.enable=true"
          - "traefik.backend=jellyfin"
          - "traefik.frontend.rule=Host:media.domain.xyz"
          - "traefik.port=8096"
          - "traefik.protocol=http"
          - "traefik.docker.network=web"
          - "traefik.frontend.headers.SSLRedirect=true"
          - "traefik.frontend.headers.STSSeconds=315360000"
          - "traefik.frontend.headers.browserXSSFilter=true"
          - "traefik.frontend.headers.contentTypeNosniff=true"
          - "traefik.frontend.headers.forceSTSHeader=true"
          - "traefik.frontend.headers.SSLHost=domain.xyz"
          - "traefik.frontend.headers.STSIncludeSubdomains=true"
          - "traefik.frontend.headers.STSPreload=true"
          - "traefik.frontend.headers.frameDeny=true"
We have a few things to edit. You'll change the traefik.frontend.rule label from media.domain.xyz to your desired domain, as well as the traefik.frontend.headers.SSLHost label to your domain (not including the subdomain!). You don't need to edit anything else. You *might* need to port forward port 8096, but I don't think you have to. If you can connect to NC but not jellyfin, you can do that, or change traefik.port=8096 to traefik.port=80.  
  
We only have one more *potential* file to update! That will be db.env, if you feel the need (which you really shouldn't). If you choose too, I imagine you're smart enough to do it. Otherwise, don't worry.  
  
Now, you can startup this docker-compose file. Let's run our command again:
`docker-compose up -d`
Let it spin up and then try to connect to your Nextcloud instance. It should ask for you to create a username and password. Do this, but don't move on. You should click on the "Database and Storage" option now. You'll be filling in some details here. Click on the MySQL/MariaDB tab and for the first 3 fields, simply type in nextcloud, and for the last you should only need to type in db (though if it fails you may need to try nextcloud_db_1 or whatever the mariadb image is called in the 'docker ps' command). If that doesn't work you should be able to click on the SQLite option and move forward from there, though performance may be slightly impacted (but not much, judging from my testing).  
  
Nextcloud should now be setup and you can connect to it using your domain or the desktop client. Jellyfin is actually easier to setup, requiring very little work. You simply go to your set subdomain, fill in the details where they apply, and when you select your media folder, just locate the *media* folder you indicated in the docker-compose folder. 

That should be it now, if there's any updates or issues, by all means, let me know. 
