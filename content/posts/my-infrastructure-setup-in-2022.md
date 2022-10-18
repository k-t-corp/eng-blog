---
title: "My Infrastructure Setup in 2022"
date: 2022-06-04T00:00:00-07:00
draft: false
---

TL;DR: In 2022 I moved infrastructure of K T Corp. from Kubernetes back to a simpler, droplet-based architecture.

For those of us who maintain side-projects, one of the headaches  is to deploy them somewhere on the Internet. I've personally experimented with several designs, and I'm currently landing on a simple one: just bunch of droplets.

But before we dive into this architecture, allow me to go through the ones that I had experimented with before.

# 1. Heroku dynos
I started with Heroku a few years back, but it had two severe limitations
* Too expensive once you need to run a worker and a clock process: $7/dyno is ok for a project that only needs a main web process, but once it needs a separate worker process for async jobs and a separate clock process for triggering those async jobs, $21/project suddenly becomes unaffordable for projects that don't generate revenue yet. Consider the amount of processing power that a clock process needs and the $7 you pay for just that, it is not a good deal.
* For non-enterprise customers, Asia regions are not available.

And to be honest, with the recent [GitHub integration security incident](https://status.heroku.com/incidents/2413), it seems to me Heroku is not getting the attention from its parent company Salesforce anymore.

# 2. Dokku on a single droplet
Dissatisfied with Heroku, I started to look for alternatives. I ultimately landed on "Dokku on a single droplet", and I was thinking of the following benefits

* A $40 droplet is definitely cheaper than paying $21 per project
* I can keep my Procfile's and migrate my projects seamlessly
* I can easily provision the droplet in Asia regions (although I didn't end up doing this)

It worked well for a while, but then I started to realize several drawbacks with this architecture

* Database access was slow. Most of my applications use MongoDB, and I chose to use [MongoDB Atlas](https://www.mongodb.com/atlas/database) and let my applications connect to their databases through the Internet. As a result, the latency for database operations was obviously high to a point that made me reconsider the whole architecture. In retrospective, this was a problem for the Heroku architecture as well, but it was during this iteration of architecture that made me realize the problem.
* Compartmentalization was not great. Since I put all my projects into one droplet on Dokku, one of them going rogue could have caused all of my projects go down.
* Dokku wasn't a convenient platform to bootstrap new applications with. I had to SSH into the Dokku host, run a bunch of commands, provision the MongoDB on MongoDB Atlas manually, and copy database credentials to the Dokku host. In retrospective, I could've documented those steps better to make my life easier, but at that time, I thought with the Heroku architecture I could've at least write Terraform to automate Heroku and MongoDB Atlas altogethers, whereas for Dokku there wasn't a mature Terraform provider available.
* Dokku was not a polished platform from an UX perspective. I missed a lot of the UI experience from Heroku, for example, viewing list of applications, viewing application add-ons (I had Dokku provision local Redis for my applications), viewing metrics, viewing environment variables, etc.

# 3. Managed Kubernetes with Terraform
Keeping the drawbacks of Heroku and Dokku in mind, I started to plan for the next iteration of my infrastructure. This time I had those clear goals in mind

* Cost-effective
* Available in Asia regions
* Local database access
* Compartmentalized
* Easy to bootstrap new projects
* Decent UX

I ultimately decided on "managed Kubernetes with infrastructure declarations written in Terraform". Here were the reasonings based on my goals

* Cost-effective: I decided on DigitalOcean's managed Kubernetes offering, which is a bit more expensive than the single Dokku host, but I was happy the slight price increase due to the other advantages it would have give. This is still cheaper than Heroku, and if I wasn't happy with DigitalOcean's pricing and I had my Kubernetes cluster basis declared in Terraform as well, I could've easily moved the cluster to somewhere else like Linode (that's one of the biggest advantage with Kubernetes).
* Available in Asia regions: I could provision the Kubernetes cluster in Asia regions.
* Local database access: I could provision the database adjacent to my application.
* Compartmentalized: This comes naturally with a Kubernetes cluster, if you have more than one worker node.
* Easy to bootstrap new projects: I could make a Terraform module that consists of both the application and its caches and databases.
* Decent UX: Being Kubernetes, I imagined it would have tons of projects that built UI on top of it (I ended up using [Lens](https://k8slens.dev) a lot)

Having justified the advantages of this architecture, I started by moving application peripherals such as S3 into Terraform, so that I didn't have to manually copy S3 bucket credentials over to my application Terraform later.

It was not easy. I was using Terraform Cloud to manage my Terraform state, and in order to import an existing S3 bucket, I had to [dump its credentials into a tfvars file in my local Terraform repo](https://github.com/hashicorp/terraform/issues/23407). This alone broke the promise of Terraform "Cloud', but I was alright with it, because importing existing S3 buckets was just a one-time thing.

I then started to build the container registry and container CI/CD. It wasn't trivial to accomplish either.

For CI/CD I uses GitLab CI, since my project code repos are mostly on GitLab.

Without Heroku or Dokku, I had to first write a production-ready Dockerfile for the programming languages that my projects are written in. I could've used [Cloud Native Buildpacks](https://buildpacks.io), but the images I was looking to use are > 1GB, and I did not like the size both in terms of container registry storage and CI/CD performance.

I also needed a container registry. Setting one up so that both GitLab CI and the Kubernetes cluster can talk to it wasn't an easy feat in Terraform. I could've used a managed container registry, but being cost-conscious, I decided to build it on the same Kubernetes cluster that my applications run. I implemented a basic one on the cluster, but I did't have the time to figured out advanced features such as container image recycling so that old images don't occupy disk space indefinitely.

I then had to write GitLab CI scripts to build the container, push it to the in-cluster container registry, and rollout the new version of the container on the cluster. I wrote some very basic scripts for the purpose, but it was a lot left to be desired since I didn't have time to implement advanced features such as gradual rollout and healthcheck applications after rollout. In retrospective, I could've used tools like ArgoCD, but I thought it just added another layer of complexity that I didn't have the bandwidth to understand.

After doing all those, I was finally able to deploy stateless applications from my GitLab repos to my Kubernetes cluster through my own container registry. In order to provision a new project, all I had to do was to use a Terraform module in my Terraform repo and pass it some Procfile-like parameters.

This whole Kubernetes infrastructure project was already occupying a lot of my time, but my next (and hopefully final?) logical step would be to move the databases into the cluster as well. At that time I had more than MongoDB to migrate, so I was trying to look for the "de-facto" Kubernetes Operator for each type of the database: [MongoDB](https://www.mongodb.com/docs/kubernetes-operator/master/), [Postgres](https://github.com/zalando/postgres-operator) and [ElasticSearch](https://www.elastic.co/blog/introducing-elastic-cloud-on-kubernetes-the-elasticsearch-operator-and-beyond).

This is the moment when I realize the infrastructure project is beyond me, at least as a solo dev without prior Kubernetes knowledge. I was very worried about its complexity and not being able to understand even one Kubernetes Operator for my database need, let alone all three. I finally tried to look at [KubeDB](https://kubedb.com) which claims to handle all different sorts of databases, but at that moment, the overwhelming amount of Kubernetes infrastructure related issues on my backlog made me rethink this architecture.

I've spent maybe more than 60 hours on this infrastructure project and still had an imperfect setup, and I wanted to stop investing more project time for infrastructure and use time for actual business logic.

# 4. A crowd of droplets
Before ultimately destroying my Kubernetes cluster and most of my Terraform code on 2022, there seems to be an Internet "renaissance" on simple, single-node "architecture" for small projects: run an application and a sqlite database on a single machine, just like it's 2005.

After evaluating this trend against my criteria, I decided to give it a try: I would basically have a bunch of droplets, each hosting one application, and each application has their MongoDB or Postgres or ElasticSearch colocated on the same machine. Here is the same goal run-down:

* Cost-effective: It is not as cheap as Dokku, and it's roughly as expensive as Kubernetes if not a bit cheaper. It's still definitely cheaper than Heroku.
* Available in Asia regions: I can spin up droplets in Asia regions much easier than spinning up new Kubernetes clusters. Although I tried to abstract the Kubernetes cluster as a Terraform module, I haven't actually used the module to spin up a second cluster, and I definitely foresee roadblocks if I tried to do so.
* Local database access: It is much easier than Kubernetes. I can just install the database software from `apt` and use it on the same machine the runs my code.
* Compartmentalized: Every application is contained in a droplet. If the droplet fails, it doesn't affect other applications.
* Easy to bootstrap new projects: This is one of the shortcomings, but it is possible to build automations that work around the issue.
* Decent UX: This is another shortcoming that is work-around-able.

I then started to provision my applications one-by-one on droplets. During the process, I did two things very consciously so that the two shortcomings of this architecture could be minimized in the future:

* While I provision the applications, I gathered all tutorials, information, command-line executions and tips/tricks into one single documentation, so that when I deploy the next application, I can just follow the documentation. This documentation is also a good reference for what I did earlier when I need to fix existing applications.
* I made sure naming is consistent. For example, all usernames for the Linux user that runs the application are the same; all systemd unit filenames that runs the web process are the same; all systemd unit filenames that runs worker processes are the same; all folder paths that store application code are the same; all file paths that store application config are the same. This ensures that when I need to manage the applications after provisioning, for example, restarting the web process for an application, I don't need to double guess its systemd unit filename, because the name is already documented and well-known.

In terms of CI/CD for my applications, there are two types: 

* For open-source self-hosted applications such as [RSSHub](https://github.com/DIYgod/RSSHub) and [Nitter](https://github.com/zedeus/nitter), I'd install Docker on the droplet, and either use the docker-compose file provided by the application or write my own docker-compose file if they don't provide one. Luckily all of them provide public Docker images, so I don't have to build the docker images. For upgrading those applications, if the application doesn't require manual operations, e.g. database migrations, then I just setup [watchtower](https://github.com/containrrr/watchtower) to refresh and restart the containers once in a while. Otherwise I'd manually manage the upgrades.
* For my proprietary applications, I wrote a [simple deployment daemon](https://github.com/k-t-corp/apollo/tree/master/apollo-cd) that watches a well-known location on the host for new application code, unzips and builds the application code into another well-known location, and restarts a well-known systemd unit that runs the built application. GitLab CI would just zip and scp the latest application code into the well-known location of the application host on every master commit.

I eventually finished migrating all of my applications from Kubernetes into droplets. Including migrating the databases (mostly MongoDB Atlas) into locally running MongoDB's, it took me just < 10 hours to research, develop, experiment, and get all my applications running on the droplets.

I think one of the biggest advantages of this setup is understandablility: It is so easy to understand. There are not many abstractions and blackboxes. Fundamentally everything is based on Linux hosts, which is well-understood, time-tested and have ample toolings and resources.

There could be more extensions to this architecture, such as building CLI tools and UX based on the consistent naming, and converting the documentation to provisioning automation, but if there is one lesson that I've learned from messing up Kubernetes, it is that I don't need to build them until I actually need them, and as a solo developer, I think this infrastructure setup is just worry-free even at its current state.  

# Appendix: PaSS honorable mentions
To answer some of the "have you tried this" questions

* Netlify: A simple and reliable platform. I use it to deploy all of my frontend code and will likely continue to do so in the future.
* Vercel: Tried to use it to deploy Next.js code.
* fly.io: Another simple and reliable platform. I have one stateless service running there. A potential next Heroku for me to experiment with projects quickly.
* porter.dev: Heard of it as a self-hosted Heroku, but seems too complicated to actually self-host if Kubernetes is involved.
* Render: At the time of writing too expensive for me.
* DigitalOcean App Platform: At the time of writing too expensive for me.
* Elastic Beanstalk or ECS or EKS: Most complicated because of IAM and VPC. Also expensive.
* CapRover: I was knocked off by its "yet another different Procfile format". Also it looks similar to Dokku and would likely inherit a lot of its shortcomings.
* Cloudflare suite of Serverless things: Doesn't suit my use cases for now but looks interesting.
