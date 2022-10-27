# Simple Docker Proxy

Tutorial to demonstrate a docker based http proxy

## Basic Proxy Setup

To start, let's create three directories to hold our different files. If you've cloned
this repo you'll wan't to do this in a directory from the repo.

```bash
mkdir server-a
mkdir server-b
mkdir proxy
```

Next we'll create the two servers images we're going to use. First we'll create two index.html
files in the server-a and server-b directories. These index.html files will be used inside
our custom web server docker containers.

```bash
echo "Hello World from Server A!" >> server-a/index.html
echo "Hello World from Server B!" >> server-b/index.html
```

Now let's create a Dockerfile for each server.

```bash
cd server-a
touch Dockerfile
```

Edit the file so it contains the following code. Note we're using the latest nginx image as our
base, then copying the index.html file we created previously into the web servers content directory.

```dockerfile
FROM nginx:latest
COPY index.html /usr/share/nginx/html
```

Now we can build and test our server images. First we'll build the servera image. Note that
we're giving this image a tag of `servera:1.0` which is a combination of a name `servera` and a
version `1.0`

```bash
sudo docker build -t servera:1.0 .
```

Once this image has been built we can run the image and test it with a web browser. We'll
pass in the `-p` argument to tell docker to map port `8080` on our local machine to port `80`
of the docker container.

```bash
sudo docker run -p 8080:80 servera:1.0
```

Once the container is running open a web-browser and navigate to `http://localhost:8080` you should
see the message "Hello World from Server A!". Use Ctrl+C to kill the docker container.

Now let's repeat that same docker build step but from the server-b directory to build the
serverb image.

```bash
cd ..
cd server-b
touch Dockerfile
```

Edit the file so it contains the following code. Note this is the exact same docker file contents
as server a

```dockerfile
FROM nginx:latest
COPY index.html /usr/share/nginx/html
```

Build the container image for serverb

```bash
sudo docker build -t serverb:1.0 .
```

Again, we can test this using a docker run command. We should see the message "Hello World from Server B!" in our web browser now.

```bash
sudo docker run -p 8080:80 servera:1.0
```

At this point we have our two web server images, but we've been manually running them one at a time. What
we need now is a *proxy* service that can take a request from a web browser and route it to one of the
web servers automatically. Fortunately for us, nginx can also act as a proxy server!

First we need to create our nginx configuration file:

```bash
cd ..
cd proxy
touch nginx.conf
```

Edit the `nginx.conf` file in your favorite editor. First we need to define a `server` groups which
tells nginx how to configure itself. Since we want this server to act as a proxy we will tell the server
that for every request (`location /`) proxy the traffic to the url `http://loadbalancer`

```conf
server {
  location / {
    proxy_pass http://loadbalancer;
  }
}
```

Next we need to define what `loadbalancer` should resolve to. For this we need to add a `upstream`
declaration that will include the list of servers to proxy traffic to.

```conf
upstream loadbalancer {
  server servera:80 weight=6;
  server serverb:80 weight=4;
}
```

Our complete nginx.conf file should now look like this:

```conf
server {
  location / {
    proxy_pass http://loadbalancer;
  }
}

upstream loadbalancer {
  server servera:80 weight=6;
  server serverb:80 weight=4;
}
```

Next we need to create the Dockerfile for our proxy service:

```bash
touch Dockerfile
```

Edit the dockerfile contents to be:

```dockerfile
FROM nginx:latest
RUN rm /etc/nginx/conf.d/default.conf
COPY nginx.conf /etc/nginx/conf.d/default.conf
```

Next build the proxy service container image:

```bash
sudo docker build -t proxy:1.0 .
```

At this point we now have three docker image files:

- proxy:1.0
- servera:1.0
- serverb:1.0

While we could manually start and configure each container manually using `docker run` commands
it's much easier to use `docker-compose` to coordinate the starting/stopping for us.

First we need to create our compose file:

```bash
touch docker-compose.yml
```

Next edit the docker-compose.yml file to contain the following items:

```yml
version: "3"

services:
  nginx-proxy:
    image: nginxproxy
    depends_on:
      - servera
      - serverb
    ports:
      - "8080:80"

  servera:
    image: servera:1.0

  serverb:
    image: serverb:1.0
```

> :exclamation: One important thing to note here is that the list of items under the `services` entry is the name that will be given to the docker containers and registered in the internal docker DNS system. The names of servera and serverb must match the names in the `nginx.conf` file since nginx will use DNS to resolve the host when performing a proxy.

Once we have the docker-compose.yml file created we can start all of the services with a single command

```bash
sudo docker-compose up -d
```

Once the services have started you can navigate to `http://locahost:8080` to see the output.
If you refresh your browser a few times you should see it toggle between the a/b server outputs.
You can also utilize the `poll.sh` script in the repo to continuously poll the servers.

```text
./poll.sh
Hello From Server A!
Hello From Server B!
Hello From Server A!
Hello From Server B!
Hello From Server A!
Hello From Server A!
Hello From Server B!
Hello From Server B!
Hello From Server A!
Hello From Server B!
Hello From Server A!
Hello From Server A!
Hello From Server B!
Hello From Server A!
Hello From Server B!
```

Finally lets stop the containers:

```bash
sudo docker-compose down
```

## Live Container Updates

Now that we've gotten the basic proxy working, we explore how this can provide us with a zero-downtime
service.

First lets go back to server-a and modify the index.html so it says:

```text
Hello World from Service A, Version 2!
```

Once modified we need to re-build the container image, but this time we'll give it a different
version tag.

```bash
sudo docker build -t servera:2.0 .
```

To confirm it's working we can test it the same way we did originally:

```bash
sudo docker run -p 8080:80 servera:2.0
```

Once we've confirmed the `servera:2.0` image is outputting the correct text we can go back to our
proxy folder and startup the services using docker compose. Note we're not changing anything in the compose
file just yet:

```bash
sudo docker compose up -d
```

Running the poll.sh command (or using a web browser) we should see the output switching back and forth
from server a and server b.

Now lets modify the docker-compose.yml file and change the image tag for server a from `:1.0` to `:2.0`

```yml
 servera:
    image: servera:2.0
```

Once saved we can perform a live update to the services by simply repeating the docker compose command:

> To see the results live run the ./poll.sh script in a secondary bash window

```bash
sudo docker-compose up -d
```

Now when looking at the output we should see the results switch from the old servera output to the new
server a output

```text
Hello From Server A!
Hello From Server B!
Hello From Server A!
Hello From Server B!
Hello From Server A!
Hello From Server A!
Hello From Server B!
Hello From Server B!
Hello From Server B!
Hello From Server B!
Hello From Server B!
Hello From Server B!
Hello From Server A, Version 2!
Hello From Server B!
Hello From Server A, Version 2!
Hello From Server A, Version 2!
Hello From Server B!
Hello From Server A, Version 2!
Hello From Server B!
Hello From Server A, Version 2!
Hello From Server A, Version 2!
Hello From Server B!
Hello From Server A, Version 2!
```

Finally lets stop the containers:

```bash
sudo docker-compose down
```
