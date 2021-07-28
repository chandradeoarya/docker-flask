# Python Flask Docker image

[![type](https://img.shields.io/badge/type-Docker-blue.svg)](https://hub.docker.com/r/devilbox/python-flask)
[![License](https://img.shields.io/badge/license-MIT-%233DA639.svg)](https://opensource.org/licenses/MIT)

This tutorial provides a steps for building and deploying Python Flask demo project.
Code repository - [https://github.com/chandradeoarya/docker-flask](https://github.com/chandradeoarya/docker-flask)

## Docker image tags

**Image name:** `chandradeoarya/hello-flask`

## Environment Variables

| Variable              | Required | Default      | Description            |
| --------------------- | -------- | ------------ | ---------------------- |
| `MYSQL_ROOT_PASSWORD` | Yes      | Helloflask@1 | mysql default password |

## Project directory structure

The following shows how to organize your project on the host operating system.

### Basic structure

The following is the least required directory structure:

```bash
<project-dir>/
└── app.py              # Entrypoint file name can be changed via env var
```

### Structure with dependencies

The following directory structure allows for auto-installing Python dependencies during startup into a virtual env.

```bash
<project-dir>/
├── app                      # Entrypoint dir name can be changed via env var
│   ├── __init__.py
│   └── main.py              # Entrypoint file name can be changed via env var
└── requirements.txt         # Optional: will pip install in virtual env
```

After you've started the container with a `requirements.txt` in place, a new `venv/` directory will be added with you Python virtual env.

```bash
<project-dir>/
├── app
├── __init__.py
├── main.py
│   └── __pycache__
├── requirements.txt
└── venv
    ├── bin
    ├── include
    └── lib
```

### Step-1 Building and running flask app image

For building the images you need to go inside your project directory `hello-flask/` :

```bash
# Clone the repository if not done yet
git clone https://github.com/chandradeoarya/docker-flask.git

# Pull step-1 code
git pull origin master

docker build --tag hello-flask .
```

You can tag image builds based on versions:

```bash
docker tag hello-flask:latest hello-flask:v1
# to remove a tag
docker rmi hello-flask:v1
```

Now you have

```bash
docker tag hello-flask:latest hello-flask:v1
```

Run a container based on this image

```bash
docker run -d --name hello-flask-c --publish 5000:5000 hello-flask
```

## Step-2 Adding mysql database in flask

We will use docker official image for MySQL and run it in a container. Before we run MySQL in a container, we’ll create a couple of volumes that Docker can manage to store our persistent data and configuration.

We are going to use volumes instead of bind mounts.

```bash
# Pull step-2 code
git pull origin step2

# Create docker volume
docker volume create hello-flask-mysql

# Create docker volume for mysql configurations
docker volume create hello-flask-mysql-config
```

Now we’ll create a network that our application and database will use to talk to each other. This will be a user-defined bridge network which gives a nice DNS lookup service which we can use when creating our connection string.

```
docker network create hello-flask-network
```

Now we can run MySQL in a container and attach to the volumes and network we created above. Docker pulls the image from Hub and runs it for you locally. In the following command, option -v is for starting the container with volumes. For more information, see Docker volumes.

```
 docker run --rm -d -v hello-flask-mysql:/var/lib/mysql \
  -v hello-flask-mysql-config:/etc/mysql -p 3306:3306 \
  --network hello-flask-network \
  --name helloflasksqldb \
  -e MYSQL_ROOT_PASSWORD=Helloflask@1 \
  mysql
```

Now, let’s test whether MySQL database is running and we can connect to it. Enter correct password when prompted:

```
docker exec -ti helloflasksqldb mysql -u root -p
```

Build docker image with mysql in flask

```
docker build --tag hello-flask-mysql .

#Run the new flask app with mysql
docker run --rm -d --network hello-flask-network --name hello-flask-mysql-c --publish 5000:5000 hello-flask-mysql

# Db connection test and getting table entries
curl http://localhost:5000
curl http://localhost:5000/initdb
curl http://localhost:5000/widgets

# Insert some data in mysql inventory.widgets table
docker exec -ti helloflasksqldb mysql -u root -p

insert into inventory.widgets (name, description) values ('flask','simple python webapi');
insert into inventory.widgets (name, description) values ('docker','best containerization tool');
insert into inventory.widgets (name, description) values ('mysql','best database service');
```

## Step-3a Using Docker compose

```bash
# Pull step-3 code
git pull origin step3

# Docker compose up
docker-compose -f docker-compose.dev.yml up --build

#checkout the mysql db with same password and insert some records
docker exec -ti hello_sqldb_dc mysql -u root -p
```

## Step-3b Set up CI

This step will be using the same branch. It will use docker hub as container registry and github actions for triggering CI.

Create `DOCKER_HUB_USERNAME` and `DOCKER_HUB_ACCESS_TOKEN` from [Docker hub security](https://hub.docker.com/settings/security) to the github repository secrets.

### Setting github actions workflow

```bash
# This is a basic workflow to help you get started with Actions

name: CI for docker hello_flask_sql

# Controls when the workflow will run
on:
  # Triggers the workflow on push or pull request events but only for the step4 branch
  push:
    branches: [step3]

  # Allows you to run this workflow manually from the Actions tab
  workflow_dispatch:

# A workflow run is made up of one or more jobs that can run sequentially or in parallel
jobs:
  # This workflow contains a single job called "build"
  build:
    # The type of runner that the job will run on
    runs-on: ubuntu-latest

    # Steps represent a sequence of tasks that will be executed as part of the job
    steps:
      # Checks-out your repository under $GITHUB_WORKSPACE, so your job can access it
      - uses: actions/checkout@v2

      - name: Login to Docker Hub
        uses: docker/login-action@v1
        with:
          username: ${{ secrets.DOCKER_HUB_USERNAME }}
          password: ${{ secrets.DOCKER_HUB_ACCESS_TOKEN }}

      # Runs a single command using the runners shell
      - name: Run a one-line script
        run: echo Hello, flasksql!

      # Building
      - name: Set up Docker Buildx
        id: buildx
        uses: docker/setup-buildx-action@v1

      # Pushing
      - name: Build and push
        id: docker_build
        uses: docker/build-push-action@v2
        with:
          context: ./
          file: ./Dockerfile
          push: true
          tags: ${{ secrets.DOCKER_HUB_USERNAME }}/docker-flasksql:latest

      # Runs a set of commands using the runners shell
      - name: Image digest
        run: echo ${{ steps.docker_build.outputs.digest }}
```

### Putting cache layer

Setting up a cache for builder. Adding the path and keys to store this under using GitHub cache.

```
      - name: Cache Docker layers
        uses: actions/cache@v2
        with:
          path: /tmp/.buildx-cache
          key: ${{ runner.os }}-buildx-${{ github.sha }}
          restore-keys: |
            ${{ runner.os }}-buildx-
```

Refer this github cache in future builds. Only changing the build and push step.

```
      - name: Build and push
        id: docker_build
        uses: docker/build-push-action@v2
        with:
          context: ./
          file: ./Dockerfile
          builder: ${{ steps.buildx.outputs.name }}
          push: true
          tags:  ${{ secrets.DOCKER_HUB_USERNAME }}/docker-flasksql:latest
          cache-from: type=local,src=/tmp/.buildx-cache
          cache-to: type=local,dest=/tmp/.buildx-cache
```

### Limiting docker hub push with tags

Changing push condition from branch to tag limits docker hub push to only tagged versions.

```
on:
  push:
    tags:
      - "v*.*.*"
```

Now, future only tagged commits will trigger push to docker hub.

```
git tag -a v1.0.2
git push origin v1.0.2
```

Since only tagged versions are going to docker hub we can store 'latest' tags in github registry. Create `GHCR_TOKEN` in repository secrets from [github personal access token](https://github.com/settings/tokens).

```
# Just take the last docker hub workflow file and change login
- name: Login to Git Hub
  if: github.event_name != 'pull_request'
  uses: docker/login-action@v1
  with:
    registry: ghcr.io
    username: ${{ github.repository_owner }}
    password: ${{ secrets.GHCR_TOKEN }}

# Also update the image tag with github registry
- name: Build and push
  id: docker_build
  uses: docker/build-push-action@v2
  with:
    context: ./
    file: ./Dockerfile
    builder: ${{ steps.buildx.outputs.name }}
    push: true
    tags: ghcr.io/${{ secrets.DOCKER_HUB_USERNAME }}/docker-flasksql:latest
    cache-from: type=local,src=/tmp/.buildx-cache
    cache-to: type=local,dest=/tmp/.buildx-cache
```

## License

**[MIT License](LICENSE)**

No copyright (c) 2021 [chandradeoarya](https://github.com/chandradeoarya)
