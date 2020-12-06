---
layout: post
title:  "Docker, protect your sensitive information with secrets"
subtitle: "A short but concrete demonstration of the use of secrets within Docker to protect sensitive information that one does not wish to store in an unsecured way."
header-img: "/assets/pictures/banner.jpg"
author: "Loris Bergeron"
tags: docker security devops
---

As simple as the use of Docker may be once you have discovered this world, there is however one aspect not to be neglected: *security*. 

Docker provides us with a variety of techniques to enable us to manage sensitive information simply but effectively. This is what we call secrets.

In Docker jargon, a secret is a set of data, such as a password, an SSH private key, an SSL certificate or any other information that you want to keep secret and therefore not store it unencrypted. You can use secrets to manage all the sensitive information that a container asks for at runtime but that you don't want us to store in an image or in your source control system. For example: 

- Usernames and passwords
- TLS certificates and keys
- SSH keys
- Other important data such as the name of a database or internal server
- Generic strings or binary content (up to 500 kb in size)

Through this article we will go through three concrete examples in which we will use this security layer. 

Note: Docker secrets are only available to swarm services, not to standalone containers. To use this feature, consider adapting your container to run as a service. Stateful containers can typically run with a scale of 1 without changing the container code.

### Useful commands to know

The commands below will be very useful and represent the basics of the secrets in Docker:

