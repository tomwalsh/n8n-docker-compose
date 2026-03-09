# n8n Main / Worker / Runner Configuration
This is a docker compose file that is designed to work with the new 2.x version of n8n. I have seen a few online versions that don't handle all of the differences required for this configuration. This is put together so that it helps others understand how all of these parts work together.

## Main Node
This is the container (named: `main` in the compose file) that handles the normal API and web interface for the n8n instance. When you create a workflow, this is the container handling the UI for creating those.

## Worker Node
This is the container (named: `worker` in the compose file) that handles the actual execution of the workflows. It also acts as the task broker in this type of deployment. Typically the main node would be responsible for this, but when you have a worker node, that is now the node that handles the task broker operations. The runners must register with the worker to ensure they get tasks.

## Runner Node
This is the container (named: `runner` in the compose file) that handles the code execution nodes for Javascript and Python. This node connects to the task broker running on the worker node and awaits for jobs. It is idle unless there is a code execution node. **Note: The current ENV variables do not work on the runner node and you must override them using the config file `custom-n8n-task-runners.json`**

The current version of the `custom-n8n-task-runner.json` file allows you to run all builtin node functions, as well as the external library "moment". Python allows for all starndard lib modules (datetime, re, etc.) If you want to change these, edit the file and then restart the runner `docker compose restart runner`.

### Operation
The compose starts up postgresql and redis first and waits for those two services to be ready via healthchecks. The system then starts the `main` instance and waits for that to be ready. The system then starts the `worker`, and then the `runner`. The `runner` will keep trying to connect to the `worker` until it is accepted. I tried to implement a healthcheck on the `worker` as well, but could never get it to show as healthy. Not sure why but I assume it has to do with being a worker instead of a main node.

### Setup
Clone this repo into a folder named `n8n`. Run the following:

```bash
chown -R 1000:1000 ./data/n8n
```

This ensures that the `data` directory that n8n main and worker nodes use has the correct ownership permissions once mounted inside of the containers.

Edit the `.env` file and update the following values:
```
POSTGRES_PASSWORD
N8N_ENCRYPTION_KEY
N8N_RUNNERS_AUTH_TOKEN
TIMEZONE
```
You can run the following to generate a random string for the AUTH_TOKEN and ENCRYPTION_KEY:

```
openssl rand -base64 32
```

When you are ready start the docker compose:

```
docker compose up -d
```

You can drop the `-d` if you want to run it in the foreground to see if there are any errors.

Then connect to the n8n instance on http://&lt;ip address&gt;:5678/

### Custom/3rd Party n8n Nodes
There are a bunch of instructions online on how to get those to be active in your n8n instances. I struggled with this a lot. The best option I found was to just use the install process in the n8n web interface. Trying to build the docker containers with the new module installed via npm never worked for me and I eventually gave up. This current implementation seems to work well enough.

## Basic Layout
```
                                +---------+
                                |  main   |
                                | UI/API  |
                                +---------+
                                  /     \
                                 /       \
                         +---------+   +---------+
                         | Postgres|   |  Redis  |
                         +---------+   +---------+
                                 \       /
                                  \     /
                               +-----------+ 
                               |   worker  |
                               |  workflow |
                               |   broker  | 
                               +-----------+ 
                                     |
                           code execution (JS/Py)
                                     |
                               +-----------+
                               |  runner   |
                               +-----------+
```