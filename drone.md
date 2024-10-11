# Drone how-to

## Introduction

[Drone](https://docs.drone.io/) is used to build CD platform. It is easy to deploy and use. In my app, I use Drone and Gogs to build my pipeline.


## Install

Optionally, use `docker-compose` to deploy Drone

```yaml
version: "3"
volumes: 
  dronedata:
services:
#drone server, use your machine ip
  drone-server:
    image: drone/drone:2
    environment:
      DRONE_AGENTS_ENABLED: "true"
      DRONE_GOGS_SERVER: "http://server:gogsport" ## example: 192.168.1.1:10880
      #generate secret using  `openssl rand -hex 16` 
      DRONE_RPC_SECRET: "[RPC_SECRET]"
      DRONE_SERVER_HOST: "server:port" ## example: 192.168.1.1:9080
      DRONE_SERVER_PROTO: "http"
      #create admin uesr, change username to gogs username
      DRONE_USER_CREATE: "username:[username],machine:false,admin:true"
    restart: always
    container_name: drone-server
    ports:
    - 9080:80
    - 9443:443
    volumes:
    - dronedata:/data
  #drone client
  drone-runner:
    image: drone/drone-runner-docker:1
    environment:
      DRONE_RPC_PROTO: "http"
      DRONE_RPC_HOST: "server:port"  ## example: 192.168.1.1:9080
      #the RPC_secret should be the same as the above RPC_SECRET
      DRONE_RPC_SECRET: "[SECRET]"
      DRONE_RUNNER_CAPACITY: "2"
      DRONE_RUNNER_NAME: "my-first-runner"
    ports:
    - 3000:3000
    restart: always
    container_name: drone-runner
    depends_on:
    - drone-server
    volumes:
    - /etc/docker/:/etc/docker
    - /var/run/docker.sock:/var/run/docker.sock
```

## Log into Drone and activate repository

Visit server IP, example: `192.168.1.1:9080` with Gogs credentials


## Construct Build Pipeline using YAML
Create `.drone.yml` in the code repo. After pushing the code to Gogs, Drone will automatically 


```yaml
kind: pipeline
type: docker
name: kubecenter-publish
steps:
  - name: build
    image: plugins/docker
    volumes:
      - name: hosts
        path: /etc/hosts
      - name: docker-ca
        path: /etc/docker
      - name: dockersock
        path: /var/run/docker.sock
    settings:
      username: admin
      password:
        from_secret: harbor_password
      repo: harbor.kubecenter.com/kubecenter/kubecenter
      registry: harbor.kubecenter.com
      tags:
        - v1.2
  - name: ssh commands
    image: appleboy/drone-ssh
    settings:
      host: [IP address]
      username: root
      password:
        from_secret: ssh_password
      port: 22
      script:
        #pull the image and deploy
        - if [ $(docker ps -a | grep kubecenter-server | wc -l) -ge 1 ];then docker stop kubecenter-server && docker rm kubecenter-server; fi
        - docker pull harbor.kubecenter.com/kubecenter/kubecenter:v1.2
        - docker run --name kubecenter-server --restart=always -d -p7080:7080 harbor.kubecenter.com/kubecenter/kubecenter:v1.2
volumes:
  - name: hosts
    host:
      path: /etc/hosts
  - name: docker-ca
    host:
      path: /etc/docker
  - name: dockersock
    host:
      path: /var/run/docker.sock
```