- [docker secret create](https://docs.docker.com/engine/reference/commandline/secret_create/)
- [docker secret inspect](https://docs.docker.com/engine/reference/commandline/secret_inspect/)
- [docker secret ls](https://docs.docker.com/engine/reference/commandline/secret_ls/)
- [docker secret rm](https://docs.docker.com/engine/reference/commandline/secret_rm/)

### An Introduction to secrets

In this first part the idea is to create very simply our first secret and use some of the commands shared above. As previously announced, secrets can only be used in a swarm operating mode. [More information here.](https://docs.docker.com/engine/swarm/)

1. The first step is to initialize the swarm.

    ```shell
    $ docker swarm init
    ```

2. Add a secret to Docker. The `docker secret create` command is there for that and reads the standard input to set the value of the secret.

    ```shell
    $ echo "This is my first secured secret" | docker secret create my_first_secured_secret -
    ```

    Please note the obligatory presence of the `-` at the end of the command. If the command was correctly performed Docker returns a random value that corresponds to the ID of the newly created secret.

3. To check the correct creation and existence of secret, the command `docker secret ls` is at our disposal for this purpose.

    ```shell
    $ docker secret ls

    ID                          NAME                      DRIVER              CREATED             UPDATED
    4wnfhom9exyvjsff5rlrxfl99   my_first_secured_secret                       2 minutes ago       2 minutes ago
    ```

4. If you want to know more about our secret, the `docker inspect` method is useful.

    ```shell
    $ docker secret inspect my_first_secured_secret

    [
        {
            "ID": "4wnfhom9exyvjsff5rlrxfl99",
            "Version": {
                "Index": 11
            },
            "CreatedAt": "2020-12-05T20:24:08.0251829Z",
            "UpdatedAt": "2020-12-05T20:24:08.0251829Z",
            "Spec": {
                "Name": "my_first_secured_secret",
                "Labels": {}
            }
        }
    ]
    ```

### Using a secret in a Docker service

Let's say we want to create a nginx service and it uses our secret to keep it running smoothly.

1. Create a nginx service and give it access to our secret. By default, the container accesses the secret via `/run/secrets/<secret_name>`.

    ```shell
    $ docker service create --name nginx --secret my_first_secured_secret nginx:latest
    ```

2. Verify that the service is working properly by using docker service ps. If everything works you should with something like:

    ```shell
    $ docker service ps nginx

    ID                  NAME                IMAGE               NODE                DESIRED STATE       CURRENT STATE           ERROR               PORTS
    q8gubtj1tvl4        nginx.1             nginx:latest        docker-desktop      Running             Running 8 seconds ago
    ```

3. Now that we know the service is running we can connect to the container and execute a command to read the content of the secret it uses.

    ```shell
    $ docker ps --filter name=nginx -q

    455b3d931bbb

    $ docker container exec 455b3d931bbb cat /run/secrets/my_first_secured_secret

    "This is my first secured secret"
    ```

4. If we try to remove our secret, it won't work because the nginx service is currently using it.

    ```shell
    $ docker secret rm my_first_secured_secret

    Error response from daemon: rpc error: code = InvalidArgument desc = secret 'my_first_secured_secret' is in use by the following service: nginx
    ```

5. The only way to get rid of our secret is to update our nginx service so that it no longer uses it.

    ```shell
    $ docker service update --secret-rm my_first_secured_secret nginx
    ```

6. To check if our service is still using our secret, we can use again the command used previously.

    ```shell
    $ docker ps --filter name=nginx -q

    3bdbd30cb9f3

    $ docker container exec 3bdbd30cb9f3 cat /run/secrets/my_first_secured_secret

    cat: /run/secrets/my_first_secured_secret: No such file or directory
    ```

7. The example of a secret in a Docker service is now over, we can move on to the removal of the service as well as the secret.

    ```shell
    $ docker stop 3bdbd30cb9f3
    $ docker rm 3bdbd30cb9f3
    $ docker secret rm my_first_secured_secret
    ```

### Using a secret in a docker-compose.yml file

The following example is a further use of the possibilities offered by Docker's secrets when deploying an application in a swarm using a `docker-compose.yml` file. Same configuration as before we have to be within the framework of the use of a swarm. 

For this simple example we assume that we want to deploy the Gitea application using Docker (which by the way is a very nice project that I invite you to find here --> [Gitea on GitHub](https://github.com/go-gitea/gitea)). 

1. Let's join a Swarm Docker before anything else. If you have already initialized a swarm in the past you will most likely encounter an error informing you that you are already part of a swarm. If you want to leave it you can use the [`docker swarm leave`](https://docs.docker.com/engine/reference/commandline/swarm_leave/) command. 

    ```shell
    $ docker swarm init
    ```

2. Let's create two secrets dedicated to our MySQL database for Gitea.

    ```shell
    $ echo "This is my root password" | docker secret create gitea_mysql_root_password -
    $ echo "This is my user password" | docker secret create gitea_mysql_user_password -
    ```

3. Creation of the docker-composer file dedicated to the deployment of the Gitea application:

    ```yaml
    version: "3.3"

    networks:
      nw:
        external: false

    services:
      app:
        image: gitea/gitea:1.13.0
        restart: always
        container_name: gitea_app
        environment:
          - USER_UID=1000
          - USER_GID=1000
          - DB_TYPE=mysql
          - DB_HOST=db:3306
          - DB_NAME=gitea
          - DB_USER=gitea
          - DB_PASSWD=/run/secrets/gitea_mysql_user_password # The value is no longer a clear text
        volumes:
          - app_data:/data
          - /etc/timezone:/etc/timezone:ro
          - /etc/localtime:/etc/localtime:ro
        ports:
          - "3000:3000"
          - "222:22"
        secrets: # Section to indicate to the app service the secrets to be used
          - gitea_mysql_user_password 
        networks:
          - nw
        depends_on:
          - db

      db:
        image: mysql:5.7
        restart: always
        container_name: gitea_db
        environment:
          - MYSQL_ROOT_PASSWORD=/run/secrets/gitea_mysql_root_password # The value is no longer a clear text
          - MYSQL_USER=gitea
          - MYSQL_PASSWORD=/run/secrets/gitea_mysql_user_password # The value is no longer a clear text
          - MYSQL_DATABASE=gitea
        volumes:
          - db_data:/var/lib/mysql
        secrets: # Section to indicate to the db service the secrets to be used
          - gitea_mysql_root_password
          - gitea_mysql_user_password
        networks:
          - nw

    volumes:
      app_data:
      db_data:
        
    secrets: # Section that indicates the secrets used throughout the yml file
      gitea_mysql_root_password:
        external: true # If set to true, specifies that this secret has already been created. Docker does not attempt to create it, and if it does not exist, a secret not found error occurs
      gitea_mysql_user_password:
        external: true
    ```

    Save this file and give it the name `docker-compose.yml`.

4. All that remains is to launch the `docker stack deploy` command and that's it:

    ```shell
    $ docker stack deploy --compose-file docker-compose.yml gitea
    ```

    You can find more information about using the `docker stack deploy` command directly in the [Docker's stack deploy documentation](https://docs.docker.com/engine/reference/commandline/stack_deploy/).