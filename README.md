# Ubuntu Server + RKE2 + UDS + Karton
I'm starting on a Macbook Pro M3 (so Arm). I plan to work this on x86 in the near future. 
##
Start here: `https://github.com/morbidsteve/uds-core-rke2/blob/main/ubuntu-arm-rke2-install-notes.md`
##
After completing the above you will have a full up UDS environment. The thing about Karton is the images that are built and available are only for x86 so we have to build them for Arm.
This can be a hefty effort, basically you need to find all of the Dockerfiles so you can build the containers from scratch (but I have already done that on my machine). **FUTURE work: I need to pull that altogether and add here**
##
`git clone ` this repo
##
The way I have been doing this is to build the redis, minio, and postgres packages in one go, so that when I am troubleshooting to build the other images, I don't keep trying to pull from the ghcr repos.
##
It's best if you build the following packages on the system that you are running UDS on for development type activity for efficiency.
# Build minio, redis, and postgres

in the zarf.yaml, comment out all components except the minio, redis, and postgres.

kind: ZarfPackageConfig
metadata:
name: karton-playground
description: Zarf package of karton-playground
version: 0.0.1

components:
- name: minio
    [...]
- name: redis
    [...]
- name: postgres
    [...]


Then run `uds run dev` from the root folder. Once complete, you should have a working instantiation of the three in the `karton2` namespace.
Use the k9s that comes with zarf/uds: `zarf tools k9s` and go to namespaces, then narrow down to `karton2`.

After that is completed, and all of your deployments are up, go ahead and reverse the commenting out of the components, so that everything **but** minio, redis, and postgres are uncommented.


kind: ZarfPackageConfig
metadata:
name: karton-playground
description: Zarf package of karton-playground
version: 0.0.1

components:
- name: mwdb
    [...]
- name: mwdb-web
    [...]
- name: karton-system
    [...]
- name: karton-classifier
    [...]
- name: karton-dashboard
    [...]
- name: uds-exemption-package
    [...]
- name: karton-mwdb-reporter
    [...]

Now again, run `uds run dev` and once complete, you should have a running implementation of kraton-playground.

You'll need to update your /etc/hosts to add the DNS resolutions for the VirtualServices to be able to interact with it now.

For me, that's the following (note, Metallb sets up the two interfaces giving the IPs below, which can be seen in your services for admin and tenant gateways).
`192.168.64.201	sso.uds.dev minio.uds.dev mwdb.uds.dev mwdb-web.uds.dev mwdb.uds.dev postgres.uds.dev karton-dashboard.uds.dev`

`192.168.64.200	keycloak.admin.uds.dev`



# Starting from karton-playground
This effort starts with taking a look at karton-playground, which is the malware triage/analysis capability. 

It is published open source and is ran with docker compose. First thing we need to do is take all of the containers and build them for arm since they are only distributed for x86.
A quick google finds the following:

[mwdb-core and mwdb-web](https://github.com/CERT-Polska/mwdb-core)

[karton-mwdb-reporter](https://github.com/CERT-Polska/karton-mwdb-reporter)

[karton-system](https://github.com/CERT-Polska/karton)

[karton-dashboard](https://github.com/CERT-Polska/karton-dashboard)

[karton-classifier](https://github.com/CERT-Polska/karton-classifier)

We need to clone each of the repos so we can build the images to our architecture. We'll also need to determine what networking is happening that Docker does natively that we'll have to specify within K8s

The way I did this was create a `docker-build-space` folder in my home directory (I'm running as root atm on a Ubuntu server that I've instantiated RKE2 + Zarf init) where I pulled all of the repos above.


## mwdb-core
Starting from the top, once cloned, cd into the directory and lets take a look at the docker-compose.yaml. We see that the mwdb service pulls it's config from `dockerfile: deploy/docker/Dockerfile`

We can also immediately see that it depends on redis and postgres.

We can simply build the image from the rood mwdb-core folder, pointing at the Dockerfile with the following command `docker build -t mwdb:alive -f deploy/docker/Dockerfile .`

The result is an image `mwdb` with tag `alive` or `mwdb'alive`. 

## mwdb-web
Similar to the above, we can see in the docker-compose.yaml that mwdb-web is built from deploy/docker/Dockerfile-web, so follow the same steps as above.

Note that mwdb-web has `ports: 80:80` in it's Dockerfile. We'll have to figure this out later

## karton-mwdb-reporter
No different than the above

## karton-system
Same same

## karton-dashboard
Same same, note the command `CMD karton-dashboard run --host 0.0.0.0`. We'll have to figure this out later

## karton-classifier
Same same

# Docker images to k8s manifests
Now that we have the containers built, we can move onto figuring out how to get the Dockerfiles/docker-compose.yaml configs/images up and running in k8s.

## kompose
A bit of googling led me to kompose, I tried putting all of the definitions together to kompose everything at once, but due to my lack of experience in this area, I found it easiest to do them one at a time.

Once I did all of these transpositions, next came getting them all up and running in k8s. It's not really worth my time to go back and do each iteration here because it took a long time for me since this was my first time.

It's enough to say that there's a bunch of troubleshooting to get things up and working one at a time. 

# Helmify
After you have all of the images up and running, and working together in k8s, next is to translate all of the k8s manifests to helm charts. 

A bit off googling drove me towards using helmify to get this done. Again, I did one image at a time to make it easier on myself. Once I become more comfortable with helm charts, I'd probably do them all at once. 

Once I had helm charts, I iterated through each to troubleshoot their individual deployment errors until I get everything up and functioning together. 

# UDS integration
Now the hard part. UDS is a **secure** k8s platform. That means that necessarily there are features/functionality that are by default restricted or limited. Figuring out how to get each component up and running in UDS was quite a difficult endeavor.

What I found easiest or at least somewhat straightforward, was to clone the [UDS package template](https://github.com/uds-packages/template/tree/main)

From there, under charts, I created a karton directory and a uds-config directory.

Under the karton directory, I created a directory for each component, and within each, a subsequent templates directory. I copied configmaps, services, and deployments into the templates directory for each component. 

Several days of iterating, I modified different parts of the helm charts to get things up and running smoothly and seemingly able to communicate with each other.

## UDS take aways
To start, it's important to understand the [UDS package](https://uds.defenseunicorns.com/structure/packages/) setup and config.


# NEED TO COMPLETE