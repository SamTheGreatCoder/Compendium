# Setup Eclipse Mosquitto MQTT Broker
References:\
https://pimylifeup.com/mosquitto-mqtt-docker/
## Steps:
1. Install - Docker Compose
2. Create a config file for the broker
    - This needs to go where you've mapped the config directory host path in the docker compose file
    - ```mosquitto.conf```

**My config file:**
```
password_file /mosquitto/config/passwd_file
allow_anonymous false
listener 1883 0.0.0.0
```
Did I really need to put 0.0.0.0 as an IP range for the listener? Probably not. Wouldn't hurt I assume though.
**Reference config file:**
```
listener 1883
listener 9001
protocol websockets
persistence true
persistence_file mosquitto.db
persistence_location /mosquitto/data/

#Authentication
allow_anonymous false
password_file /mosquitto/config/pwfile
```
- Listener is for what port for MQTT broker to listen on
    - 1883 and 9001 are for either unencrypted or encrypted, I don't remember which is for what though.
    - You can limit the IP Address range by adding a range after the port
        - Eg. ```listener 1883 127.0.0.1``` only listens on port 1883 on localhost
- Protocol is for, well the protocol of communication
    - I don't have that defined, I don't remember the reason.
- Persistence keeps the messages, "present" even after sending them
    - I would say that Home Assistant and Ring-MQTT use them, but it's not defined in my config file, and again I don't remember the reason.
- Persistence file is a database file for persistent messages
    - I feel like this is a given
- Persistence location is the folder where the ```persistence_file``` will be stored
    - Also a given?
- I set ``allow_anonymous`` to false because I don't need random/public/guest access for MQTT, since it also allows authenticated sending of messages. No thanks.
- Password file stores the authenticatable users and passwords for the broker. 99.9% sure it's needed if you're not using anonymous access.
3. Password file (assuming we're using authenticated access)
    - Needs to be wherever the docker compose has the config folder mapped, or whatever random path you set the password file to be at
    - ```chmod 0700 /opt/eclipse-mqtt/config/pwfile```
        - The broker **WILL** complain about "globally accessable" password files and at the time of commit, says it will stop supporting it in the future. You have been warned.
4. Setup users and passwords
    - **NOTE: The referenced guide attaches to the docker container to execute the commands but I don't remember if that's actually how I did it.**
    - ```docker compose exec container_name sh```
        - Where ```container_name``` is the name of the container, either randomly generated or set in the docker compose file
        - ```sh``` connects to shell
    - ```mosquitto_passwd -c /mosquitto/config/pwfile USERNAME```
        - I don't remember what the ```-c``` flag is for. Password file path?
        - Insert the path where your password file is, check ```mosquitto.conf``` and docker compose files.
        - ```USERNAME``` is the username for the first authenticated user entry in the password file.
    - You'll be prompted to enter the password, and verify.
5. Exit and restart the container
6. Profit!


**NOTE: I don't remember if this encrypts the password file or not, which you should do anyway. Worth looking into.**