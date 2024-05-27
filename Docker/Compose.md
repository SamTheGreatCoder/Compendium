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