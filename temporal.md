> The notes in the in this course are a combination of hand-written notes, and notes pasted directly from the course found at https://temporal.talentlms.com/plus/my/training/143
## Setting Up a Development Environment

Let's start by setting up an environment with the Temporal SDK:

1. `python3 -m venv env`
2. `source env/bin/activate`
3. `python -m pip install temporalio`

Now lets setup a local Temporal Service for development with the Temporal CLI.

1. `brew install temporal`
2. `echo "export PATH=\"$(dirname $(which temporal)):\$PATH\"" >> ~/.zshrc && source ~/.zshrc`

Let's run `temporal server start-dev` to start a local Temporal dev env.  This launches:
- the local Temporal Service
- a persistence layer/database
- the Temporal Web UI for observability and debugging

By default:
- the Temporal Service will be available at `localhost:7233`
- the Temporal Web UI will be available at `http://localhost:8233`

## What is Temporal?

In short, Temporal is a platform that guarantees the durable execution of your application code. It allows you to develop as if failures don't even exist. Your application will run reliably even if it encounters problems, such as network outages or server crashes, which would be catastrophic for a typical application. The Temporal platform handles these types of problems, allowing you to focus on the business logic, instead of writing application code to detect and recover from failures.

Temporal applications are built using an abstraction called **Workflows**. You'll develop those Workflows by writing code in a general-purpose programming language such as Go, Java, TypeScript, or Python. The code you write is the same code that will be executed at runtime, so you can use your favorite tools and libraries to develop Temporal Workflows.

Temporal Workflows are resilient. They can run and keeping running for years, even if the underlying infrastructure fails. If the application itself crashes, Temporal will automatically recreate its pre-failure state so it can continue right where it left off.

Conceptually, a workflow defines a sequence of steps. With Temporal, those steps are defined by writing code, known as a **Workflow Definition**, and are carried out by running that code, which results in a **Workflow Execution**.

#### Temporal Architecture

At its core, Temporal is not just an SDK, it's primarily a distributed backend service. Although Temporal includes a Web UI, the core system is not really a traditional web server. The Temporal Service itself is written in Go and communicates primarily over gRPC. The Temporal SDK and the Temporal CLI are clients that make RPC calls to the Temporal Service’s API, as would be a Temporal app you develop with the SDK.

Your Python workers and clients will connect to the Temporal Frontend Service (the Frontend Service acts as an API gateway) over gRPC & Protocol Buffers, while the Web UI lets us inspect workflow state, execution history, retries, task queues, and other runtime details in the browser.

![Temporal Server Architecture](./images/temporal-server-architecture.png)

Network communication to the frontend service can also optionally be secured with TLS to encrypt data in transit and validate certificates.

![Temporal Network Communication](./images/temporal-network-communication.png)

Like the CPU in a computer or the engine in a car, the Temporal Server is an essential part of the overall system, but requires additional components for operation. The complete system is known as the **Temporal Cluster**, which is a deployment of the Temporal Server software on some number of machines, plus the additional components used with it.

The only required component is a database, such as Apache Cassandra, PostgreSQL, or MySQL (SQLite by default for local dev). The Temporal Cluster tracks the current state of every execution of your Workflows. It also maintains a history of all Events that occur during their executions, which it uses to reconstruct the current state in case of failure. It persists this and other information, such as details related to durable timers and queues, to the database.

Elasticsearch is an optional component. It's not necessary for basic operation, but adding it will give you advanced searching, sorting, and filtering capabilities for information about current and recent Workflow Executions. This is helpful when you run Workflows millions of times and need to locate a specific one; for example, based on when it started, how long it took to run, or its final status. Two other tools are often used with Temporal. Prometheus is used to collect metrics from Temporal, while Grafana is used to create dashboards based on those metrics. Together, these tools help operations teams monitor cluster and application health.

![Temporal Cluster Architecture](./images/temporal-cluster-architecture.png)


