# Docker Compose
## Connect container to network:
"you need to declare networks before you use it in services."\
https://stackoverflow.com/a/74274260
``` yaml
version: "3.9"
services:
    result:
        image: result
        ports:
            - "8080:80"
        networks:
            - test-wp-network
networks:
  test-wp-network:
    external:
      name: "test-wp-network"
```
## Portainer/Agent in Compose
https://www.herlitz.io/2024/03/deploy-portainer-and-portainer-agent-using-docker-compose/
``` yaml
version: '3.9'
name: portainer
services:
  portainer-agent:
    container_name: portainer-agent
    image: portainer/agent
    ports:
      - "9001:9001" 
    volumes:
      # Mount the host's Docker socket into the container
      - /var/run/docker.sock:/var/run/docker.sock
      # Mount the host's Docker volumes into the container
      - /var/lib/docker/volumes:/var/lib/docker/volumes
    depends_on:
      - portainer
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 1024M
      restart_policy:
        condition: unless-stopped
        delay: 5s
        window: 120s
  portainer:
    container_name: portainer
    image: portainer/portainer-ce:latest
    ports:
      - "8000:8000"
      - "9443:9443"
    volumes:
      # Mount the host's Docker socket into the container
      - /var/run/docker.sock:/var/run/docker.sock
      # Create a named volume for persistent Portainer data storage
      - portainer_data:/data
    networks:
      - portainer_network
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 1024M
      restart_policy:
        condition: unless-stopped
        delay: 5s
        window: 120s
networks:
  portainer_network:
    driver: bridge
volumes:
  portainer_data:
    name: portainer_data
```