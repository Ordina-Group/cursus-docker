#First simple use case. 
*Building maven projects with the help of a Docker container*

![logo](images/maven-logo.png)


### Maven
Maven is a build automation tool used primarily for Java projects. 

In this chapter we are going to build an example Maven poject without having to install Java and Maven on our own machine. Instead we will use a image from the DockerHUB for this.


### Maven basic commands

- We have made a simple maven "Hello-World" project available in our GIT repo in `/mnt/sda1/cursus-docker/basics-maven`.
- We do not have Java and Maven installed on our VM. And we are NOT going to install it. It's more fun to use docker for that.
- Execute these commands:

```
# Go to the "hello-world" project dir
cd /mnt/sda1/cursus-docker/basics-maven/hello-world

# Pull maven image
docker pull maven:3-jdk-8

# Start the build in a maven container 
docker run -it --name my-maven-project -v "$PWD":/usr/src/mymaven -w /usr/src/mymaven maven:3-jdk-8 mvn clean install

# Note that the previous run command did not remove the 
# container (--rm option is not given). This allows you 
# to start the container again. Since it has build up a 
# local maven repository the build will be much faster this time.
docker start -a my-maven-project
```


### DockerHUB: Start exploring ###
As you have seen, you could fairly easy start using Apache Maven without having to install it. This applies to MANY, MANY other tools and applications that are available on the DockerHub. 

If you even want to give something a spin, make sure you check the DockerHUB. There might be an image available that runs out of the box and that saves you a lot of your time.

https://hub.docker.com/

https://hub.docker.com/r/_/maven/