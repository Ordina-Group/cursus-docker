## Running a chat application using docker-compose 
![rocket-chat](images/rocket-dot-chat-logo.png)


## What is Rocket.Chat?
Rocket.Chat is a Web Chat Server, developed in JavaScript, using the Meteor fullstack framework.

It is a great solution for communities and companies wanting to privately host their own chat service or for developers looking forward to build and evolve their own chat platforms.

In a way it's kinda like Slack


### Assignment
Run an instance of RocketChat.
<small>
- Get yourself an account on DockerHUB.com and search for the Rocket.chat image
- Create a docker-compose (in v2 format) file that:
  - ... containes two services
    1. Rocket.chat service
    2. MongoDB service
  - ... defines that the Rocket.chat service depends on MongoDB service
    - This defines the startup order
  - ... binds the a port 3333 on the docker machine to the port that Rocketchat exposes itself on

Use these pages as reference:<br>
https://hub.docker.com/_/rocket.chat/<br>
https://docs.docker.com/compose/compose-file/<br>
https://docs.docker.com/compose/reference/overview/

</small>
Note:
solution at: <a href="../../solutions/rocketchat/docker-compose.yml" /> solution </a>


### If all is ok
![rocket-chat](images/rocket-dot-chat-login.png)

then the application should be available on <br>
`http://< your-docker-machine-ip >:3333/`
