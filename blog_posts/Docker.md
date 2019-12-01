Clearwater and Docker
---------------------
We first blogged about running Clearwater on Docker back in [early 2015](Containers.md), and since then the clearwater-docker repo has been available, offering Dockerfiles and Docker Compose config for spinning up Clearwater deployments using Docker.

Recently we’ve been doing some work to further improve our Docker support: breaking clearwater-docker deployments up into smaller microservices (by splitting out our data stores into their own containers), improving scaling, and integrating Weave Scope for real time visualizations. To see how easy it is to run Clearwater as a set of microservices on Docker check out the demo video below. Or to try it for yourself head to the [clearwater-docker](https://github.com/Metaswitch/clearwater-docker) repo.

[Demo Video](https://www.youtube.com/watch?v=y2B64c1OR_8)
