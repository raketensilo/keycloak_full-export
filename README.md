# keycloak_full-export
## Configuring a Keycloak server to obtain full-export functionality

***
_We use jboss/keycloak:4.3.0.Final. Other keycloak versions may differ._


### (A) The problem

Keycloak admin-console and API offer the possibility of a full-import (realms, roles AND users) but only partial-export (realms, roles but NOT users).  
Full-export however can be achieved by modifying the keycloak server-config.

### (B) A solution

Combining
* `docker` to containerize/virtualize the keycloak server
* `jboss-cli` to manage the keycloak server
* `embed-server` to modify the keycloak-server config without starting the server

### (C) First lets try it manually
1. Create a folder 'keycloak' inside our workspace
>mkdir keycloak  
>ls
```
keycloak/
```

1. Using a Docker container that contains at least 1 keycloak server so we can quickly spin up a container with a server in it and safely play around inside the container (guest) without having to fear about any implications or changes for our OS (host).  

1. Inside our newly created folder `keycloak` lets define the docker container by creating a `docker-compose.yml` file
```
version: "3.5"
services:
  keycloak1:
    image: jboss/keycloak:4.3.0.Final
    environment:
      KEYCLOAK_USER: admin
      KEYCLOAK_PASSWORD: admin
      DB_VENDOR: H2
    ports:
      - 9080:8080
```

1. We start the container with `docker-compose`. We specify the flag `-d` to have the container run in detached mode to not bind it directly to our shell/cli and instead have the container with startup script run in the background.
>docker-compose up -d
```
Creating keycloak_keycloak1_1 ... done
```

1. Check if the docker container is running and to get its id
>docker ps  
```
CONTAINER ID        IMAGE                   COMMAND                  CREATED             STATUS              PORTS                    NAMES
5d0d791a518b        jboss/keycloak:4.3.0.Final   "/opt/jboss/docker-e…"   50 seconds ago      Up 47 seconds       0.0.0.0:9080->8080/tcp   keycloak_keycloak1_1
```

1. We jump into the container by using the container-id
>docker exec -i -t 5d0d791a518b /bin/bash
```
[jboss@5d0d791a518b ~]$
```

1. We find ourselves inside a CentOS Linux distribution, our present working directory is `/opt/jboss` and we see 2 files, 1 directory and the keycloak server is directly in front of us
> cat /etc/\*-release
```
CentOS Linux release 7.4.1708 (Core)
```
>pwd
```
/opt/jboss
```
>ls
```
build-keycloak.sh  docker-entrypoint.sh  keycloak
```

1. We move into the keycloak folder
>cd keycloak/  
>ls
```
bin  cli  docs  domain  jboss-modules.jar  License.html  LICENSE.txt  modules  standalone  themes  version.txt  welcome-content
```

1. Type `jboss-cli` to jump directly into the keycloak server
>./bin/jboss-cli.sh
```
You are disconnected at the moment. Type 'connect' to connect to the server or 'help' for the list of supported commands.
[disconnected /]
```

1. Then `embed-server` to modify the server-config without actually launching the server
> embed-server --server-config=standalone.xml
```
[standalone@embedded /]
```
> /system-property=keycloak.migration.action/:add(value=export)
```
{"outcome" => "success"}
[standalone@embedded /]
```
> /system-property=keycloak.migration.provider/:add(value=dir)
```
{"outcome" => "success"}
[standalone@embedded /]
```
> /system-property=keycloak.migration.dir/:add(value=keycloak_full-export)
```
{"outcome" => "success"}
[standalone@embedded /]
```

1. We quit the jboss-cli and exit the container to find ourselves once again in our OS (host)
> q
```
[jboss@5d0d791a518b keycloak]$
```
> exit
```
exit
```

1. Restarting the container ultimately triggers the container's `entrypoint`, a set of commands and parameters that will be executed first when a container is run. Hereby we indirectly restart the keycloak server inside the container with the new configuration.
> docker restart 5d0d791a518b
```
5d0d791a518b
```

1. Jump again into the container to verify if the new configuration took effect and as expected find a folder `keycloak_full-export` at `/opt/jboss`, exit the container again
> docker exec -i -t 5d /bin/bash
```
[jboss@5d0d791a518b ~]$
```
>ls
```
build-keycloak.sh  docker-entrypoint.sh  keycloak  keycloak_full-export
```
>pwd
```
/opt/jboss
```
>exit
```
exit
```

1. Pull that folder from the filesystem of the container to our present working directory
>docker cp 5d0d791a518b:/opt/jboss/keycloak_full-export .  

1. We inspect the folder `keycloak_full-export` and see 2 json-files per realm
```
master-realm.json  master-users-0.json
```
whereby `master-realm.json` contains the metadata of the master realm and `master-users-0.json` the user information of the master realm.
The later can only be obtained with a full-export and is not included with a partial-export via keycloak admin-console UI or API.