One thing that people new to Temporal may find surprising is that *the Temporal Cluster does not execute your code*. While the platform guarantees the durable execution of your code, it achieves this through _orchestration_. The execution of your application code is external to the cluster, and in typical deployments, takes place on a separate set of servers, potentially running in a different data center than the Temporal Cluster.

The entity responsible for executing your code is known as a Worker, and it's common to run Workers on multiple servers, since this increases both the scalability and availability of your application. The Worker, which is part of your application, communicates with the Temporal Cluster to manage the execution of your Workflows.

![Temporal Workers](./images/temporal-workers.png)

The application will contain the code used to initialize the Worker, the Workflow, and other functions that comprise your business logic, and possibly also code used to start or check the status of the Workflow. At runtime, you'll need everything needed to execute the application, which will include any libraries or other dependencies referenced in your code, on each machine where at least one Worker process will run. Each machine running a Worker will require connectivity to the frontend service on the Temporal cluster.

#### Options for Running a Temporal Cluster

One option for deploying a self-hosted Temporal Cluster is to use Docker Compose, to run temporal as many containers on one machine. It's extremely convenient for development clusters because it avoids the need to manually install and configure individual components. Temporal maintains a [GitHub repository](https://github.com/temporalio/docker-compose) that offers several configurations for you to use.

Another option for self-hosting, which was described in the is the `temporal` command's built-in support for running a development server. This runs in a single process and doesn't have any external runtime dependencies, so it is less complex and less resource-intensive than using Docker Compose.

