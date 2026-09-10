
#  Dockerizing Feast 

## Introduction

In this lab, you will learn how to run a **Feast Feature Store inside a Docker container**. You will start with a simple Feast project, create a Docker environment, build a Docker image, run Feast inside a container, persist Feast data using Docker volumes, and manage the setup with Docker Compose.

By the end of this lab, you will understand how Docker can make a Feast environment portable, reproducible, and easier to manage.


![Dockerizing Feast Deployment](<Dockerizing Feast.jpg>)
## Learning Objectives

By completing this lab, you will be able to:

- Create a Docker environment for a Feast project.
- Build and run a Feast Docker image.
- Persist Feast registry and local store data using Docker volumes.
- Manage a Feast container using Docker Compose.
- Access and verify the Feast UI from a Linux environment.

## Prerequisites

Before starting this lab, make sure you have:

- Basic knowledge of Linux terminal commands.
- Basic understanding of Docker.
- Python 3.9 or later.
- Docker installed and running.
- Basic understanding of Feast.

You should also complete the **Introduction to Feast** lab before starting this one.

## Prologue: The Challenge

A Feast project may work correctly on one machine but become difficult to reproduce on another machine because of differences in Python versions, dependencies, configurations, and system environments.

Docker solves this problem by packaging the Feast environment into a container.

In this lab, you will take a simple Feast project and gradually convert it into a containerized environment.

## Step 1: Prepare the Feast Project

First, create a working directory for the lab and move into it.

```bash
mkdir dockerizing-feast
cd dockerizing-feast
```

Create a basic Feast project:

```bash
feast init feature_repo
cd feature_repo
```

Your project structure should look similar to:

```text
feature_repo/
├── data/
├── feature_store.yaml
├── example.py
└── feature_repo.py
```

Create a Python environment and install Feast if it is not already installed:

```bash
python3 -m venv venv
source venv/bin/activate
pip install feast
```

Now initialize the Feast project:

```bash
feast apply
```

Verify that the project is working:

```bash
feast --help
```

At this point, Feast is working directly on the Linux host.

The goal of the next steps is to run the same environment inside Docker.

## Step 2: Create the Docker Environment

Inside the `feature_repo` directory, create a `requirements.txt` file:

```bash
nano requirements.txt
```

Add:

```text
feast
```

Save the file and create a `Dockerfile`:

```bash
nano Dockerfile
```

Add the following configuration:

```dockerfile
FROM python:3.11-slim

WORKDIR /app/feature_repo

COPY requirements.txt .

RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 6566

CMD ["feast", "ui", "--host", "0.0.0.0", "--port", "6566"]
```

This Dockerfile:

- Uses Python 3.11 as the base image.
- Creates `/app/feature_repo` as the working directory.
- Installs Feast.
- Copies the Feast project into the container.
- Exposes port `6566` for the Feast UI.
- Starts the Feast UI when the container runs.

Build the Docker image:

```bash
docker build -t feast-lab .
```

Verify that the image was created:

```bash
docker images
```

You should see an image named:

```text
feast-lab
```

## Step 3: Build and Run the Feast Container

Now start a container from the image:

```bash
docker run -d \
  --name feast-server \
  -p 6566:6566 \
  feast-lab
```

Check whether the container is running:

```bash
docker ps
```

You should see `feast-server` in the running containers.

Check the container logs:

```bash
docker logs feast-server
```

If Feast starts successfully, you should see output indicating that the Feast UI is running.

You can also check the container from inside:

```bash
docker exec -it feast-server bash
```

Inside the container, verify Feast:

```bash
feast --version
```

Then exit:

```bash
exit
```

This demonstrates an important Docker concept: the Feast environment is now running independently inside a container.

## Step 4: Persist Feast Data with a Docker Volume

Containers are designed to be replaceable. If a container is removed, data stored only inside that container may also disappear.

Feast uses local files such as the registry and online store. Docker volumes allow us to keep these files outside the container lifecycle.

Create a volume:

```bash
docker volume create feast-data
```

Check the volume:

```bash
docker volume ls
```

Stop and remove the previous container:

```bash
docker stop feast-server
docker rm feast-server
```

Run the container again using the volume:

```bash
docker run -d \
  --name feast-server \
  -p 6566:6566 \
  -v feast-data:/app/feature_repo/data/store \
  feast-lab
```

Check the container:

```bash
docker ps
```

The important idea here is that the Feast store data is now stored in the Docker volume rather than being tied only to the container.

This makes the Feast environment more reliable when containers are recreated.

## Step 5: Manage Feast with Docker Compose

Managing long Docker commands manually can become inconvenient.

Docker Compose allows us to describe the container configuration in a YAML file.

Create a Compose file:

```bash
nano docker-compose.yml
```

Add:

```yaml
services:
  feast:
    build: .
    container_name: feast-server
    ports:
      - "6566:6566"
    volumes:
      - feast-data:/app/feature_repo/data/store

volumes:
  feast-data:
```

Start the Feast service:

```bash
docker compose up -d
```

Check the running services:

```bash
docker compose ps
```

View the logs:

```bash
docker compose logs
```

To stop the service:

```bash
docker compose down
```

To start it again:

```bash
docker compose up -d
```

This gives you a simple and reproducible way to manage the containerized Feast environment.

## Step 6: Verify and Access the Feast UI

Finally, verify that the Feast container is running:

```bash
docker ps
```

Check the logs:

```bash
docker compose logs feast
```

The Feast UI should be available on:

```text
http://<server-ip>:6566
```

If you are practicing on a remote Linux lab environment, use the environment's provided load balancer or port-forwarding mechanism to access port `6566`.

You can also test the port from the Linux terminal:

```bash
curl http://localhost:6566
```

If the service is running correctly, you should receive a response from the Feast UI.

## Troubleshooting

### Container is not running

Check:

```bash
docker ps -a
```

Then inspect the logs:

```bash
docker logs feast-server
```

### Port 6566 is already in use

Check which process is using the port:

```bash
sudo lsof -i :6566
```

You can use another host port:

```bash
docker run -d \
  --name feast-server \
  -p 8080:6566 \
  feast-lab
```

Then access:

```text
http://<server-ip>:8080
```

### Docker Compose command not found

Check Docker installation:

```bash
docker --version
docker compose version
```

If Docker Compose is unavailable, verify that Docker Compose support is installed in your Linux environment.

## Self-Assessment

After completing this lab, answer the following questions:

1. Why is Docker useful for running Feast?
2. What is the purpose of the `Dockerfile`?
3. What does `docker build` do?
4. What is the difference between an image and a container?
5. Why do we use a Docker volume with Feast?
6. What is the purpose of `docker-compose.yml`?
7. Why do we expose port `6566`?
8. What happens when a container is removed but its data is stored in a Docker volume?

## Challenge

Modify the Docker Compose configuration so that:

- The Feast service uses a different container name.
- The host port is changed from `6566` to another port.
- The Feast data continues to persist after the container is recreated.

Then verify that the Feast UI is still accessible.

## Conclusion

In this lab, you containerized a Feast Feature Store using Docker.

You started with a Feast project on Linux, created a Docker environment, built a Feast image, ran Feast inside a container, used a Docker volume for persistent data, and finally managed the environment using Docker Compose.

You have now taken an important step toward building **portable and reproducible MLOps environments with Feast and Docker**.

