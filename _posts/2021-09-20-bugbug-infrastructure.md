---
title: "bugbug infrastructure: continuous integration, multi-stage deployments, training and production services"
tags: [machine-learning]
---

[bugbug](https://github.com/mozilla/bugbug) started as a project [to automatically assign a type to bugs](https://marco-c.github.io/2019/01/18/bugbug.html) (defect vs enhancement vs task, back when we introduced the "type" we needed a way to fill it for already existing bugs), and then evolved to be a platform to build ML models on bug reports: we now have many models, some of which are being used on Bugzilla, e.g. to assign a type, [to assign a component](https://hacks.mozilla.org/2019/04/teaching-machines-to-triage-firefox-bugs/), to close bugs detected as spam, to detect "regression" bugs, and so on.

Then, it evolved to be a platform to build ML models for generic software engineering purposes: we now no longer only have models that operate on bug reports, but also on test data, patches/commits (e.g. [to choose which tests to run for a given patch](https://hacks.mozilla.org/2020/07/testing-firefox-more-efficiently-with-machine-learning/) and to evaluate the regression riskiness associated to a patch), and so on.

Its infrastructure also evolved over time and slowly became more complex. This post attempts to clarify its overall infrastructure, composed of multiple pipelines and multi-stage deployments.

The nice aspect of the continuous integration, deployment and production services of bugbug is that almost all of them are running completely on Taskcluster, with a common language to define tasks, resources, and so on.

In bugbug's case, I consider a release as a code artifact (source code at a given tag in our repo) plus the ML models that were trained with that code artifact and the data that was used to train them.
This is because the results of a given model are influenced by all these aspects, not just the code as in other kinds of software.
Thus, in the remainder of this post, I will refer to "code artifact" or "code release" when talking about a new version of the source code, and to "release" when talking about a set of artifacts that were built with a specific snapshot (version) of the source code and with a specific snapshot of the data.

The overall infrastructure can be seen in this flowchart, where the nodes represent artifacts and the subgraphs represent the set of operations performed on them. The following sections of the blog post will then describe the components of the flowchart in more detail.
<img src="/assets/bugbug_infrastructure.svg" alt="Flowchart of the bugbug infrastructure" />

### Continuous Integration and First Stage (Training Pipeline) Deployment

Every pull request and push to the repository triggers [a pipeline of Taskcluster](https://github.com/mozilla/bugbug/blob/6e14270/.taskcluster.yml) tasks to:
- run tests for the library and its linked HTTP service;
- run static analysis and linting;
- build Python packages;
- build the frontend;
- build Docker images.

Code releases are represented by tags. A push of a tag triggers additional tasks that perform:
- integration tests;
- push of Docker images to DockerHub;
- release of a new version of the Python package on PyPI;
- update of the training pipeline definition.

After a code release, the training pipeline which performs ML training is updated, but the HTTP service, the frontend and all the production pipelines that depend on the trained ML models (the actual release) are still on the previous version of the code (since they can't be updated until the new models are trained).

### Continuous Training and Second Stage (ML Model Services) Deployment

The [training pipeline](https://github.com/mozilla/bugbug/blob/cb23f63/infra/data-pipeline.yml) runs on Taskcluster as a hook that is either triggered manually or on a cron.

The training pipeline consists of many tasks that:
- retrieve data from multiple sources (version control system, bug tracking systems, Firefox CI, etc.);
- generation of intermediate artifacts that are used by later stages of the pipeline or by other pipelines or other services;
- training of ML models using the above (there are also training tasks that depend on other models to be trained and run first to generate intermediate artifacts);
- check training metrics to ensure there are no short term or long term regressions;
- run integration tests with the trained models;
- build Docker images with the trained models;
- push Docker images with the trained models;
- update the production pipelines definition.

After a run of the training pipeline, the HTTP service and all the production pipelines are updated to the latest version of the code (if they weren't already) and to the last version of the trained models.

### Production pipelines

There are multiple production pipelines ([here's an example](https://github.com/mozilla/bugbug/blob/2cad435/infra/landings-pipeline.yml)), that serve different objectives, all running on Taskcluster and triggered either on cron or by pulse messages from other services.

### Frontend

The bugbug UI lives at [https://changes.moz.tools/](https://changes.moz.tools/), and it is simply a static frontend built in one of the production pipelines defined in Taskcluster.

The production pipeline performs a build and uploads the artifact to S3 via Taskcluster, which is then exposed at the URL mentioned earlier.

### HTTP Service

The HTTP service is the only piece of the infrastructure that is not running on Taskcluster, but currently on Heroku.

The Docker images for the service are built as part of the training pipeline in Taskcluster, the trained ML models are included in the Docker images themselves. This way, it is possible to rollback to an earlier version of the code and models, should a new one present a regression.

There is one web worker that answers to requests from users, and multiple background workers that perform ML model evaluations. These must be done in the background because of performance reasons (the web worker must answer quickly). The ML evaluations themselves are quick, and so could be directly done in the web worker, but the input data preparation can be slow as it requires interaction with external services such as Bugzilla or a remote Mercurial server.