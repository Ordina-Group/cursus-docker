## Continuous Delivery using Docker
*Embedding Docker into your Continuous Integration*

![jenkins-docker](images/docker-jenkins.png)


## Some internals
- You will build a Continuous Delivery setup for a simple REST service based on Spring Boot
- The REST service is a service which provides "Sticky Notes" functionality
- Data of the Sticky Notes REST service will be stored in a MongoDB
- The Sticky Notes REST service is built using Gradle
- We build a fat jar and distribute that jar using Docker controlled by Jenkins


## Container overview
When you are done, you will have these containers running. Some will be maniputed by Jenkins and some by using docker-compose.
 
![pipeline](images/container-overview-ci-all.png)


## The Pipeline
These are the jobs that you will configure in Jenkins.
![pipeline](images/pipeline-complete.png)


# ..But first... 
#...some preparations...


## Windows and less then 20GB on the C disk ?? (1/3)
### You're screwed... but there is a solution
We need to move the docker-machine data to a disk with more free space. Luckily you have the D drive for that!
- Exit from the SSH session of the `workshop` machine
    ```
docker@workshop:~$ exit
    ```
- Stop the workshop machine
    ```
docker-machine stop workshop
    ```
- Do the same for other running docker-machines (you can list them with docker-machine ls)
- Continue on next slide


## Windows and less then 20GB on the C disk  (2/3)
- Using Windows explorer, create a folder on a disk that does have more then 20GB available (e.g. `D:\docker-machines`)
- Using Windows explorer, move the folder `C:\Users\<username>\.docker\machine` to your created directory.
- Open a Command prompt with administrator rights 
  - (startmenu, type `cmd`, right-click `cmd`, run as administrator)
- Create a symbolic link on the original location and point it to the location where your data is stored
```
mklink /D  c:\Users\<username>\.docker\machine d:\docker-machines\machine
```


## Windows and less then 20GB on the C disk  (3/3)
- Return to your quickstart terminal
- Start the workshop machine
    ```
docker-machine start workshop
    ```
- SSH back into the workshop
    ```
docker-machine ssh workshop
    ```
- Reinstall docker-compose
    ```
# Navigate to the basics-docker/ directory.  
$ cd /mnt/sda1/cursus-docker/basics-docker/compose
$ sudo ./install-compose.sh
    ```
- Done


# Let's get started


## Setting up Git server
- We will setup a GitLAB as a GIT server. 
- This GitLAB server will be used as a server to host our REPO to Jenkins. 
- You can use GitLAB to check the sources and (if you even want to) make changes to the sources.


### Setting up GitLab (docker-compose)	
- The containers that we need for our Continuous Delivery environment will be managed by Docker Compose. Later we will add more.
- You will find a `docker-compose.yml` file in `/mnt/sda1/cursus-docker/continuous`. For now this file only contains one container definition for GitLab, our Git server.
- (Setup instructions on the next slide)

```json
git:
  image: gitlab/gitlab-ce:8.0.4-ce.1
  volumes:
    - ./.data/gitlab/config:/etc/gitlab
    - ./.data/gitlab/logs:/var/log/gitlab
    - ./.data/gitlab/data:/var/opt/gitlab
  ports:
    - "2222:22"
    - "80:80"
```


### Setting up GitLab - Start
- To test our compose file we will
 - Start the GitLAB container by using the `docker-compose` tool
 - Check the logs of GitLAB container
- Execute the following commands:

```
# navigate to /mnt/sda1/cursus-docker/continuous
docker-compose up -d

# to inspect the logs
docker-compose logs -f git

# Wait until gitlab server has started
# Look for 'The server is now ready to accept connections' (part of the redis logging)
# Use ctrl-c to quit the logging
```
You now have a running GitLAB server


### Setting up GitLab - Configure
Let's perform the basic configuration of GitLAB 

- Start a webbrowser and browse to: `http://<< docker-machine-ip >>/`
- Login with the default user and password.
  - username: `root`
  - password: `5iveL!fe`
- Next you have to set a new password
 - If you wanna be lazy.. you can change it to the original `5iveL!fe` password
- After setting the password login again.


### Setting up GitLab - Configure
We will clone a remote repo which contains our StickyNote sources. That cloned repo will be served by GitLAB. 

