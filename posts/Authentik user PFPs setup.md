---
title: Auto cropping Authentik user avatars
description: Tutorial on how to setup user avatars in authentik.
date: '23/02/2025'
icon: https://cdn.kalitsune.net/admin/files/assets/Services/FoxID/Square.svg
author: Kalitsune
tags:
  - tutorial
  - active-directory
  - docker
  - image-manipulation
published: true
---

## Introduction
Authentik doesn't provide an easy way to implement avatars for users by default.
However it is very much doable thanks to the power of policies and attributes!

despite being new to authentik and its inner workings, I was able to easily find a tutorial that did more or less what I wanted:
https://github.com/goauthentik/authentik/discussions/6824

But it was missing something! You couldn't safely upload SVGs (and god knows I love SVGs), and if the picture you used wasn't square this could generate errors down the line with other services like [Nextcloud](https://nextcloud.com/).

In short: while the solution proposed was good, it was still unpolished.

That's why I took the liberty to improve it!

## Fixing the problem: imgproxy
Something tricky about the authentik setup is that it runs inside of a docker container so you'll have an hard time running additional software if it's not present on the docker image itself.

I tried using ffmpeg or image-magick using their appimage but those don't work on docker and I wasn't able to run them in any safe way. And it's not like I could install pillow either...

But if I couldn't run those inside of the docker container, I could use another container to expose an API that would allow me to manipulate the image from my little python script!

That's when I found [imgproxy](https://imgproxy.net/). A supper tiny, power-packed image processing webserver written in go. This project was a perfect fit for my needs! Time to implement it.

## Adding imgproxy to the compose file
By that point I had already tweaked my [original docker compose file](https://goauthentik.io/docker-compose.yml) quite a bit, notably by adding configs for [traefik](https://traefik.io/traefik/), my reverse proxy, so there might be extra stuff you might want to remove.

but here's the config I used:
```yaml
services:
  postgresql:
    image: docker.io/library/postgres:16-alpine
    restart: unless-stopped
    healthcheck:
      test:
        - CMD-SHELL
        - pg_isready -d $${POSTGRES_DB} -U $${POSTGRES_USER}
      start_period: 20s
      interval: 30s
      retries: 5
      timeout: 5s
    volumes:
      - database:/var/lib/postgresql/data
    environment:
      POSTGRES_PASSWORD: ${PG_PASS:?database password required}
      POSTGRES_USER: ${PG_USER:-authentik}
      POSTGRES_DB: ${PG_DB:-authentik}
    env_file:
      - .env
    networks:
      - default
  redis:
    image: docker.io/library/redis:alpine
    command: --save 60 1 --loglevel warning
    restart: unless-stopped
    healthcheck:
      test:
        - CMD-SHELL
        - redis-cli ping | grep PONG
      start_period: 20s
      interval: 30s
      retries: 5
      timeout: 3s
    volumes:
      - redis:/data
    networks:
      - default
  authentik:
    image: ${AUTHENTIK_IMAGE:-ghcr.io/goauthentik/server}:${AUTHENTIK_TAG:-2024.12.3}
    restart: unless-stopped
    command: server
    container_name: authentik
    labels:
      traefik.enable: true
      traefik.http.routers.authentik.rule: Host(`auth.example.com`)
      traefik.http.routers.authentik.tls.certresolver: letsencryptresolver
      traefik.http.routers.authentik.entrypoints: websecure
      traefik.http.routers.authentik.service: authentik-svc
      traefik.http.services.authentik-svc.loadBalancer.server.port: 9000
    environment:
      AUTHENTIK_REDIS__HOST: redis
      AUTHENTIK_POSTGRESQL__HOST: postgresql
      AUTHENTIK_POSTGRESQL__USER: ${PG_USER:-authentik}
      AUTHENTIK_POSTGRESQL__NAME: ${PG_DB:-authentik}
      AUTHENTIK_POSTGRESQL__PASSWORD: ${PG_PASS}
    volumes:
      - media:/media
      - templates:/templates
    env_file:
      - .env
    ports:
      - ${COMPOSE_PORT_HTTP:-9000}:9000
      - ${COMPOSE_PORT_HTTPS:-9443}:9443
    networks:
      - default
      - auth
      - nextcloud-aio
      - proxy
    depends_on:
      imgproxy:
        condition: service_started
      postgresql:
        condition: service_healthy
      redis:
        condition: service_healthy
  worker:
    image: ${AUTHENTIK_IMAGE:-ghcr.io/goauthentik/server}:${AUTHENTIK_TAG:-2024.12.3}
    restart: unless-stopped
    command: worker
    environment:
      AUTHENTIK_REDIS__HOST: redis
      AUTHENTIK_POSTGRESQL__HOST: postgresql
      AUTHENTIK_POSTGRESQL__USER: ${PG_USER:-authentik}
      AUTHENTIK_POSTGRESQL__NAME: ${PG_DB:-authentik}
      AUTHENTIK_POSTGRESQL__PASSWORD: ${PG_PASS}
    # `user: root` and the docker socket volume are optional.
    # See more for the docker socket integration here:
    # https://goauthentik.io/docs/outposts/integrations/docker
    # Removing `user: root` also prevents the worker from fixing the permissions
    # on the mounted folders, so when removing this make sure the folders have the correct UID/GID
    # (1000:1000 by default)
    user: root
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - media:/media
      - certs:/certs
      - templates:/templates
    env_file:
      - .env
    depends_on:
      postgresql:
        condition: service_healthy
      redis:
        condition: service_healthy
    networks:
      - default
  imgproxy:
    image: ghcr.io/imgproxy/imgproxy:latest
    networks:
      - default
volumes:
  database: null
  redis: null
  templates: null
  certs: null
  media: null
networks:
  default: null
  auth: null
  proxy:
    external: true
```

Network-wise, my configs uses 3 different networks:
- `default`: the default network for my authentik stack, it is only intended to be used by the services in this compose file.
- `auth`: a network binding the authentik server with the services that need to interact with it directly, mainly for LDAP.
- `proxy`: this is the network used by my reverse proxy traefik.

You're free to replicate my setup as you'd like, however only the default network is important here.

## Configuring the policy
Now that we've added an imgproxy instance to our authentik stack, we can create the policy that will be using it:

Go to `Customization> Policies` and create a new policy, select `Expression Policy`.
Call it whatever you'd like, I called mine `foxid-settings-avatar`.

And paste the following python script:
```py 
AK_DOMAIN = "auth.kalitsune.net"
FILE_PATH = "/media/user-avatars/"
URL_PATH = f"https://{AK_DOMAIN}{FILE_PATH}"
IMGPROXY_URL = "http://imgproxy:8080"  # URL where imgproxy is accessible
AK_LOCAL_URL = "http://authentik:9000" # Local url where imgproxy can access authentik. If none just put the same as AK_DOMAIN
TARGET_SIZE = 1024  # Desired square dimension for the avatar
ACCEPTED_FILE_TYPES = {
  "image/png": "png",
  "image/jpeg": "jpeg",
  "image/webp": "webp",
  "image/svg+xml": "svg", # imgproxy sanitizes svgs by default so this is fine (as long as you have this option enabled on your instance)
  "image/gif": "gif",
  "image/bmp": "bmp",
  "image/vnd.microsoft.icon": "ico"
}

EMPTY_FILE = "data:application/octet-stream;base64,"

from humanize import naturalsize
from uuid import uuid4
from base64 import b64decode
from os import remove
from os.path import isfile, join
import urllib.request
import urllib.parse
import base64

def remove_old_avatar_file():
    if not hasattr(request.user, "avatar"):
        return
    avatar = request.user.attributes.get("avatar", None)
    if avatar:
        components = avatar.split(URL_PATH, 1)
        if len(components) == 2 and components[0] == "" and components[1]:
            old_filename = FILE_PATH + components[1]
            if isfile(old_filename):
                remove(old_filename)

prompt_data = request.context.get("prompt_data")
avatar_overwritten = False

if "avatar" in prompt_data.get("attributes", {}):
    avatar = prompt_data["attributes"]["avatar"]
    if avatar == EMPTY_FILE:
        # No upload file specified, ignore
        del prompt_data["attributes"]["avatar"]
    else:
        avatar_mimetype = avatar.split("data:", 1)[1].split(";", 1)[0]
        if avatar_mimetype not in ACCEPTED_FILE_TYPES:
            ak_message("User avatar must be an image (" + ", ".join(ACCEPTED_FILE_TYPES.values()) + ") file.")
            return False
        # Decode the base64 data from the upload
        avatar_base64 = avatar.split(",", 1)[1]
        avatar_binary = b64decode(avatar_base64)
        
        # Write the original image to a temporary file in FILE_PATH.
        # This file must be accessible via the public URL (URL_PATH) for imgproxy.
        temp_filename = str(uuid4()) + "." + ACCEPTED_FILE_TYPES[avatar_mimetype]
        temp_filepath = join(FILE_PATH, temp_filename)
        try:
            with open(temp_filepath, "wb") as f:
                f.write(avatar_binary)
        except Exception as e:
            ak_message("Could not write temporary avatar file: " + str(e))
            return False
        
        # Construct the public URL for the temporary file.
        original_url = AK_LOCAL_URL + FILE_PATH + temp_filename
        
        # Build the imgproxy URL for a square crop using the rs:fill transformation.
        # This transformation will resize and crop the image to TARGET_SIZE x TARGET_SIZE.
        imgproxy_transform = f"rs:fill:{TARGET_SIZE}:{TARGET_SIZE}"
        imgproxy_url = f"{IMGPROXY_URL}/insecure/{imgproxy_transform}/plain/{original_url}@png"
        
        try:
            with urllib.request.urlopen(imgproxy_url) as response:
                cropped_binary = response.read()
        except Exception as e:
            ak_message("Error retrieving cropped image from imgproxy: " + str(e))
            remove(temp_filepath)
            return False
        
        # Clean up the temporary file.
        remove(temp_filepath)
        
        # Save the cropped image with a new random file name.
        avatar_filename = str(uuid4()) + ".png"
        final_filepath = FILE_PATH + avatar_filename
        try:
            with open(final_filepath, "wb") as f:
                f.write(cropped_binary)
        except Exception as e:
            ak_message("Could not write cropped avatar file: " + str(e))
            return False
        
        avatar_overwritten = True
        remove_old_avatar_file()
        prompt_data["attributes"]["avatar"] = URL_PATH + avatar_filename

if "avatar_reset" in prompt_data.get("attributes", {}):
    del prompt_data["attributes"]["avatar_reset"]
    # If a new avatar was uploaded, previous avatar is deleted anyway, so ignore
    if not avatar_overwritten:
        prompt_data["attributes"]["avatar"] = None
        remove_old_avatar_file()
        ak_message("Deleted user avatar.")

return True
```
If you want to check the original expression I used, you can do so [here](https://gist.github.com/drpetersen/c73b1362264c056c02f3ff8973d55d70).

## Final steps: Setting up the components
Now that we've set up the policy that'll be powering our avatar system, let's create all the components we need to add this system to our GUI:

### Creating the file upload and reset pfp prompts
In order for the user to upload their pfp, we'll need to create a few prompts.
Mainly a FileUpload prompt for the user to upload their pfp, and also a reset avatar prompt to set the avatar back to defaults.

In the admin panel go to `Flows and Stages> Prompts` and create a new prompt.
Set **Field Key** to `attributes.avatar` and **Type** to `File`.
Give it a **Name** and a **Label** and set an **Order** that makes sense in your setup (mine is `350`).
![File prompt Setup](/blog/AuthentikPFPSetup/FilePrompt.png)

And create another one with **Field Key** set to `attributes.avatar_reset` and **Type** to `Checkbox`.
Again set **Name**, **Label** and **Order** to whatever makes sense for you (my **Order** is set to `351`)
![Reset prompt Setup](/blog/AuthentikPFPSetup/ResetPrompt.png)

And we should have all the prompts needed!

### Adding it to the settings
Go to your Settings Flow (in `Flows and Stages> Flows`, the default setting flow is `default-user-settings-flow`)
Open the flow and go to `Stage Bindings` and click **Edit Stage** on `default-user-settings`.

In **Fields**, add the Avatar Upload and Avatar Reset prompts we've created:
![Prompt Fields](/blog/AuthentikPFPSetup/PromptFields.png)

And in **Validation Policies** add the policy we've created (mine is `foxid-settings-avatar`):
![Prompt Fields](/blog/AuthentikPFPSetup/PromptPolicies.png)

if everything was done correctly, you should now have the option to set your avatar in the settings:
![Prompt Fields](/blog/AuthentikPFPSetup/Result.png)

### Adding it in an enrollment flow
If you want to, you can allow user creating their accounts to upload a pfp by doing the same manipulation we did in the settings:
![Prompt Fields](/blog/AuthentikPFPSetup/Enrollement.png)
The policy is compatible with this usecase and will even sanitize and convert the uploaded files!
*please bear in mind that malicious users might try to exploit this feature!*

### Customization: show the user profile picture in authentik
In the admin settings go to `System> Settings` and add `attributes.avatar,` at the start of the `Avatar` field:
![Authentik Pfp Setup](/blog/AuthentikPFPSetup/ShowInAuthentik.png)
Your pfp should now appear!
![Authentik Pfp Setup](/blog/AuthentikPFPSetup/PFPInAuthentik.png)

### Bonus: Gravatar support and identicons
If you want, you can also enable authentik to use gravatars and identicons if the user hasn't selected a pfp!
To do so, add the following policy: 
```py 
IDENTICON_URL_TEMPLATE = "https://www.gravatar.com/avatar/{mail_hash}?d=identicon"
EMPTY_FILE = "data:application/octet-stream;base64,"

import hashlib

prompt_data = request.context.get("prompt_data", {})
attributes = prompt_data.get("attributes", {})

# If avatar is missing, is the empty file, or is explicitly set to None,
# then generate an identicon.
if ("avatar" not in attributes) or (attributes.get("avatar") in [EMPTY_FILE, None, "/static/dist/assets/images/user_default.png"]):
    # Retrieve the user's email; adjust if your email is stored elsewhere.
    email = None
    if hasattr(request.user, "email") and request.user.email:
        email = request.user.email
    elif "email" in attributes:
        email = attributes["email"]
    elif "email" in prompt_data:
        email = prompt_data["email"]
    if not email:
        ak_message("Email not provided for identicon generation.")
        return False
    # Compute the MD5 hash of the normalized email.
    mail_hash = hashlib.md5(email.strip().lower().encode()).hexdigest()
    # Generate and assign the identicon URL.
    attributes["avatar"] = IDENTICON_URL_TEMPLATE.format(mail_hash=mail_hash)
    prompt_data["attributes"] = attributes

return True
```

⚠️  This policy is intended to be ran after the first policy! I recommend using it on the User Write stage to ensure this happens:

![Identicons Setup](/blog/AuthentikPFPSetup/IdenticonSetup.png)

And if everything is configured correctly you should have something like this:
![Identicons](/blog/AuthentikPFPSetup/Identicon.png)
