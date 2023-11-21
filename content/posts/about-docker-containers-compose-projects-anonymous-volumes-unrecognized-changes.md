+++
date = 2023-11-21T23:22:39+01:00
title = "About Docker Containers Compose Projects Anonymous Volumes Unrecognized Changes"
description = "The VOLUME directive in Dockerfiles can trick you into thinking your deploy pipeline will work as expected when in fact could end up not updating anything at all."
authors = ["Stefano Previato"]
tags = ["Docker", "volumes", "deploy", "pipeline"]
categories = ["Docker"]
+++

Today I spent a couple of hours trying to help a Data Engineering coworker find some nasty bug that was in hiding into one of our company workloads that are deployed using Docker.

## TLDR

A Docker deployment bug caused files not to update despite successful builds. It stemmed from an inherited VOLUME directive creating an anonymous volume, persisting changes. Solution: Add `docker compose down -v` in the deployment pipeline before `docker compose up`.

## The scenario

We have a GitLab repository that hosts a dbt project (it's not important [what dbt is](https://www.getdbt.com/), but if you're interested it's a data tool that Data engineers use, and I don't know anything about it), with the following `Dockerfile`:

```Dockerfile
FROM ghcr.io/dbt-labs/dbt-core:1.4.1 as build

# ...
# some RUN commands to install dependencies and whatnot, not relevant for our scenario
# ...

WORKDIR /usr/app
COPY ourproject/ .

# ...
# some other stuff copied over
# ...

RUN dbt deps
RUN dbt debug -t live
```

Seems simple enough to understand, even to somebody like me who has never done data stuff in their life.

Here's the `docker-compose.yml` file that spins this stuff up:

```yml
services:
  app:
    image: ourfancyregistry.lan/dbtstuff:latest
    restart: no
    entrypoint:
      - /bin/bash
      - -c
      - dbt run -t live
```

Here's the `Jenkinsfile` pipeline that builds and deploys this stuff:

```
node("linux-docker") {
  def gitData = checkout([$class: 'GitSCM', branches: [[name: '*/live']], extensions: [], userRemoteConfigs: [[credentialsId: 'JenkinsGitLab', url: 'https://ourfancygitlab.lan/xyz/dbtstuff']]])
  def commitHash = gitData["GIT_COMMIT"].substring(0, 7)

  stage("Build") {
    def dockerfile =
    docker.withRegistry("https://ourfancyregistry.lan") {
      def dockerImage = docker.build("dbtstuff:${commitHash}", ".")
      dockerImage.push()
      dockerImage.push("latest")
    }
  }

  stage("Deploy") {
    withCredentials([sshUserPrivateKey(credentialsId: 'REDACTED', keyFileVariable: 'sshIdentityFile', passphraseVariable: 'sshPassphrase', usernameVariable: 'sshUser')]) {
      def remote = [:]
      remote.name = "ourfancyserver.lan"
      # ...
      # some other ssh stuff not needed for this post
      # ...
      sshPut remote: remote, from: "./docker-compose.yml", into: "/srv/docker/dbtstuff/docker-compose.yml"
      sshCommand remote: remote, command: "cd /srv/docker/dbtstuff && docker compose up -d --pull always"
    }
  }
}
```

Again, pretty simple build pipeline that builds the image on one of our Docker build machines and then pushes it to our internal registry using the “latest” tag and a tag representing the commit hash.

## The bug

Whenever my coworker pushed some changes to the GitLab repository and triggered the manual build pipeline in Jenkins, Jenkins would do its thing and deploy everything to the remote Docker host without errors.

The logs would show:

`Container dbtstuff-app-1 Recreate`

However somehow the files inside the container were not updated with the new ones from the latest image version.

## The debugging process

We tried writing a new file inside the GitLab repository and then doing a `cat` during the build process of the Docker image:

```Dockerfile
FROM ghcr.io/dbt-labs/dbt-core:1.4.1 as build

# ...
# some RUN commands to install dependencies and whatnot, not relevant for our scenario
# ...

WORKDIR /usr/app
COPY ourproject/ .

# debug
COPY newfile.txt .
RUN cat newfile.txt

# ...
# some RUN commands to install dependencies and whatnot, not relevant for our scenario
# ...

RUN dbt deps
RUN dbt debug -t live
```

As expected, the file was copied correctly and the `cat` command would return the content since the file was there. But somehow, it did not end up in the deployed container. Running a `docker compose exec -it app ls -lah` on the target machine did not show it at all.

