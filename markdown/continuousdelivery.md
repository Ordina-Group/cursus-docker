## Continuous Delivery using Docker
*In this chapter you will learn how you can make use of Docker in your Continuous Delivery pipeline*

![jenkins-docker](images/docker-jenkins.png)


## Some internals
We've prepared a working Continuous Delivery setup for a simple Mario webapplication game based on Spring Boot
- The Mario game will be build using Gradle
- We build a fat jar and distribute that jar using Docker controlled by Jenkins


## Docker container overview
In the next steps, we will ask you to configure only the missing Jenkins configuration that make use of Docker.
When you are done, you will have these containers running. Some will be maniputed by Jenkins and some by using docker-compose.
 
![pipeline](images/container-overview-ci-all.png)


## The Pipeline
These are the jobs that take part of the pipeline in Jenkins.
![pipeline](images/pipeline-complete.png)


# ..But first... 
#...some preparations...


## Windows and less than 20GB on the C disk ?? (1/3)
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
- Do the same for other running docker-machines (you can list them with `docker-machine ls`)
- Continue on next slide


## Windows and less than 20GB on the C disk  (2/3)
- Using Windows explorer, create a folder on a disk with more than 20GB free space (e.g. `D:\docker-machines`)
- Move the folder `C:\Users\<username>\.docker\machine` to your created directory.
- Open a Command prompt with administrator rights
  - <div style="font-size: 0.8em">Windows **Start** menu, type `cmd`, right-click `cmd.exe`, click *"run as administrator"* </div>
- Create a symbolic link on the original location and point it to the location where your data is stored
```
mklink /D c:\Users\<username>\.docker\machine d:\docker-machines\machine
```


## Windows and less than 20GB on the C disk  (3/3)
- Return to your quickstart terminal and start the workshop machine
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
- Next you have to set a new password.
 - If you wanna be lazy.. you can change it to the original `5iveL!fe` password.
- After changing the password you land back on the login page.
  - Login again with username `root` and the new password.


### Setting up GitLab - Configure
We will clone a remote repo which contains our Mario sources. That cloned repo will be served by GitLAB. 

- Go to create new project, fill in the form as follows:
  - Project path: `mario`
  - Import project from: "git Any repo by URL" `https://github.com/OrdinaNederland/cursus-docker-sampleapp.git`
  - Visibility Level: `Public`
- Create project

You can now view the source files (and also change them) using the GitLAB webinterface.


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
     wget curl apt-transport-https docker.io -y \
  && apt-get clean \
  && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*

USER jenkins

COPY plugins.txt /usr/share/jenkins/plugins.txt
RUN /usr/local/bin/plugins.sh /usr/share/jenkins/plugins.txt
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


## Jenkins - Delivery pipeline
For this workshop Jenkins comes configured with several jobs that form a *Delivery Pipeline* : an aggregation of several small builds to build, test and deliver our service.

![pipeline](images/pipeline-complete.png)


## The package job
![pipeline](images/pipeline-package.png)


### Jenkins - Job package (1/2)
The package job creates a Docker image for our Mario service. The definition of this image is declared in a Dockerfile.

- Inspect the configuration of Jenkins item `service-package`
- In the Delivery Pipeline Configuration it's the `package` task in the `build` stage
- It invokes the task `buildImage` of a Gradle script. 
- When finished it triggers the build of project `service-start`, which is the next pipeline stage.


### Jenkins - Job package (2/2)
- If you like to see how we build our docker image, you can inspect the Gradle script in GitLAB located at `/build.gradle`  
- The `Dockerfile` of the image it builds is located in the GIT repo at `/src/main/resources/docker/Dockerfile`

```docker
FROM java:8-jdk

ADD mario.jar /

EXPOSE 8080

CMD ["java","-jar","/mario.jar"]
```


### Jenkins - Pipeline (1/2)
- In the main screen of Jenkins you can find a tab called `Mario pipeline`. 
- It has not ran yet, so there's nothing yet to see.
- Start a build of the `service-main` project.
- Switch to the pipeline view. Notice that `service-main` triggers `service-build` which triggers `service-package`.