- Go to create new project, fill in the form as follows:
  - Project path: `stickynote`
  - Import project from: "git Any repo by URL" `https://github.com/OrdinaNederland/cursus-docker-sampleapp.git`
  - Visibility Level: `Public`
- Create project

You can now check the sources (and change them if you like) using the GitLAB webinterface.


## Setting up Jenkins
Jenkins will be used as our main Continuous Delivery tool. It will build our code, our docker images and manage our Docker containers.


### Setting up Jenkins - Dockerfile
Jenkins is available as Docker image but for this workshop we build our own Jenkins image. We use the Jenkins base image.
- Go to the `/mnt/sda1/cursus-docker/continuous/jenkins` folder. 
- Create a file named `Dockerfile` with the following content:

```docker
FROM jenkins:1.642.1

USER root
RUN apt-get update && apt-get install \  
     wget curl apt-transport-https -y \
  && apt-get clean \
  && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*

USER jenkins

COPY plugins.txt /usr/share/jenkins/plugins.txt
RUN /usr/local/bin/plugins.sh /usr/share/jenkins/plugins.txt
```


### Setting up Jenkins - plugins
Since we try to automate most of the things we are doing, we also automate and manage our Jenkins plugins by specifying them in the `plugins.txt` file.
You can find it in the same dir as the `Dockerfile`. It has the following content: 

```
scm-api:0.2
jquery:1.7.2-1
parameterized-trigger:2.17
authentication-tokens:1.1
credentials:1.22
docker-build-step:1.33
docker-commons:1.2
git-client:1.19.0
git:2.4.0
gradle:1.24
token-macro:1.9
delivery-pipeline-plugin:0.9.8
```


### Setting up Jenkins - docker-compose
The containers for our Continuous Delivery environment are managed by Docker Compose, as seen for GitLab. Now we will add the Jenkins container to the configuration.
- First adjust Jenkins **data** folder with sufficient rights
```
# Navigate to the /mnt/sda1/cursus-docker/continuous directory first
cd /mnt/sda1/cursus-docker/continuous
sudo chmod 777 .data/jenkins
```
- Edit the file `docker-compose.yml`, add the following content at the end.

```
jenkins:
  build: ./jenkins
  volumes:
    - ./.data/jenkins:/var/jenkins_home
    - /var/run/docker.sock:/var/run/docker.sock
  ports:
    - "8080:8080"
  user: jenkins:100
  extra_hosts:
    mavenrepository: 52.30.98.133
```


### Jenkins - Add link to gitlab container
We need to make the GitLab container accessible from the Jenkins container. We do this by adding a `link` in the Jenkins container configuration.
- Add the `links:` section and `git:git` to that section, so the Jenkins container can refer to the git host, using the name "git".

```
jenkins:
  build: ./jenkins
  volumes:
     - ./.data/jenkins:/var/jenkins_home
     - /var/run/docker.sock:/var/run/docker.sock
  ports:
    - "8080:8080"
  user: jenkins:100
  extra_hosts:
    mavenrepository: 52.30.98.133
  links:
    - git:git
```


### Setting up Jenkins - docker-compose (RECAP)
A little recap:	
  - We have defined our Jenkins container in a `docker-compose` file.
  - The container will be built based on a Dockerfile in the directory `./jenkins`.
  - The container will mount the `jenkins_home` directory to the directory `.data/jenkins` on the host.
  - The container maps port `8080` from to container to `8080` on the localhost.
  - The container will have an extra entry in the host file to resolve the dockerhost for building images.


### Setting up Jenkins - LAUNCH!
Ok, time to get Jenkins up and running.
- Start the docker container(s).

  ```
  docker-compose up -d
  ```
