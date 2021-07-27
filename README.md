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
git pull origin step-2

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

## Step-3 Using Docker compose

```bash
# Pull step-3 code
git pull origin step-3

# Docker compose up
docker-compose -f docker-compose.dev.yml up --build
```

## License

**[MIT License](LICENSE)**

Copyright (c) 2021 [chandradeoarya(https://github.com/chandradeoarya)]
