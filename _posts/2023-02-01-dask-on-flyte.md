---
layout: post
title: Managing dask workloads with Flyte
author: Bernhard Stadlbauer
tags: [Python, dask, Flyte, Kubernetes]
theme: twitter
---

It is now possible to manage `dask` workloads using [Flyte](https://flyte.org/) 🎉!

The major advantages are:

- Each `task` will spin up a separate (ephemeral) `dask` cluster, using a docker image custom to the task. So Python environments are guaranteed to be consistent across the client, scheduler and workers.
- Flyte will use the existing [Kubernetes](https://kubernetes.io/) infrastructure to spin up `dask` clusters
- Spot/Preemtible instances are natively supported
- The whole `dask` task can be cached
- If you are already running Flyte, `dask` support can be enabled within minutes

This is what a Flyte `task` backed by a `dask` cluster with four workers looks like:

```python
from typing import List

from distributed import Client
from flytekit import task, Resources
from flytekitplugins.dask import Dask, WorkerGroup, Scheduler


def inc(x):
    return x + 1


@task(
    task_config=Dask(
        scheduler=Scheduler(
            requests=Resources(cpu="2")
        ),
        workers=WorkerGroup(
            number_of_workers=4,
            limits=Resources(cpu="8", mem="32Gi")
        )
    )
)
def increment_numbers(list_length: int) -> List[int]:
    client = Client()
    futures = client.map(inc, range(list_length))
    return client.gather(futures)
```

This task will run locally (using a regular `distributed.Client()`) and can then scale to arbitrary cluster sizes after is has been [registered with Flyte](https://docs.flyte.org/projects/cookbook/en/latest/auto/larger_apps/larger_apps_deploy.html).

## What is Flyte?

[Flyte](https://flyte.org/) is a Kubernetes native workflow orchestration engine. Originally developed at [Lyft](https://www.lyft.com/), it is now open source ([Github](https://github.com/flyteorg/flyte)) and Graduate Project from the Linux Foundation. Some it's core features that distinguish it from similar tools like [Airflow](https://airflow.apache.org/) or [Argo](https://argoproj.github.io/) are:

- Caching/Memoization of previously executed tasks
- Kubernetes native
- Workflows are defined as Python code, not e.g. `yaml`
- Strong typing between tasks and workflows using `protobuf`
- Dynamically generated workflow DAGs at runtime
- Workflows can be executed locally

A simple workflow would look something like the following:

```python
from typing import List

import pandas as pd

from flytekit import task, workflow, Resources
from flytekitplugins.dask import Dask, WorkerGroup, Scheduler


@task(
    task_config=Dask(
        scheduler=Scheduler(
            requests=Resources(cpu="2")
        ),
        workers=WorkerGroup(
            number_of_workers=4,
            limits=Resources(cpu="8", mem="32Gi")
        )
    )
)
def expensive_data_preparation(input_files: List[str]) -> pd.DataFrame:
    # Expensive, highly parallel `dask` code
    ...
    return pd.DataFrame(...)  # Some large DataFrame, Flyte will handle serialization


@task
def train(input_data: pd.DataFrame) -> str:
    # Model training, can also use GPU, etc.
    ...
    return "s3://path-to-model"


@workflow
def train_model(input_files: List[str]) -> str:
    prepared_data = expensive_data_preparation(input_files=input_files)
    return train(input_data=prepared_data)
```

In the above, both `expensive_data_preparation()` as well as `train()` would be run in their own Pod(s) in Kubernetes, while the `train_model()` workflow is a DSL which creates a Directed Acyclic Graph (DAG) of the workflow. It will determine the order of tasks based on their inputs and outputs. Input and output types (based on the type hints) will be validated at registration time to avoid surprises at runtime.

After registration with Flyte, the workflow can be started from the UI:

<img src="/images/dask-flyte-workflow.png" alt="Dask workflow in the Flyte UI" style="max-width: 100%;" width="100%" />

## Why use the `dask` plugin for Flyte?

At a first glance, both Flyte and `dask` look similar in what they are trying to achieve. Both tools can create a DAG out of user functions, take care of connecting in- and outputs, etc. The major conceptual difference is in how both achieve these. While `dask` has long-lived workers to run tasks, a Flyte task is a designated Kubernetes Pod. This creates a significant overhead in task-runtime.

While `dask` tasks have an overhead of around one millisecond ([docs](https://distributed.dask.org/en/stable/efficiency.html#use-larger-tasks)), spinning up a new Kubernetes pod takes on the order of seconds. In addition, due to the long-lived nature of the `dask` workers, `dask` can optimize the DAG to run tasks operating on the same piece of data to run on the same node to avoid the need to serialize the outputs between workers ("shuffling"). Due to the ephemeral nature of Flyte `task`s, this is not possible and task outputs will be serialized to a blob storage instead.

So given all oft the downsides above, why use Flyte? Flyte does not aim at replacing tools such as `dask` or Apache Spark, but rather adds an orchestration layer on top. Sure, while you can run workloads directly in Flyte - think training a model on a single GPU for instance - Flyte offers [a lot of integrations](https://flyte.org/integrations) for other common data processing tools.

In this particular case, a `dask` Flyte task will manage the `dask` cluster lifecycle. When the task is started from the UI, Flyte will first create a dedicated `dask` cluster (made up of Kubernetes pods) which will then be used to execute the user code. This has the advantage of different tasks being able to use different docker images with different dependencies, whilst always ensuring that the dependencies of the client, scheduler and workers are the same.

## What prerequisites are required to run `dask` tasks in Flyte?

- The Kubernets cluster needs to have the [dask operator](https://kubernetes.dask.org/en/latest/operator.html) installed.
- Flyte needs to be `>=1.3.0`
- The `dask` plugin needs to be enabled in the Flyte (propeller) config ([docs](https://docs.flyte.org/en/latest/deployment/plugins/k8s/index.html#specify-plugin-configuration))
- The Python environment within the docker image associated task needs to have `flytekitplugins-dask` installed

## How do things work under the hood?

_Note: The following should only be informative and should not be required for pure users of the plugin, it might be helpful for easier debugging though_

On a birdseye view, the following steps happen when a `dask` task is started from within Flyte:

1. A `FlyteWorkflow` [Custom Resource (CR)](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/) is created in Kubernetes
1. Flyte propeller (a [Kubernetes Operator](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/)) picks up on the workflow creation
1. The operator checks the task's spec, sees that the task is of type `dask`. It checks whether it has a plugin associated with it and will find the installed `dask` plugin
1. The `dask` plugin (within Flyte propeller) picks up on the task defintion and creates a `DaskJob` Custom Resource ([docs](https://kubernetes.dask.org/en/latest/operator_resources.html#daskjob)) using the [dask-k8s-operator-go-client](https://github.com/bstadlbauer/dask-k8s-operator-go-client/)
1. The [dask operator](https://kubernetes.dask.org/en/latest/operator.html) picks up on the `DaskJob` resource and run the job accordingly. It will spin up one pod to run the job (client/job-runner), one pod for the scheduler and as many worker pods as specified in the Flyte task decorator
1. While the `dask` task is running, Flyte (propeller) will continously monitor the created `DaskJob` resource, waiting on it to report success or failure. After the job has finished (or the Flyte task has been terminated), all `dask` related resources will get cleaned up

## Useful links

- [Flyte documentation](https://docs.flyte.org/en/latest/)
- [Flyte community](https://docs.flyte.org/en/latest/community/index.html)
- [flytekitplugins-dask user documentation](https://docs.flyte.org/projects/cookbook/en/latest/auto/integrations/kubernetes/k8s_dask/index.html)
- [flytekitplugins-dask deployment documentation](https://docs.flyte.org/en/latest/deployment/plugins/k8s/index.html)
- [dask-kubernetes documentation](https://kubernetes.dask.org/en/latest/)
- [Blog post on the dask kubernetes operator](http://blog.dask.org/2022/11/09/dask-kubernetes-operator)

In case there are any questions, don't hesitate to reach out. Either in the [Flyte slack](https://flyte-org.slack.com/join/shared_invite/zt-1joarlg5w-cS7sT~pz0oH6PjyITj3mfg#/shared-invite/email) ("Bernhard Stadlbauer") or on [GitHub](https://github.com/bstadlbauer).

I would like to give a shoutout for all of the help I've received from [Jacob Tomlinson](https://jacobtomlinson.dev/) (Dask) as well as [Dan Rammer](https://github.com/hamersaw) (Flyte). This would not have been possible without your support.