- Open Jenkins in a browser (http://<< docker-machine-ip >>:8080).
- Jenkins might not be available straight away. Give it some time to start before entering panic mode.

You might have noticed that only Jenkins got started after the docker-compose command. Docker-compose is smart enough to notice that the GitLAB container was already running.


## Configuring Jenkins
Jenkins requires some configuration before it can interact with Docker. 


### Jenkins - Global configuration
- First we have to set some global configurations in Jenkins.
  - Open the Jenkins application in your browser
  - Go to: Manage Jenkins -> Configure system
  - Search for: `Docker Builder` and provide the url: `unix:///var/run/docker.sock`
  - Press the **Apply** (Toepassen) button at the bottom of the page
  - Now you can press `Test Connection`


## Jenkins - Delivery pipeline
For this workshop Jenkins comes configured with several jobs that form a *Delivery Pipeline* : an aggregation of several small builds to build, test and deliver our service.

![pipeline](images/pipeline-complete.png)


## The main job
![pipeline](images/pipeline-main.png)


### Jenkins - Job main
The **main** job is a simple job that is responsible for checking out the StickyNote GIT repo. Everytime you want to start the pipeline, you will do so by starting this job.

- Inspect the configuration of Jenkins item `service-main`
- In the Delivery Pipeline Configuration it's put in the first **Stage** we called `build` with **Task Name** `clean`
- It sets up a workspace with Git repo `http://git/root/stickynote.git` (remember the linked container was also named `git`)
- Then it triggers the build of another project `service-build` with predefined parameters that are passed onto all the next items:
    ```text
    SOURCE_BUILD_NUMBER=${BUILD_NUMBER}
    SOURCE_BUILD_WORKSPACE=${WORKSPACE}
    VERSION=1.0.${SOURCE_BUILD_NUMBER}
    ```


## The Build job
![pipeline](images/pipeline-build.png)


### Jenkins - Job build
The **build** job runs the Gradle "build" task. This compiles our code and run the unit tests.

- Inspect the configuration of Jenkins item `service-build`
- In the Delivery Pipeline Configuration it's the `build` task in the first *Stage* also called `build`
- The workspace directory is passed through from `service-main` with parameter `${SOURCE_BUILD_WORKSPACE}`
- It invokes the task `build` of a Gradle script
- Then it triggers the build of project `service-package` with the same set of predefined parameters


## The package job
![pipeline](images/pipeline-package.png)


### Jenkins - Job package (1/2)
The package job creates a Docker image for our StickyNote service. The definition of this image is declared in a Dockerfile.

- Inspect the configuration of Jenkins item `service-package`
- In the Delivery Pipeline Configuration it's the `package` task in the `build` stage
- It invokes the task `buildImage` of a Gradle script. 
- When finished it triggers the build of project `service-start`, which is the next pipeline stage.


### Jenkins - Job package (2/2)
- If you like to see how we build our docker image, you can inspect the Gradle script in GitLAB located at `/build.gradle`  
- The `Dockerfile` of the image it builds is located in the GIT repo at `/src/main/resources/docker/Dockerfile`

```docker
FROM java:8-jdk

RUN echo 127.0.2.1 mongodb > /etc/hosts; cat /etc/hosts

ADD stickynote-service.jar /

EXPOSE 8080

CMD ["java","-jar","/stickynote-service.jar"]
```


### Jenkins - Pipeline (1/2)
- In the main screen of Jenkins you can find a tab called `Stickynote pipeline`. 
- It has not ran yet, so there's nothing yet to see.
- Start a build of the `service-main` project.
- Switch to the pipeline view. Notice that `service-main` triggers `service-build` which triggers `service-package`.


### Jenkins - Pipeline (2/2)
- The pipeline will also go through the stages `QA` and `Release` we have not looked at yet. These jobs are still empty, so that won't hurt.

![pipeline](images/pipeline-view.png)


## DockerUI
In the next part we are creating many docker containers. Just for the fun we install a DockerUI on our host.

DockerUI is a web interface for the Docker Remote API. Is gives us a view of running containers and allows us to perform commands like stop, start, remove, etc. 


### Adding docker ui

- Edit `docker-compose.yml` (in `/mnt/sda1/cursus-docker/continuous`) and add the `dockerui` section

```json
dockerui:
  image: abh1nav/dockerui
  privileged: true
  ports:
    - "9090:9000"
  volumes:
    - /var/run/docker.sock:/var/run/docker.sock
```
- Use `docker-compose up -d` to start the dockerui container.
- Browse to `http://<< docker-machine-ip >>:9090` to see which containers are running.


## The start job
![pipeline](images/pipeline-start.png)


### Jenkins - Job service-start (1/3)
The start job will be creating and starting the StickyNote Docker container and the MongoDB database container.

- Go edit the configuration in Jenkins of item `service-start`
- In the Delivery Pipeline Configuration it's the `start` task in the `QA` stage
- When finished it triggers the build of project `service-tests`, which in turn will trigger `service-stop`


### Jenkins - Job service-start (2/3)

 - Add build step: **Execute docker command**
   - Docker command: **pull image**
   - Image name: `mongo`
   - Tag: `3.0.6`
 - Add build step: **Execute docker command**
   - Docker command: **create container**
   - Image name: `mongo:3.0.6`
   - Container name: `test_db`


### Jenkins - Job service-start (3/3)
- Add build step: **Execute docker command**
  - Docker command: **create container**
  - Image name: `stickynote/stickynote-service:latest`
  - Container name: `test_service`
  - *Advanced* - Links: `test_db:mongodb`
- Add build step: **Execute docker command**
  - Docker command: start container(s)
  - Container ID(s): `test_db, test_service`
  - *Advanced* - Port bindings: `8888:8080`
  - *Advanced* - Wait for ports:
  ```text
  test_db 27017
  test_service 8080
  ```
- Save


### The start job (RECAP)
This job effectively performs these Docker CLI command but then through the the Docker API.
```
# Pull and create MongoDB container
docker pull mongodb:3.0.6
docker create --name test_db mongodb:3.06

# StickyNote service: No need to pull, since we've build the image on the same Dockerhost. 
# Therefore it's already available for us.
docker create --name test_service -p 8888:8000 --link test_db:mongodb stickynote/stickynote-service:latest 

# Start containers
docker start test_db test_service
```


## The stop job
![pipeline](images/pipeline-stop.png)


### Jenkins - Job service-stop (1/2)
This job will stop and remove the earlier created containers.

- Go edit the configuration in Jenkins of item `service-stop`
- In the Delivery Pipeline Configuration it's the `stop` task in the `QA` stage
- When finished it triggers the build of project `service-sonar`


### Jenkins - Job service-stop (2/2)
- Add build step: **Execute docker command**
  - Docker command: **Stop container(s)**
  - Container ID(s): `test_service, test_db`
- Add build step: **Execute docker command**
  - Docker command: **Remove container(s)**
  - Container ID(s): `test_service, test_db`
  - *Advanced* - Ignore if not found to TRUE
  - *Advanced* - Remove volumes to TRUE
  - *Advanced* - Force remove to TRUE
- Save and test by starting the `Stickynote pipeline` (or `service-main` project).


### The stop job (RECAP)
This job effectively performs these Docker CLI command but then through the the Docker API.
```
# Stop the containers
docker stop test_service test_db

# Remove the containers
docker rm -f -v test-service test_db 
```


## The tests job
![pipeline](images/pipeline-tst.png)


### Jenkins - Job service-tests (1/3)
The test job perform tests against our running container. Gradle will start a JMeter test and JMeter will use our `test_service` container as its target.

- Go edit the configuration in Jenkins of item `service-tests`
- In the Delivery Pipeline Configuration it's the `test` task in the `QA` stage
- When finished it triggers the build of project `service-stop`


### Jenkins - Job service-tests (2/3)
- Add build step: **Execute shell script**
  - Command:
```
#!/bin/bash
dockerhostip=$(/sbin/ip route|awk '/default/ { print $3 }')
find src/it-test -type f -print0 | xargs -0 sed -i "s/dockerhost/$dockerhostip/g"
find build -type f -print0 | xargs -0 sed -i "s/dockerhost/$dockerhostip/g"
```

<hr>
Background: <br>This piece of bash script replaces every occurence of 'dockerhost' in your job's workspace files with the IP address of your docker host machine.
<hr>


### Jenkins - Job service-tests (3/3)
- Add build step: **Invoke Gradle script**
   - Use gradle wrapper
   - Tasks (click on arrow on the far right for multiline content!)
    ```text
    integrationTest
    jmeterRun
	```
- Save and test by starting the `Stickynote pipeline`.


## The deploy job
![pipeline](images/pipeline-deploy.png)


### Jenkins - Job service-deploy (1/4)
The deploy job deploys (no shit) our's service to a "production (demo)" environment. After the tests have succeeded this job will:
- Tag the image created in the `package` job.
- Remove the current production containers
- Create new production containers based on the new image
- Start the newly created containers


### Jenkins - Job service-deploy (2/4)
- Go edit the configuration in Jenkins of item `service-deploy`.
  This is the only task `deploy` of the `Release` stage of the Delivery Pipeline

- Add build step: **Execute docker command**
   - Docker command: **Tag image**
   - Name of image: `stickynote/stickynote-service:latest`
   - Target repository of the new tag: `stickynote/stickynote-service`
   - The tag to set: `${VERSION}`


### Jenkins - Job service-deploy (3/4)
- Add build step: **Execute docker command**
  - Docker command: **Remove container(s)**
  - Container ID(s): `demo_db, demo_service`
  - *Advanced* - Ignore if not found to TRUE
  - *Advanced* - Remove volumes to TRUE
  - *Advanced* - Force remove to TRUE
- Add build step: **Execute docker command**
  - Docker command: **create container**
  - Image name: `mongo:3.0.6`
  - Container name: `demo_db`


### Jenkins - Job service-deploy (4/4)
- Add build step: **Execute docker command**
  - Docker command: **create container**
  - Image name: `stickynote/stickynote-service:${VERSION}`
  - Container name: `demo_service`
  - *Advanced* - Links: `demo_db:mongodb`
- Add build step: **Execute docker command**
  - Docker command: **start container(s)**
  - Container ID(s): `demo_db, demo_service`
  - *Advanced* - Port bindings: `8887:8080`
  - *Advanced* - Wait for ports:

  ```text
  demo_db 27017
  demo_service 8080
  ```
- Save and test by starting the `Stickynote pipeline`.


### The deploy job (RECAP)
This job effectively performs these Docker CLI command but then through the the Docker API.
```
# Give the image a TAG
docker tag stickynote/stickynote-service:latest stickynote/stickynote-service:<SOMEVERSIONNUMBER>

# Stop and remove existing containers
docker rm -f -v demo_db demo_service

# Create the new containers
docker create --name demo_db mongodb:3.06
docker create --name demo_service -p 8887:8000 --link demo_db:mongodb stickynote/stickynote-service:<SOMEVERSIONNUMBER>

# Start containers
docker start demo_db demo_service
```
The sticky note (REST)application is available at: 

`http://<<machine-ip>>:8887/notes/`


## The sonar job
![pipeline](images/pipeline-sonar.png)


### Docker - Sonar (1/5)
We will add Sonar to keep track of our code quality.

- First we will stop all of our docker services, and add the Sonar docker image to our `docker-compose` file.
  - Switch back to the `ssh` session of our Docker workshop VM
  - Stop the docker services using `docker-compose` 
```
cd /mnt/sda1/cursus-docker/continuous
docker-compose stop
```


### Docker - Sonar (2/5)
- Edit the `docker-compose.yml` file in `/mnt/sda1/cursus-docker/continuous`
- Add a container for the database:

```json
sonardb:
  image: postgres:9
  environment:
    - POSTGRES_USER=sonar
    - POSTGRES_PASSWORD=secret
  volumes:
    - ./.data/sonardb/data:/var/lib/postgresql/data
```


### Docker - Sonar (3/5)
- Add a container for Sonar:

```json
sonar:
  image: sonarqube:5.1.1
  links:
    - sonardb:db
  environment:
    - SONARQUBE_JDBC_USERNAME=sonar
    - SONARQUBE_JDBC_PASSWORD=secret
    - SONARQUBE_JDBC_URL=jdbc:postgresql://db/sonar
  ports:
    - "9000:9000"

```


### Docker - Sonar (4/5)
- And link the two Sonar containers to the Jenkins container
- In the `links:` section of the Jenkins configuration add links from Jenkins to Sonar (see below).
- Save the `docker-compose.yml` file

```
jenkins:
  ...
  links:
    - git:git
    - sonar:sonar
    - sonardb:sonardb
  ...
```


### Docker - Sonar (5/5)
- Restart compose
```
docker-compose up -d
```
- When that is finished, the (empty) SonarQube dashboard should be visible at `http://<<docker-machine-ip>>:9000/`


### Jenkins - job Sonar
Back to Jenkins to include Sonar in our pipeline
- Go edit the configuration in Jenkins of item `service-sonar`.
- Add build step: **Invoke Gradle script**
   - Use gradle wrapper
   - Tasks: `sonarRunner`
- Save and test the pipeline by starting the `Stickynote pipeline`.
- Check the results at `http://<< docker-machine-ip >>:9000/`


And you are..
# DONE!
This is just one way of setting up a simple Continuous Delivery pipeline. 
You might encounter clusters of Docker engines and other ways of controlling where and how containers will be created, started, stopped, etc...

This solution is a simple one, but gives you some idea of how a CD like solution might look like and the various Docker commands it relates to.