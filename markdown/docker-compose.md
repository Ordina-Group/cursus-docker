## Running a chat application using docker-compose 
![rocket-chat](images/rocket-dot-chat-logo.png)


## What is Rocket.Chat?
Rocket.Chat is a Web Chat Server, developed in JavaScript, using the Meteor fullstack framework.

It is a great solution for communities and companies wanting to privately host their own chat service or for developers looking forward to build and evolve their own chat platforms.

In a way it's kinda like Slack.


### Assignment: Run an instance of RocketChat

- Visit [DockerHub.com](https://hub.docker.com) and search for details of the *official* Rocket.Chat docker image.
- Create a [docker-compose file](https://docs.docker.com/compose/compose-file/) (in [v2 format](https://docs.docker.com/compose/compose-file/#/version-2)) that:
  - contains two services:
    1. Rocket.Chat service
    2. MongoDB service
  - defines that the Rocket.Chat service [depends on](https://docs.docker.com/compose/compose-file/#/depends-on) the MongoDB service (this defines the startup order)
  - [binds port](https://docs.docker.com/compose/compose-file/#/ports) 3333 on the docker machine to the port that RocketChat exposes itself on

Note:
solution at: <a href="../../solutions/rocketchat/docker-compose.yml" /> solution </a>


### If all is ok
![rocket-chat](images/rocket-dot-chat-login.png)

then the application should be available on <br>
`http://< your-docker-machine-ip >:3333/`