Self-hosted Temporal Clusters are often run on Kubernetes (to run temporal on many machines), although this is not required. The documentation provides [more information about cluster deployment](https://docs.temporal.io/cluster-deployment-guide).

The alternative to hosting your own Temporal Cluster is to use Temporal Cloud, a fully-managed cloud service operated and staffed by Temporal. It's a simple, secure, scalable way to power your Temporal applications, providing 99.9% uptime and SOC2 compliance. It also comes with developer and production support from the experts at Temporal.

Using Temporal Cloud frees your organization from the operational workload of running and supporting your own cluster, which involves not only the initial planning and deployment, but ongoing work to monitor, update, and scale it.

Temporal Cloud uses consumption-based pricing, so you only pay for what you use, and you can see your current and past usage at any time right from the web interface.

## Developing a Workflow

In Temporal, you define a Workflow in Python by creating a class. The code that makes up that class is known as the **Workflow Definition**.

`greeting.py`:
```python
class GreetSomeone:
    async def run(self, name: str) -> str:
        return f"Hello {name}!"
```

> The Temporal Python SDK uses the native `asyncio` functionality in the Python Standard Library, so we will define our method as an `async` method. Using non-async methods is supported, but require a more complex multi-process setup. For this reason we recommend using `async`. 

Let's now write a program to invoke this workflow:

`local_runner.py`:
```python
import sys
import asyncio
from workflow import GreetSomeone


async def main():
    name = sys.argv[1]
    greeter = GreetSomeone()
    greeting = await greeter.run(name)
    print(greeting)

if __name__ == "__main__":
    asyncio.run(main())
```

Now if we run `python local_runner.py Tyler` we get output `Hello Tyler!`.

On its own though, this is not yet a workflow definition, nothing from Temporal is at play here yet, so let's work towards that. 

Temporal doesn't impose any rules about how to name your Workflow method, so there's no need to rename the class we already wrote. Every Workflow has a name, which Temporal refers to as the **Workflow Type**. It's perhaps a confusing term because "type" can mean so many things, but you can think of it like a "type" in a programming language. In the Python SDK, by default, a Workflow's type is the name of the class used to define that Workflow, but it is possible to override it and provide a more user-friendly value since the Web UI displays Workflow Executions by their type.

Turning this method into a Temporal Workflow Definition requires just three steps:
- Import the `workflow` module from the Temporal Python SDK
- Decorate the class that will contain your Workflow Definition with the `@workflow.defn` decorator
- Decorate the method that defines your Workflow Definition with the `@workflow.run` decorator

`workflow.py`:
```python
from temporalio import workflow


@workflow.defn
class GreetSomeone:
    @workflow.run
    async def run(self, name: str) -> str:
        return f"Hello {name}!"
```


#### Input Parameters & Return Values

###### Values must be Serializable

In order for Temporal to store the Workflow's input and output, data used in input parameters and return values must be serializable. By default, Temporal can handle null or binary values, as well as any data that can be serialized into JSON. This means that most of the types you'd typically use in a function, such as integers and floating point numbers, boolean values, and strings, are all handled automatically, as are [`dataclasses`](https://docs.python.org/3/library/dataclasses.html) composed from these types, but types such as `datetime`, functions, or other non-serializable data types are prohibited as either input parameters or return values.

###### Data Confidentiality

Although the input parameters and return values are stored as part of the Event History of your Workflow Executions, you can create a custom Data Converter to encrypt the data as it enters the Temporal Cluster and decrypt it upon exit, thereby maintaining the confidentiality of any sensitive data used as input or output of your applications. Custom data converters are beyond the scope of the Temporal 101 course, but the [documentation provides more information](https://docs.temporal.io/concepts/what-is-a-data-converter/) and you can view [samples on GitHub](https://github.com/temporalio/samples-python/tree/main/encryption).

###### Avoid Passing Large Amounts of Data

Because the Event History contains the input and output, which is also sent across the network from the application to the Temporal Cluster, you'll have better performance if you limit the amount of data sent. For example, imagine you've created a Workflow that will convert audio files from one format to another. It would be much better to pass the path or URL for the files as input than to pass the _content_ of the files.

To protect against unexpected failures caused by sending or storing too much data, the Temporal Server imposes various limits beyond which it will emit warnings or errors, depending on the severity. The documentation includes a page that [details these limits](https://docs.temporal.io/kb/temporal-platform-limits-sheet).

#### Initializing the Worker

As mentioned earlier, Workers execute your Workflow code. The Worker itself is provided by the Temporal SDK, but your application will include code to configure and run it. When that code executes, the Worker establishes a persistent connection to the Temporal Cluster and begins polling a Task Queue on the Cluster, seeking work to perform. Since Workers execute your code, any Workflows you execute will make no progress unless one Worker is running.

In our current code, the `@workflow.defn` and `@workflow.run` annotations are effectively doing nothing because we never start the workflow through the Temporal client or run it inside a Temporal worker. That means no Temporal server, workflow history, replay, task queue, durability, or orchestration is involved, it is just regular Python execution with unused Temporal decorators attached. You can still initiate and persist a real workflow in Temporal without a worker running, but the workflow cannot make any progress until a worker connects to process tasks

There are typically three things you need in order to configure a Worker:
1. A Temporal Client: used to communicate with the Temporal Cluster
2. A Task Queue (more specifically, the name of a task queue): which is maintained by the Temporal Server and polled by the Worker
3. The Workflow Definition class: used to register the Workflow implementation with the Worker

`starter.py`:
```python
import asyncio

from temporalio.client import Client
from temporalio.worker import Worker

from greeting import GreetSomeone


async def main():
    client = await Client.connect("localhost:7233", namespace="default")
    # Run the worker
    worker = Worker(client, task_queue="greeting-tasks", workflows=[GreetSomeone])
    await worker.run()


if __name__ == "__main__":
    asyncio.run(main())
```

> The lifetime of the Worker and the duration of a Workflow Execution are unrelated. The `worker.run()` function used to start this Worker is a blocking function that doesn't stop unless it is shut down or encounters a fatal error. The Worker's process may last for days, weeks, or longer. If the Workflows it handles are relatively short, then a single Worker might execute thousands or even millions of them during its lifetime. On the other hand, a Workflow can run for years, while the server where a Worker process is running might be rebooted after a few months by an administrator doing maintenance. If the Workflow Type was registered with other workers, one or more of them will automatically continue where the original Worker left off. If there are no other Workers available, then the Workflow Execution will continue where it left off as soon as the original Worker is restarted. In either case, the downtime will not cause Workflow Execution to fail.

## Executing a Workflow

One way to start a Workflow is by using the `temporal` CLI. The `temporal workflow start` command specifies several arguments:
- Workflow Type: In the Python SDK, this defaults to the name of the class you specified as your Workflow definition with the `@workflow.defn` decorator.
- Task Queue Name: The task queue name the Temporal Cluster will use, which must exactly match the value supplied when initializing the Worker. Since task queues are dynamically created, typing the task queue name incorrectly would not cause an error, but it would result in two different task queues, and since the Cluster and Worker wouldn't share the same queue in this case, the Workflow Execution would never progress.
- Workflow ID: An optional, user-defined identifier, which typically has some business meaning. If omitted, a UUID will be automatically assigned as the Workflow ID.
- Input: Since this Workflow requires input, we should supply that value. When submitting a Workflow for execution through the command line, the input is always in JSON format, which is why the input in this command shows double quotes inside of single quotes. Typing JSON directly on the command line is fine for a simple case like this, where there's just one parameter and a single value, but it would be a clumsy way of passing more complex data. Luckily, you can save the input to a file, in JSON format, and specify its path to the `--input-file` option, rather than using the `--input` option to specify the data inline, as shown here.

```
temporal workflow start \
    --type GreetSomeone \
    --task-queue greeting-tasks \
    --workflow-id my-first-workflow \
    --input '"Tyler"'
```

When you run the command, it submits your execution request to the cluster, which responds with the Workflow ID, which will be the same as the one you provided, or assigned UUID if you omitted it. It also displays a Run ID, which uniquely identifies this _specific execution_ of the Workflow. However, it does not display the result returned by the Workflow, since Workflows might run for months or years. You can use the `temporal workflow show --workflow-id <WorkflowID>` command to retrieve the result.

In the `hellow-workflow` exercise with code:

`exercises/hello-workflow/solution/worker.py`:
```python
import asyncio

from temporalio.client import Client
from temporalio.worker import Worker

from greeting import GreetSomeone


async def main():
    client = await Client.connect("localhost:7233", namespace="default")
    # Run the worker
    worker = Worker(
        client,
        task_queue="greeting-tasks",
        workflows=[GreetSomeone],
    )
    print("Starting worker...")
    await worker.run()


if __name__ == "__main__":
    asyncio.run(main())
```

`exercises/hello-workflow/solution/greeting.py`:
```python
from temporalio import workflow


@workflow.defn
class GreetSomeone:
    @workflow.run
    async def run(self, name: str) -> str:
        return f"Hello {name}!"
```

In `exercises/hello-workflow/practice`, we can run the worker with `python worker.py`, then run the workflow with `temporal workflow start` with parameters above. Now if we do  `temporal workflow show -w my-first-workflow` we get output:

```sh
Progress:
  ID           Time                     Type           
    1  2026-05-11T22:11:58Z  WorkflowExecutionStarted  
    2  2026-05-11T22:11:58Z  WorkflowTaskScheduled     
    3  2026-05-11T22:11:58Z  WorkflowTaskStarted       
    4  2026-05-11T22:11:58Z  WorkflowTaskCompleted     
    5  2026-05-11T22:11:58Z  WorkflowExecutionCompleted

Results:
  Status          COMPLETED
  Result          "Hello Tyler!"
  ResultEncoding  json/plain
```

Instead of the CLI, we could have also started the workflow programtically:

`programatically_start_workflow.py`:
```python
import asyncio
import sys

from greeting import GreetSomeone
from temporalio.client import Client


async def main():
    # Create client connected to server at the given address
    client = await Client.connect("localhost:7233")

    # Execute a workflow
    handle = await client.start_workflow(
        GreetSomeone.run,
        sys.argv[1],
        id="greeting-workflow",
        task_queue="greeting-tasks",
    )

    print(f"Started workflow. Workflow ID: {handle.id}, RunID {handle.result_run_id}")

    result = await handle.result()

    print(f"Result: {result}")


if __name__ == "__main__":
    asyncio.run(main())
```