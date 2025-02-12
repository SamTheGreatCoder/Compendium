# Setup Nginx Proxy Manager
References:\
https://www.linode.com/docs/guides/using-nginx-proxy-manager/\
https://www.youtube.com/watch?v=qlcVx-k-02E
## Steps:
1. Install - Docker Compose
2. Admin portal: http://```HOSTNAME```:81
    - Where *HOSTNAME* is either the local DNS or IP address of the system where you're running nginx proxy manager from
    - Port 81 is used specifically for the admin console, probably a good idea to keep this one secure, and not forward it
    - Port 80 and 443 are used by the actual proxy system, these should be forwarded
3. Default login: ```admin@example.com```, ```changeme```
    - You'll be prompted to update both on first login
4. Create proxy host
    - ```Proxy hosts``` > ```Add Proxy Host```
    - Domain name is what you/others will enter URL to navigate
    - Scheme is for internal access
        - Portainer could be HTTP/S, Frigate NVR would be HTTP
    - Forward Hostname/IP is the service being proxied
        - **REMEMBER**: Using docker bridge networks? Just use the name of the container, it uses the docker network's built in DNS resolver, even after container IP addresses may change through recreates
        - Eg. Portainer on Docker = ```portainer```, Frigate NVR = ```frigate```, Home Assistant = ```homeassistant```, Authentik = ```authentik-server```
    - Forward Port is just the access port for the proxied service
        - Eg. Portainer HTTPS = ```9443```, Frigate NVR = ```5000```, Home Assistant = ```8123```
    - I'd argue that caching assets for services hosted on the same system isn't necessary. I have no data in favor or against it, but given that assets weren't cached before Nginx Proxy Manager and it loaded just fine, I see no need to explore that option.
    - Enabling ```Block Common Exploits``` has no downside that I can tell, at least from references. Also the name seems to explain pretty straightforward.
    - Enabling ```Websockets Support``` depends on the service. For example, Home Assistant requires websockets. I'd argue it's better to leave disabled until you verify the forwarded service requires it. Less things to have open when public facing.
5. SSL Certificate for the proxy host **(The whole point of using this, right?)**
    - Request a new SSL Certificate
        - **UNLESS** you have a paid domain through (eg. Cloudflare) then you'll probably want to use wildcard domains setup separately. Free systems like DuckDNS/No-IP require a separate SSL certificate for each subdomain since they'll fail the ```DNS Challenge``` option.
    - Enable ```Force SSL``` (again, it's the whole point, right?)
    - Enable ```HTTP/2 Support``` (I forget why it's needed, or rather really good to have on)
    - Enable ```HSTS Enabled``` and ```HSTS Subdomains``` (Really no clue what it's for, but why not I suppose)
    - **DISABLE** ```DNS Challenge```
        - See note under ```Request a new SSL Certificate``` - I don't have a paid domain, just some DuckDNS entries.
6. ```Save``` and profit!