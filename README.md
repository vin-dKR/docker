### Docker Basic Commands

#### 1. Docker Images
* `docker images`: Lists all the images available on the system.
* `docker pull <image_name>`: Pulls an image from Docker Hub.
* `docker rmi <image_name>`: Removes an image from the system.

#### 2. Docker Containers
* `docker ps`: Lists all running containers.
* `docker ps -a`: Lists all containers, including stopped ones.
* `docker run <image_name>`: Runs a new container from an image.
* `docker run -p <host_port>:<container_port> <image_name>`: Runs a new container with port mapping.
* `docker stop <container_id>`: Stops a running container.
* `docker start <container_id>`: Starts a stopped container.
* `docker rm <container_id>`: Removes a container.

#### 3. Building Images
* `docker build -t <image_name> .`: Builds an image from the current directory.
* `docker build -t <image_name> -f Dockerfile .`: Builds an image from a specific Dockerfile.

#### 4. Running Images
* `docker run -d <image_name>`: Runs a container in detached mode.
* `docker run -it <image_name>`: Runs a container in interactive mode.

#### 5. Pushing to Docker Hub
* `docker login`: Logs in to Docker Hub.
* `docker tag <image_name> <username>/<image_name>`: Tags an image for Docker Hub.
* `docker push <username>/<image_name>`: Pushes an image to Docker Hub.

#### 6. Optimizing Dockerfile with Layers
* `docker system prune -a`: Removes all unused images.
* `docker system prune -af`: Forces the removal of all unused images.
* `docker system prune -af && docker system prune -af`: Removes all unused images and then forces the removal again to ensure all layers are removed.

#### 7. Networks
* `docker network create <network_name>`: Creates a new network.
* `docker network connect <network_name> <container_id>`: Connects a container to a network.
* `docker network disconnect <network_name> <container_id>`: Disconnects a container from a network.

#### 8. Volumes
* `docker volume create <volume_name>`: Creates a new volume.
* `docker volume prune -a`: Removes all unused volumes.
* `docker volume prune -af`: Forces the removal of all unused volumes.

#### 9. Docker Compose
* `docker-compose up`: Starts all services defined in docker-compose.yml.
* `docker-compose down`: Stops all services defined in docker-compose.yml.
* `docker-compose ps`: Lists all services defined in docker-compose.yml.
