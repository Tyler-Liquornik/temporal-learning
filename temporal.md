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


