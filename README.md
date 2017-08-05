# Description

When you run web servers or other services inside Docker containers on your local machine, you may need domains to access them.
In that case you probably forward ports from the host machine to the containers or use a reverse proxy to made it easier, then edit /etc/hosts manually, unless you forget it. 

This image based on the great [jwilder/docker-gen](https://hub.docker.com/r/jwilder/docker-gen). Using it you can generate a list of IP addresses and host names, which can be used to update your local machine's hosts file by [rimelek/hosts-updater](https://hub.docker.com/r/rimelek/hosts-updater/) 

You can decide per container whether the hosts belong to a specific container (reverse proxy for example) or the container where the hosts are defined.

- Before you start the updater, make sure you have backup for /etc/hosts that won't be touched any container.
- The next step is creating a hosts template. Copy the original hosts file as /etc/hosts.docker.tpl.   

Here is an example Docker Compose file without reverse proxy:

    version: "2"
    
    volumes:
      hosts:
    
    services:
      hosts-updater:
        image: rimelek/hosts-updater
        container_name: hosts-updater
        volumes:
          - /etc/hosts:/hosts/orig
          - /etc/hosts.docker.tpl:/hosts/tpl:ro
          - hosts:/hosts
      hosts-gen:
        image: rimelek/hosts-gen
        container_name: hosts-gen
        volumes:
          - /var/run/docker.sock:/tmp/docker.sock:ro
          - hosts:/hosts
        environment:
          UPDATER_CONTAINER: hosts-updater
          
The environment variable UPDATER_CONTAINER contains the real name of the container and not service name.
Now you have a working updater and you can run your application:

    version: "2"
    
    services:
      httpd:
        image: httpd:2.4
        environment:
          VIRTUAL_HOST: my.first.domain.local,my.second.domain.local
          
The environment variable VIRTUAL_HOST can contain multiple domains separated by commas

Let's see how the application's compose file looks like when you forward the ports from your host machine to the container and want to use local IP addresses.

    version: "2"
    
    services:
      httpd:
        image: httpd:2.4
        environment:
          VIRTUAL_HOST: my.first.domain.local,my.second.domain.local
        ports:
          - "80:80"
        labels:
          hosts.updater.target: 127.0.0.1
          
When you have a reverse proxy and you need the hosts to point the ip address of the proxy container, you need to add a label to the proxy container and refer this label in your application's compose file:

    version: "2"
    
    services:
      proxy:
        image: nginx:1:10
        labels:
          - hosts.updater.proxy
    # ...   
    
The above code is just an incomplete proxy definition example. See application's definition below:

    version: "2"
    
    services:
      httpd:
        image: httpd:2.4
        environment:
          VIRTUAL_HOST: my.first.domain.local,my.second.domain.local
        labels:
          hosts.updater.target: label:hosts.updater.proxy
          
Note that when you refer to the target container by label, the prefix "label:" must be used.
The name of the label after the prefix is optional. It just must be the same as the label you added to the proxy.

Now I have to call your attention again to make backup for the original hosts file. I have never experienced any issue yet but always expect the unexpected! 