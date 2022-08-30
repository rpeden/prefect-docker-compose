# Prefect with Docker Compose

This repository contains everything you need to run Prefect Orion, a Prefect agent, or the Prefect CLI using Docker Compose. 

# Why Docker Compose?

There are a few reasons you might want to run Prefect using Docker Compose:
* You want to try Prefect without installing anything new on your workstation. 
* You are comfortable with Docker and prefer it over virtual environments or Anaconda. 
* You want to run Prefect on a system like CentOS 7 where installing Prefect dependencies is difficult.

# Prerequisites

* A Linux, MacOS, or Windows computer or VM with Docker installed. If you are running Windows, you must have Docker set up to use Linux containers, not Windows containers.

# Limitations

* If you run a Prefect agent in Docker, it will not be able to run `DockerContainer` deployments because Docker-in-Docker is not supported. 

# Getting Started

Start by cloning this repository.

The `docker-compose.yml` contains four services:
* `database` - Postgres database for the Orion API
* `orion` - Prefect Orion API and UI
* `agent` - Prefect Orion Agent
* `cli` - A container that mounts this repository's `flows` directory and offers an ideal environment for building and applying deployments and running flows. 

## Running locally

### Prefect Orion
To try Prefect locally, open a terminal, navigate to the directory where you cloned this repository, and run:

```
docker-compose --profile orion up
```

This will start PostgreSQL and Prefect Orion. When Orion is ready, you will see a line that looks like:

```
orion_1     | INFO:     Uvicorn running on http://0.0.0.0:4200 (Press CTRL+C to quit)
```

The Orion API container shares port 4200 with the host machine, so if you open a web browser and navigate to `http://localhost:4200` you will see the Prefect Orion UI.

### Prefect CLI

Next, open another terminal in the same directory and run:

```
docker-compose run cli
```

This runs an interactive Bash session in a container that shares a Docker network with the Orion server you just started. If you run `ls`, you will see that the container shares the `flows` subdirectory of the repository:

```
flow.py
root@fb032110b1c1:~/flows#
```

To demonstrate the container is connected to the Orion API you launched earlier, run:

```
python flow.py
```

Then, in a web browser on your host machine, navigate to `http://localhost:4200/runs` and you will see the flow you just ran in your CLI container.

If you'd like to use the CLI container to interact with Prefect Cloud instead of a local Orion instance, update `docker-compose.yml` and change the agent service's `PREFECT_API_URL` environment variable to match your Prefect Cloud API URL. Then, uncomment the `PREFECT_API_KEY` environment variable and replace `YOUR_API_KEY` with your own API key. If you'd prefer not to put your API key in a Docker Compose file, you can also store it in an environment variable on your host machine and pass it through to Docker Compose like so:

```
- PREFECT_API_KEY=${PREFECT_API_KEY}
```

### Prefect Agent

You can run a Prefect Agent by updating `docker-compose.yml` and changing `YOUR_WORK_QUEUE_NAME` to match the name of the Prefect work queue you would like to connect to, and then running the following command:

```
docker-compose --profile agent up
```

This will run a Prefect agent and connect to the work queue you provided. 

As with the CLI, you can also use Docker Compose to run an agent that connects to Prefect Cloud by updating the agent's `PREFECT_API_URL` and `PREFECT_API_KEY` settings in `docker-compose.yml`.



