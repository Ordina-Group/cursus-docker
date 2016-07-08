# The Docker ABC
*An introduction to docker*

![logo](images/docker-logo-med.png)


### Docker
The next slides briefly explain the docker concepts and commands that you need to understand today.


### Prerequisites
If you have closed the Docker Quickstart Terminal, open it again.

Here is what we've done there so far:

```
# ask for the ip address
$ docker-machine ip workshop

# start a ssh session
$ docker-machine ssh workshop

# clone our git repository
docker@workshop:~$ git clone https://github.com/OrdinaNederland/cursus-docker

# move it to a persistent location
docker@workshop:~$ sudo mv cursus-docker /mnt/sda1/
```


### Docker command examples
*Below some examples of docker **`image`** commands*

```
Commands:
    images    List images
    inspect   Return low-level information on a container or image
    ps        List containers
    pull      Pull an image or a repository from a Docker registry
    run       Run a command in a new container

Run 'docker help' for all commands.
Run 'docker COMMAND --help' for more information on a command.
```

`DIY`: Try for yourself 
```
$ docker help
```


### Running a web server using docker (1/4)
- In the next slide you will start a docker container that runs the nginx static content server.
- The server is running on port 80 in the container, you can find the exposed port using `docker inspect`. This port is mapped to port 8080 on your docker-machine.


### Running a web server using docker (2/4)
`DIY`: In the terminal session
```
# First pull the nginx from the DockerHUB:
$ docker pull nginx

# Next check your list of local images
$ docker images
# If this list would be too large, then you can filter by image name
$ docker images nginx

# Inspect the nginx image
$ docker inspect nginx

# Create and start a container (instance of your image)
$ docker run -d -p 8080:80 --name mywebserver nginx

# Inspect the running docker processes
$ docker ps
```
- Try to access your webserver using your browser<br>
`http://< your-docker-machine-ip >:8080/`


### Running a web server using docker (3/4)
*Below are some examples of other docker **`container`** commands*
```
Commands:
    logs      Fetch the logs of a container
    rm        Remove one or more containers
    start     Start a stopped container
    stop      Stop a running container

Run 'docker help' for all commands.
Run 'docker COMMAND --help' for more information on a command.
```


### Running a web server using docker (4/4)
`DIY`: In the terminal session
```
# Stop the container
$ docker stop mywebserver

# Inspect the processes
$ docker ps
$ docker ps -a

# Start the container
$ docker start mywebserver

# Inspect the logging
$ docker logs mywebserver

# Stop and remove the container
$ docker stop mywebserver 
$ docker rm -v mywebserver
```


### Building a web server with custom content (1/3)

Next we build a Docker image ourselves. It is based on the nginx image and a very simple index.html as static content for our server and a `Dockerfile`, the build file for docker.
  
```
# Navigate to /mnt/sda1/cursus-docker/basics-docker/nginx directory
$ cd /mnt/sda1/cursus-docker/basics-docker/nginx

# inspect the files
$ ls .
$ cat in-container/index.html
$ cat Dockerfile

# Build your own docker image labeled 'starter/docker'. It uses the Dockerfile 
# and replaces the index.html of the base nginx image with your own
$ docker build -t starter/docker .

# Check that your new image exists in the list of available images
$ docker images
```


### Building a web server with custom content (2/3)
- Start your new container in the foreground
  - `-d  ` option is missing, therefore daemon mode is not enabled)
  - `--rm` option will remove the container once it shuts down.
   ```
   $ docker run -p 8080:80 --rm starter/docker
   ```
- Try to access your webserver using your browser<br>
`http://< your-docker-machine-ip >:8080/`
- You might need a Ctrl-F5
- Verify that your new ```index.html``` is in place
- Once done hit Ctrl-C (in the terminal) to stop the container


### Running a web server with custom content (3/3)
In the previous example we added our static content to the image with a docker build. Alternatively we can add the content via a mount. In the next example we mount the local file system to a mount point in the container.

```
# **********************************************************
# * Also run in the basics-docker/nginx directory.  
# **********************************************************
# The variable ${PWD} refers to the current working dir.
# Hint, try refreshing your browser after this.  
$ cd /mnt/sda1/cursus-docker/basics-docker/nginx
$ docker run --rm -p 8080:80 -v ${PWD}/via-mount:/usr/share/nginx/html nginx
```
- Once done hit Ctrl-C (in the terminal) to stop the container


### Container linking
- Docker has a linking system that allows you to link multiple containers together and send connection information from one to another.
- When containers are linked, information about a source container can be sent to a recipient container.
- To establish links, Docker relies on the names of your containers.
----
** For example (do _NOT_ execute this) **
- First we create a container for our database.
   ```
   docker run -d --name postgres <image>
   ```
- Secondly we link our database to our web container
   ```
   docker run -d --link postgres:db --name web <image>
   ```


### Docker Compose
Compose is a tool for defining and running complex applications with Docker. 
With Compose, you define a multi-container application in a single file, then spin up your application using a single command. 
Everything needed is started automatically.
<iframe src="https://www.youtube.com/embed/WgQWndFPMpg" frameborder="0" allowfullscreen width="530" height="396"></iframe>


### Docker Compose - Installation
In the next slides you'll roughly repeat what you saw in the previous demo video, made by a member of the Docker team.

By default docker-compose is not available inside a machine provided by docker-machine, so you need to install it: 

```
# Navigate to the basics-docker/compose directory.  
$ cd /mnt/sda1/cursus-docker/basics-docker/compose
$ sudo ./install-compose.sh
```


### Docker Compose
With Compose we create two containers that are linked.
- The first container contains a python application that serves a simple web page with a counter. This container will be built by Compose.
- The second container contains a Redis key value store, to store the counter.
   ```
   # Navigate to the basics-docker/compose directory.  
   $ cd /mnt/sda1/cursus-docker/basics-docker/compose
   
   # Inspect the compose file note the container linking
   $ cat docker-compose.yml
   
   # start and build the containers.
   $ docker-compose up
   ```
- Inspect with `curl` or a browser the web app is working.
- Hint: port * **is not** * 8080
- Use Ctrl-C to stop the container


### Clean up
You can remove docker containers by executing the command `docker rm`. 
The switch `-v` will remove all implicit mounted volumes and the switch `-f` will remove running containers as well.  
Using the command for the `-f` switch, `$(docker ps -q -a)` will list ids of all docker containers.

```
# List all docker containers
$ docker ps -a

# List ids only of the docker containers
$ docker ps -q -a

# Remove docker containers using their ids
$ docker rm -v -f $(docker ps -q -a)
```