### Jenkins - Pipeline (2/2)
- The pipeline will also go through the stages `QA` and `Release` we have not looked at yet. These jobs still need some Docker configurations, but for now it won't hurt.

![pipeline](images/pipeline-view.png)


## The start job
![pipeline](images/pipeline-start.png)


### Jenkins - Job service-start (1/2)
The start job will be creating and starting the Mario Docker container.

- Go edit the configuration in Jenkins of item `service-start`
- In the Delivery Pipeline Configuration it's the `start` task in the `QA` stage
- When finished it triggers the build of project `service-tests`, which in turn will trigger `service-stop`


### Jenkins - Job service-start (2/2)
- Under **Bouwstappen** click **Voeg een bouwstap toe** (or *Add build step*) and pick **Voer shell-script uit**
  ```bash
  # Remove existing (if exists, if not ignore error (that what '|| true' is for))
  docker rm -f test_mario || true
  
  # Start a fresh instance of a test_mario container based upon the latest mario/mario image build by the package job
  docker run --name test_mario -p 8888:8080 -d mario/mario:latest
  ```
- Save


## The stop job
![pipeline](images/pipeline-stop.png)


### Jenkins - Job service-stop (1/2)
This job will stop and remove the earlier created container.

- Go edit the configuration in Jenkins of item `service-stop`
- In the Delivery Pipeline Configuration it's the `stop` task in the `QA` stage
- When finished it triggers the build of project `service-sonar`


### Jenkins - Job service-stop (2/2)
- Under **Bouwstappen** click **Voeg een bouwstap toe** (or *Add build step*) and pick **Voer shell-script uit**
  ```bash
  # Stop test_mario container
  docker stop test_mario
  
  # Remove the test_mario container
  docker rm -f test_mario
  ```
- Save and test by starting the `Mario pipeline` (or `service-main` project).


## The tests job
![pipeline](images/pipeline-tst.png)


### Jenkins - Job service-tests (1/3)
The test job performs integration tests against our running container. 

- Go edit the configuration in Jenkins of item `service-tests`
- In the Delivery Pipeline Configuration it's the `test` task in the `QA` stage
- When finished it triggers the build of project `service-stop`


### Jenkins - Job service-tests (2/3)
- Add build step: **Voer shell-script uit** (or *Execute shell script*)
  - Command:
  
```
#!/bin/bash
dockerhostip=$(/sbin/ip route|awk '/default/ { print $3 }')
find src/it-test -type f -print0 | xargs -0 sed -i "s/dockerhost/$dockerhostip/g"
find build -type f -print0 | xargs -0 sed -i "s/dockerhost/$dockerhostip/g"
```

----

*Background*:

<small>This piece of bash script replaces every occurence of 'dockerhost' in your job's workspace files with the IP address of your docker host machine.</small>

----


### Jenkins - Job service-tests (3/3)
- Add build step: **Invoke Gradle script**
   - Use gradle wrapper
   - Tasks: `integrationTest`
- Save and test by starting the `Mario pipeline`.


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

- Under **Bouwstappen** click **Voeg een bouwstap toe** (or *Add build step*) and pick **Voer shell-script uit**
  ```bash
  # Tag the image mario/mario:latest with a version number so we can refer to it later
  docker tag mario/mario:latest mario/mario:${VERSION}
  
  # Stop and remove the existing Mario production container
  docker stop demo_mario || true
  docker rm -f demo_mario || true
  
  # Create and start a new Mario production container
  docker run --name demo_mario -p 8887:8080 -d mario/mario:${VERSION}
  ```
- Save and test by starting the `Mario pipeline`.


### Start playing!
The Mario demo application is available at: 

`http://<<machine-ip>>:8887/`


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
- Save and test the pipeline by starting the `Mario pipeline`.
- Check the results at `http://<< docker-machine-ip >>:9000/`


And you are..
# DONE!
This is just one way of setting up a simple Continuous Delivery pipeline. 
You might encounter clusters of Docker engines and other ways of controlling where and how containers will be created, started, stopped, etc...

This solution is a simple one, but gives you some idea of how a CD like solution might look like and the various Docker commands it relates to.