To much of our surprise, we tried bringing the compose project up again by issuing a `docker compose up -d --pull always` to try and see if somehow the new image was not being pulled from our registry, but we always got:

```
[+] Pulling 1/1
✔ app Pulled
```

suspiciously instantaneously... it was not actually downloading anything since it already did when Jenkins told it to.

What surprised us is that when issuing a `docker compose down` and a `docker compose up -d --pull always`, everything seemed to fall into place correctly, because then running a `docker compose exec -it app ls -lah` showed us that our test file was there as supposed to.

What could it be that made Docker download the wrong image somehow? Was there a bug into Docker compose or the Docker engine itself? Did we do something wrong on our side?

## Anonymous volumes and the VOLUME directive

In the [Dockerfile reference guide](https://docs.docker.com/engine/reference/builder/#volume) there's a directive called `VOLUME`. This directive instructs the Docker engine to create an anonymous volume in the host that mounts in the specified directory in the container.

Why am I talking about volumes here? Our `docker-compose.yml` did not have any volumes mapped, and our Dockerfile did not have any VOLUME directives specified in it, so what's this all about?

Well, when you create a Dockerfile, you usually reference a base image as a starting point with the FROM directive, in our case `FROM ghcr.io/dbt-labs/dbt-core:1.4.1`. The base image is in turn based on some Dockerfile that starts from another image (or `FROM scratch` in some special cases), and in that Dockerfile there could be directives that subsequent layers will inherit. In our case, the dbt-core Dockerfile contained the following directive ([see the Dockerfile here](https://github.com/dbt-labs/dbt-core/blob/307a618ea80d4e074e9554eedab970c22eaf495c/docker/Dockerfile#L53)):

`VOLUME /usr/app`

This directive is inherited by our Dockerfile that starts from the base image created from this dbt Dockerfile, and thus ends up being used by Docker when starting a new container from our Dockerfile too!

This directive effectively mounts an anonymous volume (remember that we did not mount any volume in our `docker-compose.yml` file) at the first start of the container, and is persisted for each run of the same container until it is destroyed, for example by running a `docker compose down`.

We can see that it is mounted as an anonymous volume by issuing the `docker inspect dbtstuff-app-1` command and looking at the JSON manifest for the container:

```json
// ...
"Mounts": [
    {
        "Type": "volume",
        "Name": "b6536a91b06811db8da8b446e947bf8c69e6aef0247d9476271fb2d23ee07687",
        "Source": "/var/lib/docker/volumes/b6536a91b06811db8da8b446e947bf8c69e6aef0247d9476271fb2d23ee07687/_data",
        "Destination": "/usr/app",
        "Driver": "local",
        "Mode": "",
        "RW": true,
        "Propagation": ""
    }
],
// ...
```

As you can see, the `/usr/app` folder of the container has been mounted by Docker as an anonymous volume (hence the strange hash name it has) and will be persisted across restarts, meaning that a `docker compose up -d --pull always` will pull the new image, but will not overwrite anything that is located inside the volume mounted folders, thus preventing us from seeing changes that are built into the image itself.

## The thing that got me fooled

If you search the `docker inspect dbtstuff-app-1` output, you will see that there is an `Image` property with the SHA256 hash of the image the container is running (or that is created on, if the container is stopped). If you do a `docker compose up -d --pull always` you will see that the image SHA256 will change, tricking you into thinking the container was recreated when effectively it was not! Or at least it was recreated, but it kept the same manifest with the same configuration as far as volumes are concerned.

```json
// before pulling the image
"Image": "sha256:4849b591445c29d3a1bed5b616cb90205a6b02e938ef32a15d1de949cf422bd1",

// after pulling the image
"Image": "sha256:14999a5b0ffac8874d152fdd610c670c4881ec96e09a152b510ce395b6ea6533",
```

## The solution

You can just append a `docker compose down` command to your deploy pipeline, right before doing the `docker compose up` command, and everything will work fine. You can even go as far as adding a `-v` flag (meaning `docker compose down -v`) to get rid of anonymous volumes so that they do not pile up in your Docker host, but be careful that if you're using other named volumes in your `docker-compose.yml` file, they will be destroyed too!

## Conclusion

In newer versions of the `dbt-core` image that we used here, it seems [they got rid of that anonymous volume in the Dockerfile](https://github.com/dbt-labs/dbt-core/commit/30def98ed948658ea012071fcd8c249bde4360f0), but for a few reason we cannot upgrade yet to the newest one, so we ended up solving this problem by adding the `docker compose down -v` command in the Jenkinsfile right before deploying everything.