1. Now we can do a full-import of the exported data to another keycloak server either
  * manually via [Administration-Console UI](http://localhost:9080/auth/) or
  * programmatically via the [Keycloak Admin REST API](https://www.keycloak.org/docs-api/4.3/rest-api/index.html).

### (D) Automation through scripting

1. We put the `embed-server` commands in a new file `keycloak_full-export.conf`
```
embed-server --server-config=standalone.xml
/system-property=keycloak.migration.action/:add(value=export)
/system-property=keycloak.migration.provider/:add(value=dir)
/system-property=keycloak.migration.dir/:add(value=keycloak_full-export)
```

1. Create a `Dockerfile` that uses the base image `jboss/keycloak:4.3.0.Final` that copies our `keycloak_full-export.conf` into the new image to then be used by the `jobss-cli` command.
```
FROM jboss/keycloak:4.3.0.Final
#ENV KEYCLOAK_USER=admin
#ENV KEYCLOAK_PASSWORD=admin
COPY keycloak_full-export.conf /opt/jboss/keycloak_full-export.conf
EXPOSE 8080
ENTRYPOINT [ "/opt/jboss/docker-entrypoint.sh" ]
CMD ["-b", "0.0.0.0"]
```
1. Building the docker image
> docker build -t keycloak_full-export:4.3.0.Final .
```
Sending build context to Docker daemon  4.096kB
Step 1/6 : FROM jboss/keycloak:4.3.0.Final
 ---> 75c249eec186
Step 2/6 : COPY keycloak_full-export.conf /opt/jboss/keycloak_full-export.conf
 ---> cef5240ea5f2
Step 3/6 : RUN /opt/jboss/keycloak/bin/jboss-cli.sh --file=keycloak_full-export.conf
 ---> Running in 726c11f021b4
{"outcome" => "success"}
{"outcome" => "success"}
{"outcome" => "success"}
Removing intermediate container 726c11f021b4
 ---> 0f34afa3dd8d
Step 4/6 : EXPOSE 8080
 ---> Running in 70d8a4873da0
Removing intermediate container 70d8a4873da0
 ---> 65161e9098e1
Step 5/6 : ENTRYPOINT [ "/opt/jboss/docker-entrypoint.sh" ]
 ---> Running in 670aacb97d3f
Removing intermediate container 670aacb97d3f
 ---> 99978a7b089a
Step 6/6 : CMD ["-b", "0.0.0.0"]
 ---> Running in c6b9c1c6cec7
Removing intermediate container c6b9c1c6cec7
 ---> 746bba026f2c
Successfully built 746bba026f2c
Successfully tagged keycloak_full-export:4.3.0.Final
```
> docker images
```
REPOSITORY             TAG                 IMAGE ID            CREATED             SIZE
keycloak_full-export   4.3.0.Final         746bba026f2c        31 seconds ago      801MB
```
1. Derive and start 2 keycloak servers from that image with the following `docker-compose.yml` and
```
version: "3.5"
services:
  kc1:
    image: keycloak_full-export:4.3.0.Final
    environment:
      KEYCLOAK_USER: admin
      KEYCLOAK_PASSWORD: admin
      DB_VENDOR: H2
    ports:
      - 9080:8080
  kc2:
    image: keycloak_full-export:4.3.0.Final
    environment:
      KEYCLOAK_USER: admin
      KEYCLOAK_PASSWORD: admin
      DB_VENDOR: H2
    ports:
      - 9081:8080
```
> docker-compose up -d
```
Creating automated_kc1_1 ... done
Creating automated_kc2_1 ... done
Attaching to automated_kc1_1, automated_kc2_1
...
kc2_1  | INFO  [org.jboss.as] (Controller Boot Thread) WFLYSRV0025: Keycloak 4.3.0.Final (WildFly Core 3.0.8.Final) started in 46638ms
kc1_1  | INFO  [org.jboss.as] (Controller Boot Thread) WFLYSRV0025: Keycloak 4.3.0.Final (WildFly Core 3.0.8.Final) started in 48920ms
```

5. Both docker containers (each containing a Keycloak server) are now up and running
>docker ps
```
CONTAINER ID        IMAGE                              COMMAND                  CREATED             STATUS              PORTS                    NAMES
2d0fd1afa786        keycloak_full-export:4.3.0.Final   "/opt/jboss/docker-e…"   5 minutes ago       Up 55 seconds       0.0.0.0:9081->8080/tcp   automated_kc2_1
c9517c0cc94d        keycloak_full-export:4.3.0.Final   "/opt/jboss/docker-e…"   5 minutes ago       Up 53 seconds       0.0.0.0:9080->8080/tcp   automated_kc1_1
```

6. We can now pull the keycloak data from the docker container filesystem to the filesystem of our host OS with
> docker cp [CONTAINER-ID]:/opt/jboss/keycloak_full-export .


7. And then perform a full-import with that data to another keycloak server by going the
  * manual route - [Administration-Console UI](http://localhost:9080/auth/) or
  * programmatical route - the [Keycloak Admin REST API](https://www.keycloak.org/docs-api/4.3/rest-api/index.html).

Hope this helps!
