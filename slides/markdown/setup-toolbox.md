# Setup
*Go grab your toolbox!*

![logo](images/toolbox.jpg)


### No, not kidding, get that toolbox
To get things started you will need the docker toolbox. 

https://www.docker.com/docker-toolbox

What's in the toolbox:
- Docker Client
- Docker Machine
- Docker Compose
- Docker Kitematic
- Oracle VirtualBox


###  After installation
After installation of the docker toolbox:
- run the "Docker Quickstart Terminal".
 - A `default` Virtualbox VM will start.

The `default` docker machine not gonna cut it for this workshop. We will stop it for now and create our own `workshop` VM on the next slide
Stop the `default` VM

```
docker-machine stop default
```


## Create the workshop VM
Create a new VM which will be used for our workshop.
- You will need around 20GB's of free storage space. For Ordina lappies that requires the VM to be installed on the *D* disk.

```
docker-machine create workshop --driver virtualbox --engine-insecure-registry 172.18.18.29/32 --engine-registry-mirror http://172.18.18.29:5000 --virtualbox-memory "4096" --virtualbox-cpu-count "4"  
```

This machine differs in two ways from the `default` VM.
- It's assigned 4 GB's of memory instead of 1GB
- It's assigned 4 CPU's instead of 1 CPU
- It uses a local `Docker registry` to speed up our Docker pulls. Make sure you have a connection to the Ordina WORK network.


## Login to your VM
```
$ docker-machine ssh workshop
```
- clone the workshop GIT repo:

```
docker@workshop:~$ git clone https://github.com/OrdinaNederland/cursus-docker
```
- The filesystem of the docker machine is volatile. For you to keep your changes upon restarts, we need to perform a workaround.

```
docker@workshop:~$ sudo mv cursus-docker /mnt/sda1/
```


## You are good to go!
Head to the next chapter