# n8n Main / Worker / Runner Configuration
This is a docker compose file that is designed to work with the new 2.x version of n8n. I have seen a few online versions that don't handle all of the differences required for this configuration. This is put together so that it helps others understand how all of these parts work together.

## Main Node
This is the container (named: `main` in the compose file) that handles the normal API and web interface for the n8n instance. When you create a workflow, this is the interface for creating those in the web interface.

## Worker Node
This is the container (named: `worker` in the compose file) that handles the actual execution of the workflows. It also acts as the task broker in this type of deployment. Typically the main node would be responsible for this, but when you have a worker node, that is now the node that handles the task broker operations. The runners must register with the worker to ensure they get tasks.

## Runner Node
This is the container (named: `runner` in the compose file) that handles the code execution nodes for Javascript and Python. This node connects to the task broker running on the worker node and awaits for jobs. It is idle unless there is a code execution node. **Note: The current ENV variables do not work on the runner node and you must override them using the config file `custom-n8n-task-runners.json`**

### Setup
Clone this repo into a folder named `n8n`. Run the following:

```bash
chown -R 1000:1000 ./data/n8n
```

This ensures that the `data` directory that n8n main and worker nodes use has the correct ownership permissions once mounted inside of the containers.

### Custom/3rd Party n8n Nodes
There are a bunch of instructions online on how to get those to be active in your n8n instances. I struggled with this a lot. The best option I found was to just use the install process in the n8n web interface. Trying to build the docker containers with the new module installed via npm never worked for me and I eventually gave up. This current implementation seems to work well enough for me.

flowchart TD
    subgraph MainNode["🖥️ Main n8n Node"]
        WH["Webhook / Trigger\nReceiver"]
        API["n8n REST API\n& UI Server"]
        SCH["Scheduler\n& Orchestrator"]
    end

    subgraph Redis["⚡ Redis (Queue Broker)"]
        JQ["Job Queue\n(Bull/BullMQ)"]
        PQ["Progress &\nEvent Channel"]
    end

    subgraph Workers["⚙️ n8n Worker Pool"]
        W1["Worker 1\n(Job Consumer)"]
        W2["Worker 2\n(Job Consumer)"]
        W3["Worker N\n(Job Consumer)"]
    end

    subgraph Runners["🏃 Task Runners"]
        R1["Runner 1\n(JS Sandbox)"]
        R2["Runner 2\n(JS Sandbox)"]
        R3["Runner N\n(JS Sandbox)"]
    end

    subgraph Postgres["🗄️ PostgreSQL Database"]
        WD["Workflow\nDefinitions"]
        EX["Execution\nResults & Logs"]
        CR["Credentials\n& Settings"]
    end

    %% Main → Redis (enqueue jobs)
    SCH -->|"Enqueue workflow\nexecution jobs"| JQ
    WH -->|"Enqueue triggered\nexecution jobs"| JQ

    %% Main ← Redis (progress events)
    PQ -->|"Subscribe to\nprogress events"| API

    %% Workers ← Redis (dequeue)
    JQ -->|"Dequeue &\nclaim job"| W1
    JQ -->|"Dequeue &\nclaim job"| W2
    JQ -->|"Dequeue &\nclaim job"| W3

    %% Workers → Redis (heartbeat/progress)
    W1 -->|"Publish progress\n& heartbeat"| PQ
    W2 -->|"Publish progress\n& heartbeat"| PQ

    %% Workers ↔ Runners (task delegation via HTTP/IPC)
    W1 <-->|"Delegate node\ntasks via HTTP"| R1
    W2 <-->|"Delegate node\ntasks via HTTP"| R2
    W3 <-->|"Delegate node\ntasks via HTTP"| R3

    %% Runners → Workers (results returned)
    R1 -->|"Return task\nresults"| W1
    R2 -->|"Return task\nresults"| W2
    R3 -->|"Return task\nresults"| W3

    %% Workers → Postgres (store results)
    W1 -->|"Store completed\nexecution data"| EX
    W2 -->|"Store completed\nexecution data"| EX
    W3 -->|"Store completed\nexecution data"| EX

    %% Main ← Postgres (read results)
    EX -->|"Read execution\nresults & status"| API
    WD -->|"Load workflow\ndefinitions"| SCH
    CR -->|"Load credentials\nfor execution"| W1
    CR -->|"Load credentials\nfor execution"| W2

    %% Styling
    classDef mainStyle fill:#ea4b71,stroke:#c23d60,color:#fff,font-weight:bold
    classDef redisStyle fill:#dc382c,stroke:#b52e23,color:#fff,font-weight:bold
    classDef workerStyle fill:#f59e0b,stroke:#d97706,color:#fff,font-weight:bold
    classDef runnerStyle fill:#3b82f6,stroke:#2563eb,color:#fff,font-weight:bold
    classDef dbStyle fill:#10b981,stroke:#059669,color:#fff,font-weight:bold

    class MainNode,WH,API,SCH mainStyle
    class Redis,JQ,PQ redisStyle
    class Workers,W1,W2,W3 workerStyle
    class Runners,R1,R2,R3 runnerStyle
    class Postgres,WD,EX,CR dbStyle