# Prefect 2 with Docker Compose

This repository contains everything you need to run Prefect Orion, a Prefect agent, or the Prefect CLI using Docker Compose. 

Thanks to Paco Ibañez for his excellent work in [this repository](https://github.com/fraibacas/prefect-orion), which helped me get started. I don't intend to replace his work. Instead, my repository aims to use Docker Compose to help you experiment with Prefect 2 locally and explore the different services you may need in a production environment.

# Why Docker Compose?

There are a few reasons you might want to run Prefect using Docker Compose:
* You want to try Prefect without installing anything new on your workstation. 
* You are comfortable with Docker and prefer it over virtual environments or Anaconda. 
* You want to run Prefect on a system like CentOS 7 where installing Prefect dependencies is difficult.

# Prerequisites

* A Linux, MacOS, or Windows computer or VM with Docker installed. If you are running Windows, you must have Docker set up to use Linux containers, not Windows containers.

# Limitations

* If you run a Prefect agent in Docker, it will not be able to run `DockerContainer` deployments unless you share the host's Docker socket with the agent container because Docker-in-Docker is not supported. 

# Getting Started

Start by cloning this repository.

The `docker-compose.yml` file contains five services:
* `database` - Postgres database for the Orion API
* `minio` - MinIO S3-compatible object store, useful for experimenting with remote file storage without needing a cloud storage account.
* `orion` - Prefect Orion API and UI
* `agent` - Prefect Orion Agent
* `cli` - A container that mounts this repository's `flows` directory and offers an ideal environment for building and applying deployments and running flows. 

## Prefect Server
To run the Prefect Orion API and UI, open a terminal, navigate to the directory where you cloned this repository, and run:

```
docker-compose --profile server up
```

This will start PostgreSQL and Prefect Server. When the serveris ready, you will see a line that looks like:

```
server_1     | INFO:     Uvicorn running on http://0.0.0.0:4200 (Press CTRL+C to quit)
```

The Prefect Server container shares port 4200 with the host machine, so if you open a web browser and navigate to `http://localhost:4200` you will see the Prefect Orion UI.

## Prefect CLI

Next, open another terminal in the same directory and run:

```
docker-compose run cli
```

This runs an interactive Bash session in a container that shares a Docker network with the Orion server you just started. If you run `ls`, you will see that the container shares the `flows` subdirectory of the repository on the host machine:

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

## Prefect Agent

You can run a Prefect Agent by updating `docker-compose.yml` and changing `YOUR_WORK_QUEUE_NAME` to match the name of the Prefect work queue you would like to connect to, and then running the following command:

```
docker-compose --profile agent up
```

This will run a Prefect agent and connect to the work queue you provided. 

As with the CLI, you can also use Docker Compose to run an agent that connects to Prefect Cloud by updating the agent's `PREFECT_API_URL` and `PREFECT_API_KEY` settings in `docker-compose.yml`.

## MinIO Storage

MinIO is an S3-compatible object store that works perfectly as remote storage for Prefect deployments. You can run it inside your corporate network and use it as a private, secure object store, or just run it locally in Docker Compose and use it for testing and experimenting with Prefect deployments. 

If you'd like to use MinIO with Prefect in Docker compose, start them both at once by running:

```
docker compose --profile orion --profile minio up
```

Although Orion won't need to talk to MinIO, Prefect agents and the Prefect CLI will need to talk to both MinIO _and_ Orion to create and run depoyments, so it's best to start them simultaneously.

After the MinIO container starts, you can load the MinIO UI in your web browser by navigating to `http://localhost:9000`. Sign in by entering `minioadmin` as both the username and password. 

Create a bucket named `prefect-flows` to store your Prefect flows, and then click **Identity->Service Accounts** to create a service account. This will give you an access key and a secret you can enter in a Prefect block to let the Prefect CLI and agents write to and read from your MinIO storage bucket.

After you create a MinIO service account, open the Prefect UI at `http://localhost:4200`. Click **Blocks**, then add a **Remote File System** block. Give the block any name you'd like, but remember what name you choose because you will need it when creating a deployment. 

In the *Basepath* field, enter `s3://prefect-flows`.

Finally, the *Settings* JSON field should look like this:

```
{
  "key": "YOUR_MINIO_KEY",
  "secret": "YOUR_MINIO_SECRET",
  "client_kwargs": {
    "endpoint_url": "http://minio:9000"
  }
}
```
Replace the placeholders with the key and secret MinIO generated when you created the service account. You are now ready to deploy a flow to a MinIO storage bucket! If you want to try it, open a new terminal and run:

```
docker compose run cli
```

Then, when the CLI container starts and gives you a Bash prompt, run:

```
prefect deployment build -sb "remote-file-system/your-storage-block-name" -n "Awesome MinIO deployment" -q "awesome" "flow.py:greetings"
```

Now, if you open `http://localhost:9001/buckets/prefect-flows/browse` in a web browser, you will see that flow.py has been copied into your MinIO bucket.

## Next Steps

You can run as many profiles as once as you'd like. For example, if you have created a deployment and want to start and agent for it, but don't want to open two separate terminals to run Prefect Server, an agent, *and* MinIO you can start them all at once by running: 

```
docker compose --profile server --profile minio --profile agent up
```

And if you want to start two separate agents that pull from different work queues? No problem! Just duplicate the agent service, give it a different name, and set its work queue name. For example:

```
agent_two:
    image: prefecthq/prefect:2.3.0-python3.10
    restart: always
    entrypoint: ["prefect", "agent", "start", "-q", "YOUR_OTHER_WORK_QUEUE_NAME"]
    environment:
      - PREFECT_API_URL=http://server:4200/api
    profiles: ["agent"]
```
Now, when you run `docker-compose --profile agent up`, both agents will start, connect to the Prefect Server API, and begin polling their work queues.
