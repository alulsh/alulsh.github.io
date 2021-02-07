---
name: Docker .npmrc security
type: security
repo: docker-npmrc-security
website: https://www.alexandraulsh.com/2018/06/25/docker-npmrc-security/
image: /images/blog/docker-npmrc-secure/docker-history.png
image_alt: Npm tokens leaking in the docker history
order_number: 1
---

{% include project.html %}

This is a companion repo with code samples for a blog post I wrote titled _[Securely using .npmrc files in Docker images](https://www.alexandraulsh.com/2018/06/25/docker-npmrc-security/)_. I published a follow up post called _[Docker build secrets and private npm packages](https://www.alexandraulsh.com/2019/02/24/docker-build-secrets-and-npmrc/)_ several months later.

I discovered this issue while launching [Mapbox Atlas v2](https://blog.mapbox.com/atlas-gets-an-upgrade-access-vector-maps-studio-offline-59074b81124a) in August 2018. [Mapbox Atlas](https://www.mapbox.com/atlas) is a self-hosted container-based version of the Mapbox platform.