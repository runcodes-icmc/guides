# Deployment Guide

This guide will walk you through the process of deploying the run.codes project to a server, thinking about a
production environment. For local development, we recommend to just use the included Docker Compose setup on
most of the repositories.

## Deployment Strategies

There are multiple ways by which you can deploy the project, each with its own advantages and disadvantages.

Here are the main ones:

### Docker Compose - Single Machine

This is the simplest deployment strategy, and the one we recommend for most cases. It uses Docker Compose to
orchestrate the deployment of the project, using Docker behind the scenes to run the containers. It supports
deployment to a single machine, and is the easiest to setup.

You can also generalize it to deploy to multiple machines, but if that's your goal, it might be better to use
the Ansible strategy as it keeps the configuration consistent accross the various services.

**To read more about this deployment strategy, check the [Docker Compose - Single Machine README](./compose-single/README.md).**

### Ansible (with Docker behind the scenes)

This one is the most complex, but also the most flexible. It uses Ansible to orchestrate the deployment of the
project, using Docker behind the scenes to run the containers. It supports deployment the services to multiple
machines, managing the configuration of them.

**The documentation for this deployment strategy is still a work in progress.**

### Kubernetes

This is the most complex deployment strategy, but also the most flexible. It uses Kubernetes to orchestrate the
deployment of the project. It is overkill and likely not worth it for small deployments, but if you are already
using Kubernetes for other projects, it might be worth it to use it for this one too.

**The documentation for this deployment strategy is still a work in progress.**
