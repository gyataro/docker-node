# **Node Docker**
Learning Docker, courtesy of [freeCodeCamp](https://www.youtube.com/watch?v=9zUHg7xjIqQ).

System design: MongoDB, Express, Node, nginx. References [[1]](https://losikov.medium.com/part-9-docker-docker-compose-complete-intro-2cfcc510bd8e) & [[2]](http://docs.opencb.org/display/cellbase/Architecture).

## **1. Managing Individual Containers**
### Build Image
```bash
docker build -t node-app-image .
```

### Run Container
- `-d`: detach container from the root process (CLI) running it.
- `-v`: shared filesystems between local environment and container, `:ro` read only.
- `-p`: map Docker host port to TCP port in the Docker container.
```bash
[Windows CMD] docker run -v %cd%:/app:ro -v /app/node_modules --env-file ./.env -p 3000:4000 -d --name node-app node-app-image
[Windows PS] docker run -v ${pwd}:/app:ro -v /app/node_modules --env-file ./.env -p 3000:4000 -d --name node-app node-app-image
[Linux/Mac] docker run -v $(pwd):/app:ro -v /app/node_modules -p 3000:4000 -d --name node-app node-app-image
```

### Remove Container
```bash
docker rm node-app -fv
```

## **2. Managing Multiple Containers**
### Build Image & Run Containers
Use a YAML file to configure an application’s services. Then, with a single command, create and start all the services from the configuration. 
- `--build`: force build images before starting containers
```bash
[Development] docker-compose -f docker-compose.yml -f docker-compose.dev.yml up -d --build
[Production] docker-compose -f docker-compose.yml -f docker-compose.prod.yml up -d --build
```

### Remove Containers
`prune` deletes all unused local volumes.
```bash
docker volume prune
docker-compose down
```

## **3. Docker Concepts**
### Image Layers & Caching
Each Dockerfile command is an image layer. Docker caches image layers and rebuilds layers that have experienced changes.

### Port Accessibility
Dockerfile `EXPOSE` allows inter-container communication, `docker run -p` makes the service in the container accessible from anywhere, even outside Docker.

### Running Commands in Containers
`docker exec` runs a new command in a running container. Note: this is for testing in development environment, not recommended for production.
```bash
[Format] docker exec [OPTIONS] CONTAINER COMMAND [ARG...]
[Example] docker exec -it node-docker-redis-1 redis-cli
```

### Scaling Instances
We use nginx to setup a load balancer. Then, we use `docker-compose up --scale` to scale services to a defined number of instances.
```bash
docker-compose -f docker-compose.yml -f docker-compose.dev.yml up -d --scale node-app=2
```

## 4. **Deploying to Production**
### Get Docker on VM Instance
After creating a VM instance on AWS/GCP/Azure/DO or any other cloud service provider, we must SSH into the VM and install Docker.
```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sh get-docker.sh
```
Get stable release of Docker Compose.
```bash
sudo curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
```

### Configure Environment Variables
Do not add production .env variables to `docker-compose.prod.yml` as this will expose all production API keys and secrets, which is a security risk. Instead, add it manually in the VM instance.

Add a `.env` file in root, and persist the variable across reboots.
```bash
[Add variables] vi .env
[Modify startup file] vi .profile
    set -o allexport; source /root/.env; set +o allexport
```

### Improved Dockerhub Workflow
Building images require computing resources, which may compete with services running in production environment. Therefore, our usual workflow is to:
1. [Development] Build images in local environment
2. [Development] Push images to Docker repository
3. [Production] Pull from Docker repository
```bash
[Login] docker login
[Modify image tag] docker image tag node-docker_node-app gyataro/docker-node
[Push to repository] docker push gyataro/docker-node
[Push specific image] docker-compose -f docker-compose.yml -f docker-compose.prod.yml push node-app
```

### Automating with Watchtower
```bash
docker run -d --name watchtower -e WATCHTOWER_TRACE=true -e WATCHTOWER_DEBUG=true -e WATCHTOWER_POLL_INTERVAL=50 -v /var/run/docker.sock:/var/run/docker.sock containrrr/watchtower docker-node_node-app_1
```

### Container Orchestration
We can automate the deployment, management, scaling, and networking of containers with orchestrators such as Kubernetes. Docker's built in orchestrator is Docker Swarm.
```
Manager Node (Control Plane) <---> Worker Node (Data Plane)
```
```bash
docker swarm init
docker stack deploy -c docker-compose.yml -c docker-compose.prod.yml docker-node
docker stack services docker-node
```