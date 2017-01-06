#  Hello Docker
![docker-logo](images/what_is_docker.jpg)


!SUB
### VM vs Containers
![contenaterVsVM](images/07_vm-compare.jpg)


!SUB
### Docker daemon architecture
![architecture](images/docker-execdriver-diagram.png)


!SUB
![architecture](images/Docker-API-infographic-container-devops-nordic-apis.png)


!SUB
## Docker stages
![stages](images/docker-stages.png)

!SUB
### The Life of a Container by example.

```
 docker pull mongo:latest     # pull the mongo image from the registry
 docker inspect mongo:latest  # list information of the container
 docker run -p 27017:27017 \
        --name my-mongo -d \  # create and start a mongo container
        mongo:latest          
 docker inspect my-mongo      # inspect the running container info
 docker logs -f my-mongo      # tail the log of the container
 docker stop my-mongo         # stop the container
 docker rm -v my-mongo        # remove the container
 docker rmi mongo:latest      # remove the image from the local repo
```

!Note
Inzoomen met alt+click

!SUB
### Information of a container.

| command      | description           |
| ------------ |---------------|
| ps |shows running containers.|
| logs |gets logs from container.|
| inspect |looks at all the info on a container (including IP address).|
| events |gets events from container.|
| port |shows public facing port of container.|
| top |shows running processes in container.|
| stats |shows container's resource usage statistics.|

!SUB
## Dockerfile

Each Dockerfile is a script, composed of various commands and arguments listed successively to automatically perform actions on a base image in order to create a new one. They are used for organizing things and greatly help with deployments by simplifying the process start-to-finish.

!SUB
## Dockerfile by example
```
FROM java:8-jre
MAINTAINER Sebastiaan Renkens <srenkens@gmail.com>

ADD some-awesome-application.jar /some-awesome-application.jar

EXPOSE 8080

CMD ["java","-jar","/some-awesome-application.jar"]

```