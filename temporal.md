
### What is Temporal?

Temporal is a **durable workflow orchestration platform** for building reliable distributed systems.

  

### Setting Up a Development Environment

Let's start by setting up an environment with the Temporal SDK:

1. `python3 -m venv env`
2. `source env/bin/activate`
3. `python -m pip install temporalio`

Now lets setup a local Temporal Service for development with the Temporal CLI.

1. `brew install temporal`
2. `echo "export PATH=\"$(dirname $(which temporal)):\$PATH\"" >> ~/.zshrc && source ~/.zshrc`

At its core, Temporal is not just an SDK, it's primarily a distributed backend service. Although Temporal includes a Web UI, the core system is not really a traditional web server. The Temporal Service itself is written in Go and communicates primarily over gRPC. The Temporal SDK and the Temporal CLI are clients that make RPC calls to the Temporal Service’s API. The service also includes a full persistence layer for durable workflow execution, using SQLite by default in local development mode.

Let's run `temporal server start-dev` to start a local Temporal dev env.  This launches:
- the local Temporal Service
- a persistence layer/database
- the Temporal Web UI for observability and debugging

By default:
- the Temporal Service will be available at `localhost:7233`
- the Temporal Web UI will be available at `http://localhost:8233`

Your Python workers and clients will connect to the Temporal Service over gRPC, while the Web UI lets us inspect workflow state, execution history, retries, task queues, and other runtime details in the browser.


