---
layout:     post
title:      Securely using .npmrc files in Docker images
date:       2018-06-25
summary:    .npmrc files are often used insecurely in Docker images. Here's how you can use them securely.
---

## Building Docker images with private npm packages

I recently completed a security audit of Docker images for a project. These images used `.npmrc` files ([npm config files](https://docs.npmjs.com/files/npmrc)) to download private npm packages. By default, an `.npmrc` file contains a token with read/write access to your private npm packages. It looks like this:

`//registry.npmjs.org/:_authToken=<npm token>`

Most blog posts, Stack Overflow answers, and documentation recommend you delete `.npmrc` files from your `Dockerfile` after installing private npm packages. Many of these guides don't cover how to remove `.npmrc` files from [intermediate images](https://medium.com/@jessgreb01/digging-into-docker-layers-c22f948ed612) or npm tokens from the image commit history though. In fairness, most of these resources date from before Docker shipped [multi-stage builds](https://docs.docker.com/develop/develop-images/multistage-build/) in [Docker v17.05 in May 2017](https://docs.docker.com/release-notes/docker-ce/#17050-ce-2017-05-04). 

Multi-stage builds allow us to securely use `.npmrc` files in our Docker images. Only the intermediate images and commit history from the last build stage end up in our final Docker image. This enables us to `npm install` our private packages in earlier build stages without leaking our tokens in the final image.

## Overview

In this blog post, I'll first describe the common ways people use `.npmrc` files insecurely in Docker images. For each scenario, I'll show how an attacker can exploit it to steal your npm access tokens. Finally, I'll explain how [multi-stage builds](https://docs.docker.com/develop/develop-images/multistage-build/) enable you to securely use `.npmrc` files in your Docker images.

This blog post focuses on using Docker with Node.js and npm. The concepts I cover apply to any `Dockerfile` that uses tokens, passwords, or other secrets though.

I also created a companion GitHub repository for this blog post so you can follow along with my examples. You can check it out at [https://github.com/alulsh/secure-npmrc-docker](https://github.com/alulsh/secure-npmrc-docker).

I have revoked all npm tokens featured in all screenshots.

## #1 - Leaving `.npmrc` files in Docker containers

If you fail to remove your `.npmrc` file from your `Dockerfile`, it will be saved in your Docker image. It will exist on the file system of any Docker container you create from that image. 

Here's a `Dockerfile` where we create an `.npmrc` file but fail to delete it after running `npm install`:

```
FROM node:8.11.3-alpine
ARG NPM_TOKEN

WORKDIR /private-app
COPY . /private-app

RUN echo "//registry.npmjs.org/:_authToken=$NPM_TOKEN" > .npmrc
RUN npm install

EXPOSE 3000

CMD ["node","index.js"]
```

This `Dockerfile` creates the `.npmrc` file using an `NPM_TOKEN` environment variable that we pass in as a build argument (`ARG NPM_TOKEN`). To build it locally, first clone the companion GitHub repository at [https://github.com/alulsh/secure-npmrc-docker](https://github.com/alulsh/secure-npmrc-docker) then:

1. Run `npm token create --read-only` to create a [read only npm token](https://docs.npmjs.com/getting-started/working_with_tokens#how-to-create-a-new-read-only-token).
1. Run `export NPM_TOKEN=<npm token>` to set this token as an environment variable.
1. Run `docker build . -f Dockerfile-insecure-1 -t insecure-app-1 --build-arg NPM_TOKEN=$NPM_TOKEN`.

### Stealing `.npmrc` files from Docker containers

If an attacker compromises your Docker container or manages to execute arbitrary commands on your application servers, they can steal your npm tokens by running `ls -al && cat .npmrc`.

You can try it out yourself:

1. Run `docker run -it insecure-app-1 ash` to start the container. We need to use `ash` instead of `bash` since we're running Alpine Linux.
1. Run `ls -al`. You should see an `.npmrc` file in the `/private-app` directory.
1. Run `cat .npmrc`.

![Stealing npm tokens from a running Docker container](/images/blog/docker-npmrc-secure/insecure-1.png)

{: style="text-align:center"}
_Stealing npm tokens from a running Docker container_

## #2 - Leaving `.npmrc` files in Docker intermediate images

Most guides recommend deleting your `.npmrc` file after running `npm install` in your `Dockerfile`. For example:

```
ARG NPM_TOKEN

RUN echo "//registry.npmjs.org/:_authToken=$NPM_TOKEN" > .npmrc
RUN npm install
RUN rm -f .npmrc
```

Fortunately, this does remove the `.npmrc` file from the top layer of your Docker image and from any containers. If an attacker compromised your Docker container they would not be able to steal your npm token.

Unfortunately, each `RUN` instruction creates a separate layer (also called intermediate image) in a Docker image. [Multiple layers make up a Docker image](https://medium.com/@jessgreb01/digging-into-docker-layers-c22f948ed612). In the above `Dockerfile`, the `.npmrc` file is stored in the layer created by the `RUN echo "//registry.npmjs.org/:_authToken=$NPM_TOKEN" > .npmrc` instruction.

### Stealing `.npmrc` files from intermediate images

To view the layers of your Docker image an attacker would need to either compromise your Docker daemon or obtain a copy of your Docker image. You likely won't publish Docker images with private source code to public Docker registries. You might distribute these Docker images to contractors, consultants, and customers though. These third parties need your private source code but they do not need your npm tokens.

Here's how an attacker or third-party could steal `.npmrc` tokens from layers in your Docker images:

1. Build the example Docker image with `docker build . -f Dockerfile-insecure-2 -t insecure-app-2 --build-arg NPM_TOKEN=$NPM_TOKEN`.
1. Run `docker save insecure-app-2 -o ~/insecure-app-2.tar` to save the Docker image as a tarball.
1. Run `mkdir ~/insecure-app-2 && tar xf ~/insecure-app-2.tar -C ~/insecure-app-2` to untar to `~/insecure-app-2`.
1. Run `cd ~/insecure-app-2`. For fun, try running `ls` and `cat manifest.json` in this directory to view all of the layers.

![Viewing layers of a Docker image](/images/blog/docker-npmrc-secure/viewing-layers.png)

{: style="text-align:center"}
_Layers in a Docker image_

5. Run `for layer in */layer.tar; do tar -tf $layer | grep -w .npmrc && echo $layer; done`. [Credit goes to this StackOverflow answer](https://stackoverflow.com/questions/40575752/docker-extracting-a-layer-from-a-image/44030483#44030483) for that one liner. You should see a list of layers with `.npmrc` files.
6. Run `tar xf <layer id>/layer.tar private-app/.npmrc` to extract `private-app/.npmrc` from the layer tarball. In my case, I needed to run `tar xf 1c3c8a7a05b2ffddbdbd9b1e93f662f68efae9246302122f756aae908e41676c/layer.tar private-app/.npmrc`.
7. Run `cat private-app/.npmrc` to view the `.npmrc` file and npm token.

![Stealing .npmrc files from Docker layers](/images/blog/docker-npmrc-secure/insecure-2-layers.png)

{: style="text-align:center"}
_Stealing .npmrc files and npm tokens from Docker layers_

## #3 - Leaking npm tokens in the image commit history

More security conscious guides are aware of the Docker layer problem. They advocate creating and deleting the `.npmrc` file in the same `RUN` instruction or layer. Other guides recommend using the `--squash` flag when running `docker build`. Here's a `Dockerfile` where we create and delete our `.npmrc` file in the same layer:

```
ARG NPM_TOKEN

RUN echo "//registry.npmjs.org/:_authToken=$NPM_TOKEN" > .npmrc && \
    npm install && \
    rm -f .npmrc
```

Both approaches prevent `.npmrc` files from being saved in layers. Unfortunately, the npm token is still visible in the [commit history](https://docs.docker.com/engine/reference/commandline/history/) of the Docker image. The Docker image build process logs the plaintext values for build arguments (`ARG NPM_TOKEN`) into the commit history of an image. For this reason, the [official Docker documentation on Dockerfiles](https://docs.docker.com/engine/reference/builder/#arg) warns that you should not use build arguments for secrets.

### Stealing npm tokens from Docker image commit histories

Similar to viewing Docker layers, to view your image commit history an attacker or third-party would need to compromise your Docker daemon or obtain a copy of your Docker image. This "hack" is easier though and only requires one command - `docker history`. To try this yourself:

1. `docker build . -f Dockerfile-insecure-3 -t insecure-app-3 --build-arg NPM_TOKEN=$NPM_TOKEN`
1. `docker history insecure-app-3`

![Docker history](/images/blog/docker-npmrc-secure/docker-history.png)

{: style="text-align:center"}
_Stealing npm tokens from Docker image commit histories_

## Solution - Multi-stage builds

We can protect our build arguments from leaking with [multi-stage builds](https://docs.docker.com/develop/develop-images/multistage-build/). Only the final build in a multi-stage build will show up in the commit history of the final Docker image.

In the first stage of the build, we'll use our `NPM_TOKEN` build argument to `npm install` our project. We can then copy our built Node application from the first stage to the second stage build using the `COPY` instruction.

Most multi-stage build tutorials and examples show two different base images. You can use the same base image for multi-stage builds. In this example, I use `node:8.11.3-alpine` as my base image for both stages.

```
# First build
FROM node:8.11.3-alpine AS build
ARG NPM_TOKEN

WORKDIR /private-app
COPY . /private-app

RUN echo "//registry.npmjs.org/:_authToken=$NPM_TOKEN" > .npmrc && \
    npm install --production && \
    rm -f .npmrc

# Second build
FROM node:8.11.3-alpine
WORKDIR /private-app

EXPOSE 3000
COPY --from=build /private-app /private-app

CMD ["node","index.js"]
```

To build this image, run `docker build . -f Dockerfile-secure -t secure-app --build-arg NPM_TOKEN=$NPM_TOKEN`. To view the history, run `docker history secure-app`.

![Secure Docker commit history thanks to multi-stage builds](/images/blog/docker-npmrc-secure/secure-docker-history.png)

{: style="text-align:center"}
_Secure Docker commit history thanks to multi-stage builds_

Our npm tokens no longer leak in the commit history!

### Delete untagged images from multi-stage builds

Multi-stage builds [create untagged images of earlier build stages in our local Docker daemon](https://github.com/moby/moby/issues/13490#issuecomment-324730806) for the Docker build cache. The commit history of these images leaks plaintext build arguments. You should delete these images after you finish your builds.

Run `docker history` on these untagged images to steal npm tokens from them. Run `docker rmi <image id>` to delete them.

![Npm tokens in untagged images from multi-stage builds](/images/blog/docker-npmrc-secure/npm-token-untagged-image.png)

{: style="text-align:center"}
_Npm tokens in untagged images from multi-stage builds_

## Assessing risk

If an attacker compromises your containers or Docker daemon, you'll likely be dealing with bigger issues than your `.npmrc` files. Removing npm tokens from your Docker images means you'll be dealing with one less problem though.

Depending on your threat model, deleting `.npmrc` files from your Docker images and containers may be all you need to do to mitigate most of your risk. If you distribute your Docker images to third party contractors, consultants, or customers you should use multi-stage builds to remove any secrets. This also applies if you publish images to Docker registries.

Multi-stage builds are easy to use and have little to no downsides. They can significantly improve the security, performance, and readability of your Docker images. If you use secrets to build your Docker images I recommend multi-stage builds even if you don't distribute your Docker images and aren't worried about the security of your Docker daemon.

## Raising awareness of multi-stage builds

The Docker community is aware of [the lack of a built-in way to manage secrets in Dockerfiles](https://github.com/moby/moby/issues/13490). They're currently working on ways to improve using secrets in Docker. In the meantime multi-stage builds allow us to securely use `.npmrc` files or other secrets in our Docker builds. They also improve the readability of our Dockerfiles and decrease the size of our images.

Most of the guides for using `.npmrc` files in Docker images date from before multi-stage builds in May 2017. I'm currently drafting a pull request to [the official npm documentation](https://github.com/npm/docs) to update their guidance to use multi-stage builds. I'll update this blog post with a link to the pull request as well as updates on its status!

**Update - 22:40 EST June 25th 2018** - I submitted an [issue](https://github.com/npm/docs/issues/1020) and a separate [pull request](https://github.com/npm/docs/pull/1021) to [https://github.com/npm/docs](https://github.com/npm/docs) with new guidance about multi-stage builds. 