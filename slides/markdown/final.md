# More Docker material
*Wanna know more.. here is some stuff you might like*


## Docker networking
You can see a Docker network as an alternative to the "--link" option which you have used to link together containers.	

To build web applications that act in concert but do so securely, use the Docker networks feature. Networks, by definition, provide complete isolation for containers. So, it is important to have control over the networks your applications run on. Docker container networks give you that control.
https://docs.docker.com/engine/userguide/networking/dockernetworks/


## Docker Swarm - intro
Docker Swarm is native clustering for Docker. It turns a pool of Docker hosts into a single, virtual Docker host. Because Docker Swarm serves the standard Docker API, any tool that already communicates with a Docker daemon can use Swarm to transparently scale to multiple hosts. Supported tools include, but are not limited to, the following:

- Dokku
- Docker Compose
- Krane
- Jenkins

And of course, the Docker client itself is also supported.
https://docs.docker.com/swarm/overview/


## Docker Swarm - overlay networking
If you are using Docker networking in combination with Docker Swarm, you might wanne use `Overlay Networking` to enable your containers to live on different hosts and still be able to communicate with each other.

With this article will guide you in creating a Docker Swarm cluster combined with Consul to create an overlay network ready cluster.
https://docs.docker.com/engine/userguide/networking/get-started-overlay/