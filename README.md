# varnish-httpd

[Running Varnish on Docker](https://www.varnish-software.com/developers/tutorials/running-varnish-docker/)

Varnish can be run inside a Docker container and Varnish Software maintains official Docker images for both Varnish Cache 6.0 LTS and for fresh releases of Varnish Cache.

[View the official Varnish images on the Docker Hub](https://hub.docker.com/_/varnish)

1. Select the right version

   We advise you to install **Varnish Cache 6.0 LTS**, which is a stable and supported version. It is maintained by _Varnish Software_ and receives frequent updates, including backported features.

   The _Varnish Cache community_ does 2 releases per year, which are considered _fresh releases_. These releases are primarily feature-based and do not guarantee backward compatibility. _Varnish Cache 6.6_ and _Varnish Cache 7.0_ are the current community-managed releases.

   The various tagged versions of the Docker images reflect these versions:

   * The `varnish:stable` and `varnish:6.0` tags refer to _Varnish Cache 6.0 LTS_
   * The `varnish:fresh` and `varnish:latest` tags refer to the latest _fresh_ version of Varnish Cache.
   * The `varnish:fresh-alpine` and `varnish:alpine` tags refer to an _Alpine-based_ image that runs the latest _fresh_ version of Varnish Cache.

   You can already download the Varnish image to your server before running the container. Please run the command below to download the Docker image for _Varnish Cache 6.0 LTS_:

   ```
   docker pull varnish:stable
   ```

2. Create a VCL file

   Unlike traditional Varnish setups where the software needs to be installed first, the Docker images allow you to spin up containers that have a full Varnish setup in a matter of seconds.

   This means you can focus on your VCL, rather than the stack running your VCL code. Here’s the absolute minimum amount of VCL you need to connect to your origin server:

   ```
   vcl 4.1;

   backend default {
       .host = "backend.example.com";
       .port = "80";
   }
   ```

   The backend definition in the VCL file refers to the origin server for which we are caching HTTP resources. The `.host` property contains the hostname of the origin and the `.port` property contains the port number on which your origin is listening for incoming HTTP connections.

   For real-world applications, there’s probably some more VCL required that depends on the type of application you’re trying to accelerate.

   [View an example VCL template](https://www.varnish-software.com/developers/tutorials/example-vcl-template/)

   [Learn about Varnish integrations for popular frameworks and platforms](https://www.varnish-software.com/developers/tutorials/#integrations)

   Store your VCL code in a file on your host system, we will mount it into the container in a later step.

3. Running the Varnish Docker container

   The following command will spin up a Varnish Docker container:

   ```
   docker run -v /path/to/default.vcl:/etc/varnish/default.vcl:ro \
    --tmpfs /var/lib/varnish:exec \
    --name my-varnish-container \
    -p 8080:80 \
    -e VARNISH_SIZE=2G \
    varnish:stable
   ```

   Let’s break this command down and explain the various parameters into the subsections below.

   Selecting the version tag

   The `varnish:stable` value refers to `stable` tag of the `varnish` images. It implies that Docker only uses the stable version of Varnish, which is _Varnish Cache 6.0 LTS_.

   Mounting the VCL file

   The `-v` option is used to mount volumes into your Docker container. These are shared filesystems that are used to share files from the host system.

   The `-v /path/to/default.vcl:/etc/varnish/default.vcl:ro` definition will mount `/path/to/default.vcl` from your host system into `/etc/varnish/default.vcl` in your container.

   The `:ro` option will only allow read-only access to the VCL file inside the Docker container.

   Loading /var/lib/varnish into memory

   The `/var/lib/varnish` folder is frequently accessed by the `varnishd` program. Loading this folder into memory and accessing it through `tmpfs` would accelerate access to this folder.

   `--tmpfs /var/lib/varnish:exec` takes care of this. The `:exec` option gives container execution permissions on that folder.

   Naming the container

   The `--name my-varnish-container` option will assign `my-varnish-container` as the name of the container. If no `--name` option is added, Docker will generate a random name.

   Exposing the listening port

   The `-p 8080:80` option will expose port `80` from the container as port `8080` on the host system. This allows users to access Varnish by calling the `http://localhost:8080` endpoint on the host system.

   Setting the cache size

   The standard size of the cache is _100 MB_. The `VARNISH_SIZE` environment variable is used to override this value. Environment variables are set using the `-e` option in Docker.

   This means that the `-e VARNISH_SIZE=2G` option from the example above will set the cache size to _2 GB_.

4. Additional parameters

   The Varnish Docker image is designed in such a way that extra varnishd runtime parameters can be added.

   Any option that is added after the image, will be attached as a varnishd runtime parameter.

   For example docker run varnish:stable -p default_ttl=3600 will assign a -p option to the varnishd program. This allows you to extend Varnish’s runtime parameters.

   -p default_ttl=3600 wil set the standard Time-To-Live of Varnish to an hour, whereas the standard value is 2 minutes.

   [More information about varnishd runtime parameters](http://varnish-cache.org/docs/6.0/reference/varnishd.html#options)

5. Running commands

   We can use docker exec to execute commands on our running Varnish container.

   The following access gives you access to the Bash shell of the my-varnish-container container:

   ```
   docker exec -ti my-varnish-container bash
   ```

   This allows you to execute commands like varnishreload, varnishadm, varnishlog, varnishncsa and various other commands on the system.

   It is also possible to access these commands directly without going through the Bash shell.

   Here’s an example where we reload the VCL configuration using a single command:

   ```
   docker exec my-varnish-container varnishreload
   ```

   The following command will interactively run the varnishlog -g request -q "ReqURl eq '/'" command:

   ```
   docker exec -ti my-varnish-container varnishlog -g request -q "ReqURl eq '/'"
   ```

   This command will display Varnish Shared Memory Logs in real-time, group the output by request and filter requests for the homepage.

6. Docker Compose

   If you’re planning to orchestrate a multi-container setup using Docker Compose, here’s an example of a docker-compose.yml file that contains a Varnish configuration:

   ```
   version: "3"
   services:
     varnish:
       image: varnish:stable
       container_name: varnish
       volumes:
         - "./default.vcl:/etc/varnish/default.vcl"
       ports:
         - "80:80"
       tmpfs:
         - /var/lib/varnish:exec
       environment:
         - VARNISH_SIZE=2G
       command: "-p default_keep=300"
       depends_on:
         - "httpd"
     httpd:
       image: httpd:latest
       container_name: httpd
       ports:
         - "8080:80"
   ```

   This docker-compose.yml file will spin up 2 containers:

   * A Varnish container named varnish
   * An Apache container named httpd

   The Varnish container

   The Varnish container uses the varnish:stable image that runs Varnish Cache 6.0 LTS.

   HTTP port 80 will be exposed to the host system and the /var/lib/varnish folder from the container will be mounted through tmpfs. This means the directory will be a separate RAM disk volume with execute permissions.

   Through the VARNISH_SIZE environment variable, we’re extending the size of the cache to 2 GB.

   The command: "-p default_keep=300" statement allows us to attach extra parameters to the varnish program and override some runtime parameters. In this case we’re setting the value of the default_keep runtime parameter to 300 seconds.

   The VCL file

   The default.vcl file from the host system will be mounted into the Varnish container and will be available through /etc/varnish/default.vcl.

   Here’s what the VCL file could look like:

   ```
   vcl 4.1;

   backend default {
       .host = "httpd";
       .port = "80";
   }
   ```

   The httpd hostname in the VCL backend refers to the Apache container where Varnish will proxy requests to.

   The Varnish container depends on the Apache container. The depends_on syntax will ensure the Varnish container is created after the Apache container. Otherwise the httpd hostname would not be resolvable in VCL.

   The Apache container

   The Apache container itself is pretty basic: it uses the httpd:latest images.

   Its HTTP port is exposed to the host system as port 8080. Internally, port 80 is still used, as you can see in the VCL example above.

   The Apache webserver acts as the origin and its contents will be cached by Varnish.

   Starting the containers

   To set up our Varnish and Apache stack using Docker Compose, run the following command:

   ```
   docker compose up
   ```

   This command will read the docker-compose.yml file in the current directory and start the containers based on the configuration.
   Access the containers

   After the environment has been started, you can access Varnish through the http://localhost endpoint on your host system. If you want access the Apache webserver directly you can use the http://localhost:8080 endpoint.

   To access the bash shell of the Varnish container, simply run the following command:

   ```
   docker exec -ti varnish bash
   ```

   This will allow you to use tools like varnishadm, varnishstat and varnishlog.

   Cleaning up

   When you’re done running this Docker Compose environment, you can tear it down using a single command:

   ```
   docker compose down
   ```

