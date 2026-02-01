# Portainer

## Overview

## Getting Started

### Setup Volumes

- Create Docker Volume e.g. `portainer_data` : `docker volume create portainer_data`

### Run Portainer

- Run the container and point to the volume created:

```shell
docker run -d \
  -p 9000:9000 \
  --name portainer \
  --restart always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce
```

- Verify with `docker ps` and navigate to `http://localhost:9000` (adjust port depending on selection)

### First Logon Steps

- Create `admin` user account, and select `Local` environment to have Portainer connect to your local environment.
- From here, you're done!

## Shutting Down Portainer

- Stop the container: `docker stop portainer`
- Remove the container: `docker rm portainer`
- Remove the volume & data: `docker rm portainer_data`
- (Optional) Remove the image: `docker image rm portainer/portainer-ce`
