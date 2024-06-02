# Eclipse Mosquitto MQTT Broker

## My Version:
NOTE: My current Home Assistant setup doesn't requrie MQTT to be used for any external services so I'm restricting to the MQTT broker for only the services running inside the docker bridge network.
``` yaml
services:
  mosquitto:
    image: eclipse-mosquitto
    container_name: mosquitto-mqtt
    restart: unless-stopped
    volumes:
      - /opt/pve/mosquitto-mqtt/config:/mosquitto/config
      - /opt/pve/mosquitto-mqtt/data:/mosquitto/data
      - /opt/pve/mosquitto-mqtt/log:/mosquitto/log
    # ports:
        # - 1883:1883
        # - 9001:9001
    networks:
      - backend
networks:
  backend:
    external: true
```
## Based on:
https://pimylifeup.com/mosquitto-mqtt-docker/
``` yaml
services:
  mosquitto:
    image: eclipse-mosquitto
    container_name: mosquitto
    volumes:
      - ./config:/mosquitto/config
      - ./data:/mosquitto/data
      - ./log:/mosquitto/log
    ports:
      - 1883:1883
      - 9001:9001
    stdin_open: true 
    tty: true
```