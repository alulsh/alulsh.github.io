---
layout:     post
title:      Docker build secrets and private npm packages
date:       2019-02-24
summary:    Protect your .npmrc files with the new --secret flag in Docker 18.09. Use this flag to securely download private npm packages in Docker images.
---

## Multi-stage builds

On June 25th, 2018 I published a [blog post on using Docker multi-stage builds](/2018/06/25/docker-npmrc-security/) to securely download private npm packages. At the time this was the most secure method for using `.npmrc` files in Docker builds.

Instead of rewriting that old post I'm publishing a follow-up post on Docker build secrets. This blog post assumes you've read my previous post on [multi-stage builds](/2018/06/25/docker-npmrc-security/).

## Docker build secrets

In November 2018 [Docker 18.09](https://docs.docker.com/engine/release-notes/#18090) introduced a new `--secret` flag for `docker build`. This allows us to pass secrets from a file to our Docker builds. These secrets aren't saved in the final Docker image, any intermediate images, or the image commit history. With build secrets, you can now securely build Docker images with private npm packages without build arguments and multi-stage builds.

## Enabling BuildKit support

To use build secrets you'll first need to enable support for [Moby BuildKit](https://github.com/moby/buildkit). If you don't enable BuildKit you'll get the error message `Error response from daemon: Dockerfile parse error line 7: Unknown flag: mount` when trying to use build secrets.

To enable BuildKit, run `export DOCKER_BUILDKIT=1`. If you don't want to set this environment variable you can instead prepend `docker build` commands with `DOCKER_BUILDKIT=1`.

Docker 18.09 is the first version to ship with [optional BuildKit support](https://docs.docker.com/develop/develop-images/build_enhancements/). [Eventually, BuildKit will be the default](https://medium.com/@tonistiigi/build-secrets-and-ssh-forwarding-in-docker-18-09-ae8161d066) Docker build engine.

## Using Docker build secrets with npm tokens

Here's a Dockerfile called [`Dockerfile-secure-secrets`](https://github.com/alulsh/docker-npmrc-security/blob/master/Dockerfile-secure-secrets) that uses build secrets to install a Node.js app using private npm packages. The full source code and instructions are available at [https://github.com/alulsh/docker-npmrc-security](https://github.com/alulsh/docker-npmrc-security).

```
# syntax = docker/dockerfile:1.0-experimental
FROM node:8.11.3-alpine

WORKDIR /private-app
COPY . /private-app

RUN --mount=type=secret,id=npm,target=/root/.npmrc npm install

EXPOSE 3000

CMD ["node","index.js"]
```

To build this Docker image locally, run `docker build . -f Dockerfile-secure-secrets -t secure-app-secrets --secret id=npm,src=$HOME/.npmrc`.

This docker build command passes your local `.npmrc` file at `$HOME/.npmrc` to the `RUN` command. Then `RUN --mount=type=secret,id=npm,target=/root/.npmrc` adds your `.npmrc` file to `/root/.npmrc` on the Docker image. The `npm install` command then uses the `/root/.npmrc` file to access and install private npm packages.

You can check out the full syntax for `--mount=type=secret` in the [Moby BuildKit documentation](https://github.com/moby/buildkit/blob/master/frontend/dockerfile/docs/experimental.md#run---mounttypesecret).

## Verifying no secrets

To verify Docker build secrets didn't leak our `.npmrc` file or npm tokens, run `docker history secure-app-secrets`.

![Verifying docker build secrets](/images/blog/docker-build-secrets/docker-history.png)

{: style="text-align:center"}
_Verifying that build secrets did not leak our npmrc file or npm tokens_

Docker build secrets do leave a 0-byte blank file in our final Docker image though. It leaves this file at the target destination for our secrets, in this case `/root/.npmrc`. 

![Empty file for mounted secrets](/images/blog/docker-build-secrets/blank-file.png)

{: style="text-align:center"}
_Empty file bug with Docker build secrets_

This blank file is empty and so far there's no evidence it can be exploited. It's also a [known issue](https://github.com/docker/cli/pull/1288#pullrequestreview-146184186) from the Docker CLI pull request that added this feature in August 2018. I verified this bug still exists with `Docker version 18.09.2, build 6247962`.

## Experimental support?

Though the `--secret` flag appears in a production version of Docker, there are a few hints that support is still somewhat experimental.

For example, the first line of a Dockerfile using build secrets must be `# syntax = docker/dockerfile:1.0-experimental`. Without this line you'll get the error `failed to create LLB definition: Dockerfile parse error line 6: Unknown flag: mount`. This line enables the Docker CLI to use the "[experimental Dockerfile frontend](https://github.com/moby/buildkit/blob/master/frontend/dockerfile/docs/experimental.md#use-experimental-dockerfile-frontend)" for Moby BuildKit.

The build secrets [launch blog post](https://medium.com/@tonistiigi/build-secrets-and-ssh-forwarding-in-docker-18-09-ae8161d066) acknowledges that "secrets are currently not enabled in the stable channel of external Dockerfiles, so you need to use one of the releases in the experimental channel." The [Docker CLI pull request](https://github.com/docker/cli/pull/1288#issuecomment-413874936) mentions this feature doesn't use the stable default front end.

Even if the secrets CLI feature itself is stable, the external dockerfile it relies on is experimental. The syntax directive at the top of the Dockerfile feels like a hack or workaround that makes me feel uneasy about production usage.

## Conclusion

Docker build secrets finally provide a secure way to use secrets in Docker images. Now [multi-stage builds](https://docs.docker.com/develop/develop-images/multistage-build/) can be used for smaller Docker images instead of removing secrets.

Support for Docker build secrets still relies on an experimental Dockerfile frontend. As a result, Docker build secrets may or may not be ready for production use depending on your tolerance for risk. 

I'll update this blog post when Docker build secrets use a stable external Dockerfile frontend. We likely won't have to wait long!
