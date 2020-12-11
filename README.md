# Deploy Pancake Swap interface on Akash



![title picture](https://github.com/yuravorobei/pancake-swap-on-akash-guide/blob/main/media/main.png)


  In this guide for the "Akashian Challenge: Phase 3 Start Guide" I'll show you how to deploy the [Pancake Swap interface](https://exchange.pancakeswap.finance) on the [Akash Decentralized Cloud (DeCloud)](https://akash.network/).  

 **PancakeSwap** is a decentralized exchange running on Binance Smart Chain, with lots of other features that let you earn and win tokens. 
It's fast, cheap, and anyone can use it. The exchange is automated market maker (“AMM”) that allows two tokens to be exchanged on the Binance Smart Chain. On top of that, you can earn CAKE with yield farms, earn SYRUP and CAKE with Staking, and earn even more tokens with SYRUP pools.
 
  **Akash** is a marketplace for cloud compute resources which is designed to reduce waste, thereby cutting costs for consumers and increasing revenue for providers. Akash DeCloud is a faster, better, and lower cost cloud built for DeFi, decentralized projects, and high growth companies, providing unprecedented scale, flexibility, and price performance. 10x lower in cost, Akash serverless computing platform is compatible with all cloud providers and all applications that run on the cloud.

This guide is divided into three main parts: dockerizing, installing and preparing Akash, deploying. All actions I perform on a Ubuntu 18.04 operating system. And so we go.

# 1. Dockerizing
The Akash works with a Docker images, so the first thing we need to do to deploy is to get the Docker image of the PancakeSwap interface application. If you already have the image or know how to get it, you can skip this step.


### 1.1 Source code
First, you need to get the source code of the PancakeSwap interface application on your computer. To do this, clone the [PancakeSwap interface GitHub repository](https://github.com/pancakeswap/pancake-swap-interface):
  ```bash
  $ git clone https://github.com/pancakeswap/pancake-swap-interface.git
  $ cd pancake-swap-interface
  ```
Then you need to create a `.env` file with environment variables that determine on which network the application will run - Mainnet or Testnet. The developers have provided sample files for both networks: `.env.production` for the Mainnet and `.env.development` for the Testnet. We just need to copy the one we need and save it under the name `.env`, I deploy the application for the Mainnet:
  ```bash
  $ cp .env.production .env
  ```

### 1.2 Dockerfile
Docker can build images automatically by reading the instructions from a Dockerfile. A Dockerfile is a text document that contains all the commands a user could call on the command line to assemble an image. 
First, you need to build the project and then deploy it to a web server as static resources. For this we will split the Dockerfile in multiple stages by using the Docker multi-stage builds feature. 

Let's create the Dockerfile:
  ```bash
  $ nano Dockerfile
  ```

And put the following code in it:
  ```dockerfile
  # ------- BUILD STAGE --------
  FROM node:lts-alpine as build-stage

  # set the current working directory
  WORKDIR /app

  # copy all project files and folders to the current working directory
  COPY . .

  # adding the git package needed to install the dependencies
  RUN apk add git

  # install project dependencies for production version
  RUN yarn install

  # build app for production with minification
  RUN yarn build


  # ------- SERVING STAGE --------
  FROM nginx:alpine as serving-stage
  
  # copy all project files and folders to the project directory
  COPY . /app/
  
  # copy final project build files
  COPY --from=build-stage /app/build/ /app/

  # copy Nginx configuration file
  COPY nginx.conf /etc/nginx/nginx.conf

  # set Nginx production mode to improve performance and security
  ARG NGINX_MODE=prod

  # Expose the container listening port
  EXPOSE 80

  # run Nginx in the foreground
  CMD ["nginx", "-g", "daemon off;"]
  ```
  
The first stage is responsible for building a production-ready artifact of app. I am using the `node:lts-alpine` image as a base image, this is a very compact image that will have a beneficial effect on the size of the container. This step installs the dependencies and builds the project into the `build` directory.

At the second stage, you need to organize hosting for the newly created assembly. I recommend using Nginx as a web server for the production version of the application. Nginx is a fast web server that has the best performance in this case and supports routing, static content, etc., in an objectively faster time to provide the greater user experience. 

As a base image used here Alpine-version of `nginx: alpine`, it is a lightweight image that will save space. 
Next, the project build results obtained in the previous step are copied to the working directory and the `ngix.conf` file, which will be created in the next step of the tutorial, copied to the Nginx configuration directory. Then we open port `80`, you can specify any other. The container will wait for connections on this port to web-server and the same port will need to be specified further in the web server configuration. The last line of the file is used to start Nginx in foreground so that the Docker container does not close immediately after starting.


### 1.3 nginx.conf
For the Nginx web server to work correctly, you need to create a configuration file:
  
  ```bash
  $ nano nginx.conf
  ```

And put the following code in it:
  ```Nginx
  user  nginx;
  worker_processes  1;

  events {
    worker_connections  1024;
  }

  http {
    include /etc/nginx/mime.types;
    default_type  application/octet-stream;
    client_max_body_size 100m;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
    '$status $body_bytes_sent "$http_referer" '
    '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main; # mapped to stdout/stderr
    sendfile on;
    tcp_nopush on;
    keepalive_timeout 65;
    gzip  on;
    charset utf-8;
    proxy_read_timeout 1000s;

    server {
      listen  80;
      charset utf-8;
      root    /app;

      location / {
        index index.html;
        try_files $uri $uri/ /index.html;
      }
    }
  }
  ```
This configuration is written based on standard examples, you can add or change any configurations you need.
Currently the most important fields here are `http.server.listen` is the port that the web server will listen to (in my case port `80`) and` http.server.root` is the application working directory (`/app`).


### 1.4 .dockerignore
There are additional files and folders in the project directory that do not need to be added to the Docker image to save weight and time to build the image. To do this, a file named `.dockerignore` is used, which lists all files and directories that will be ignored when building the image. Let's create the file:
  ```bash
  $ nano .dockerignore
  ```
  And put the following strings in it:
  ```
  .git
  .github
  .gitignore
  README.md
  listing.md
  .env.development
  .env.production
  Dockerfile
  .dockerignore
  ```


### 1.5 Building the Docker image
To build the Docker image run the following command:
  ```bash
  $ docker build -t pancake-swap .
  ```
Any other name can be used here instead of `pancake-swap`, most importantly, do not forget to also change this name in the following commands. Waiting for the completion of the function.


### 1.6 Publishing the image
To transfer the created image to the cloud server, you need to share it. To do this I will push the image to the [Docker Hub](https://hub.docker.com/). Log into the Docker Hub from the command line. Enter the following command, replace `<your_hub_username>` with your Docker Hub usesrname:
  ```bash
  $ docker login --username=<your_hub_username>
  ```
And enter your Docker Hub password. 

Then find the created image ID using `docker images` command:
  ```
  $ docker images

  REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
  pancake-swap        latest              02909a25a46a        8 minutes ago       43MB
  ```
In my case image ID is `02909a25a46a`.  
Then tag your image with the following command, replace `<image_id>` with your image ID and replace `<your_hub_username>` with your Docker Hub usesrname. Any name can be used instead of `pancake-swap`:
  ```bash
  $ docker tag <image_id> <your_hub_username>/pancake-swap2
  ```
Push your image to the Docker Hub repository with the following command:
  ```bash
  $ docker push <your_hub_username>/pancake-swap2
  ```
Waiting for loading.
Dockerization done!



# 2. Installing and preparing Akash
To interact with the Akash system, you need to install it and create a wallet. I will follow the official [Akash Documentation guides](https://docs.akash.network/v/master/).

### 2.1 Choosing a Network
At any given time, there are a number of different active Akash networks running, each with a different akash version, chain-id, seed hosts, etc… Three networks  are currently available: mainnet, testnet and edgenet. In this guide for the "Akashian Challenge: Phase 3 Start Guide" i'll use the `edgenet` network.

So let’s define the required shell variables:
  ```bash
  $ AKASH_NET="https://raw.githubusercontent.com/ovrclk/net/master/edgenet"
  $ AKASH_VERSION="$(curl -s "$AKASH_NET/version.txt")"
  $ AKASH_CHAIN_ID="$(curl -s "$AKASH_NET/chain-id.txt")"
  $ AKASH_NODE="$(curl -s "$AKASH_NET/rpc-nodes.txt" | shuf -n 1)"
  ```

### 2.2 Installing the Akash client
There are several ways to install Akash: from the [release page](https://github.com/ovrclk/akash/releases), from the source or via `godownloader`. As for me, the easiest and the most convenient way is to download via `godownloader`:
  ```bash
  $ curl https://raw.githubusercontent.com/ovrclk/akash/master/godownloader.sh | sh -s -- "$AKASH_VERSION"
  ```
And add the akash binaries to your shell PATH. Keep in mind that this method will only work in this terminal instance and after restarting your computer or terminal this PATH addition will be removed. You can read how to set your PATH permanently [on this page](https://stackoverflow.com/questions/14637979/how-to-permanently-set-path-on-linux-unix) if you want:
  ```bash
  $ export PATH="$PATH:./bin"
  ```


### 2.3 Wallet setting
Let's define additional shell variables. `KEY_NAME` value of your choice, I uses a value of “crypto”:
  ```bash
  $ KEY_NAME="crypto"
  $ KEYRING_BACKEND="test"
  ```
Derive a new key locally:
  ```bash
  $ akash --keyring-backend "$KEYRING_BACKEND" keys add "$KEY_NAME"
  ```

You'll see a response similar to below:
  ```yaml
  - name: crypto
    type: local
    address: akash1kfd3adu7sgcu2vxd3ucc3pehmrautp25kxenn5
    pubkey: akashpub1addwnpepqvc7x8r2dyula25ucxtawrt39henydttzddvrw6xz5gvcee4arfp7ppe6md
    mnemonic: ""
    threshold: 0
    pubkeys: []


  **Important** write this mnemonic phrase in a safe place.
  It is the only way to recover your account if you ever forget your password.

  share mammal walnut direct plug drive cruise real pencil random regret chunk use live entire gloom donate require autumn bid brown impact scrap slab
  ```
In the above example, your new Akash address is `akash1kfd3adu7sgcu2vxd3ucc3pehmrautp25kxenn5`, define a new shell variable with it:
  ```bash
  $ ACCOUNT_ADDRESS="akash1kfd3adu7sgcu2vxd3ucc3pehmrautp25kxenn5"
  ```

### 2.4 Funding your account
In this guide I use the edgenet Akash network. Non-mainnet networks will often times have a "faucet" running - a server that will send tokens to your account. You can see the faucet url by running:
  ```bash
  $ curl "$AKASH_NET/faucet-url.txt"

  https://akash.vitwit.com/faucet
  ```
Go to the resulting URL and enter your account address; you should see tokens in your account shortly.
Check your account balance with:
  ```yaml
  $ akash --node "$AKASH_NODE" query bank balances "$ACCOUNT_ADDRESS"

  balances: 
  - amount: "100000000" 
   denom: uakt 
  pagination: 
   next_key: null 
   total: "0"
  ```
As you can see, the balance is non-zero and contains 100000000 uakt.
Okay, now you're ready to deploy the application.



# 3. Deploying
Make sure you have the following set of variables defined on your shell in the previous step "2. Installing and preparing Akash":
  * `AKASH_CHAIN_ID` - Chain ID of the Akash network connecting to
  * `AKASH_NODE` - Akash network configuration base URL
  * `KEY_NAME` - The name of the key you will be deploying from
  * `KEYRING_BACKEND` - Keyring backend to use for local keys
  * `ACCOUNT_ADDRESS` - The address of your account

### 3.1 Creating a deployment configuration
For configuration in Akash uses [Stack Definition Language (SDL)](https://docs.akash.network/v/master/documentation/sdl) similar to Docker Compose files. Deployment services, datacenters, pricing, etc.. are described by a YAML configuration file. These files may end in .yml or .yaml.
Create `deploy.yml` configuration file:
  ```bash
  $ nano deploy.yml
  ```
With following content:  

  ```yaml
  version: "2.0"

  services:
    web:
      image: yuravorobei/pancake-swap2
      expose:
        - port: 80
          as: 80
          to:
            - global: true

  profiles:
    compute:
      web:
        resources:
          cpu:
            units: 0.1
          memory:
            size: 512Mi
          storage:
            size: 512Mi
    placement:
      westcoast:    
        pricing:
          web: 
            denom: uakt
            amount: 5000

  deployment:
    web:
      westcoast:
        profile: web
        count: 1
  ```
First, a service named `web` is created in which its settings are specified.
The `services.web.images` field is the name of your published Docker image. In my case it is `yuravorobei/pancake-swap2`.
The `services.expose.port` field means the container port to expose. I have specified listening port `80` in my Nginx settings.
The `services.expose.as` field is the port number to expose the container port as specified. Port `80` is the standard port for web servers HTTP protocol.  

Next section `profiles.compute` is map of named compute profiles. Each profile specifies compute resources to be leased for each service instance uses uses the profile. This defines a profile named `web` having resource requirements of 0.1 vCPUs, 512 megabytes of RAM memory, and 512 megabytes of storage space available. `cpu.units` represent a virtual CPU share and can be fractional.

`profiles.placement` is map of named datacenter profiles. Each profile specifies required datacenter attributes and pricing configuration for each compute profile that will be used within the datacenter. This defines a profile named `westcoast` having required attribute `pricing` with a max price for the `web` compute profile of 5000 uakt per block. 

The `deployment` section defines how to deploy the services. It is a mapping of service name to deployment configuration.
Each service to be deployed has an entry in the deployment. This entry is maps datacenter profiles to compute profiles to create a final desired configuration for the resources required for the service.


### 3.2 Creating the deployment
In this step, you post your deployment, the Akash marketplace matches you with a provider via auction. To deploy on Akash, run:
  ```bash
  $ akash tx deployment create deploy.yml --from $KEY_NAME \
    --node $AKASH_NODE --chain-id $AKASH_CHAIN_ID \
    --keyring-backend $KEYRING_BACKEND -y
  ```
Make sure there are no errors in the command response. The error information will be in the `raw_log` responce field.


### 3.3 Wait for your lease
You can check the status of your lease by running:
  ```
  $ akash query market lease list --owner $ACCOUNT_ADDRESS \
    --node $AKASH_NODE --state active
  ```
  You should see a response similar to:
  ```yaml
  - lease_id:
      dseq: "192910"
      gseq: 1
      oseq: 1
      owner: akash1kfd3adu7sgcu2vxd3ucc3pehmrautp25kxenn5
      provider: akash15ql9ycjkkxhpc2nxtnf78qqjguwzz8gc4ue7wl
    price:
      amount: "1310"
      denom: uakt
    state: active
  pagination:
    next_key: null
    total: "0"
  ```

From this response we can extract some new required for future referencing shell variables:
  ```bash
  $ PROVIDER="akash15ql9ycjkkxhpc2nxtnf78qqjguwzz8gc4ue7wl"
  $ DSEQ="192910"
  $ OSEQ="1"
  $ GSEQ="1"
  ```


### 3.4 Uploading manifest
Upload the manifest using the values from above step:
  ```bash
  $ akash provider send-manifest deploy.yml \
    --node $AKASH_NODE --dseq $DSEQ \
    --oseq $OSEQ --gseq $GSEQ \
    --owner $ACCOUNT_ADDRESS --provider $PROVIDER
  ```
Your image is deployed, once you uploaded the manifest. You can retrieve the access details by running the below:
  ```bash
  $ akash provider lease-status \
    --node $AKASH_NODE --dseq $DSEQ \
    --oseq $OSEQ --gseq $GSEQ \
    --provider $PROVIDER --owner $ACCOUNT_ADDRESS
  ```
  You should see a response similar to:
  ```json
  {
    "services": {
      "web": {
        "name": "web",
        "available": 1,
        "total": 1,
        "uris": [
          "6g64rsv4ihcp52r0cs9l3tnv7g.provider4.akashdev.net"
        ],
        "observed-generation": 0,
        "replicas": 0,
        "updated-replicas": 0,
        "ready-replicas": 0,
        "available-replicas": 0
      }
    },
    "forwarded-ports": {}
  }
```
You can access the application by visiting the hostnames mapped to your deployment. In above example, its http://6g64rsv4ihcp52r0cs9l3tnv7g.provider4.akashdev.net. Following the address, make sure that the application works:
![result1](https://github.com/yuravorobei/pancake-swap-on-akash-guide/blob/main/media/deployed.png)

### 3.5 Service Logs
You can view your application logs to debug issues or watch progress using `akash provider service-logs` command, for example:
```
  $ akash provider service-logs --node "$AKASH_NODE" --owner "$ACCOUNT_ADDRESS" \
  --dseq "$DSEQ" --gseq 1 --oseq $OSEQ --provider "$PROVIDER" \
  --service web \
```

You should see a response similar to this:
```
[web-86b97b6944-7ggt2] /docker-entrypoint.sh: /docker-entrypoint.d/ is not empty, will attempt to perform configuration
[web-86b97b6944-7ggt2] /docker-entrypoint.sh: Looking for shell scripts in /docker-entrypoint.d/
[web-86b97b6944-7ggt2] /docker-entrypoint.sh: Launching /docker-entrypoint.d/10-listen-on-ipv6-by-default.sh
[web-86b97b6944-7ggt2] 10-listen-on-ipv6-by-default.sh: Getting the checksum of /etc/nginx/conf.d/default.conf
[web-86b97b6944-7ggt2] /docker-entrypoint.sh: Launching /docker-entrypoint.d/20-envsubst-on-templates.sh
[web-86b97b6944-7ggt2] /docker-entrypoint.sh: Configuration complete; ready for start up
```

### 3.6 Close your deployment

When you are done with your application, close the deployment. This will deprovision your container and stop the token transfer. Close deployment using deployment by creating a deployment-close transaction:
  ```shell
  $ akash tx deployment close --node $AKASH_NODE \
    --chain-id $AKASH_CHAIN_ID --dseq $DSEQ \
    --owner $ACCOUNT_ADDRESS --from $KEY_NAME \
    --keyring-backend $KEYRING_BACKEND -y
  ```

Additionally, you can also query the market to check if your lease is closed:
  ```bash
  $ akash query market lease list --owner $ACCOUNT_ADDRESS --node $AKASH_NODE
  ```
You should see a response similar to:
  ```yaml
  - lease_id:
      dseq: "192910"
      gseq: 1
      oseq: 1
      owner: akash1kfd3adu7sgcu2vxd3ucc3pehmrautp25kxenn5
      provider: akash15ql9ycjkkxhpc2nxtnf78qqjguwzz8gc4ue7wl
    price:
      amount: "1310"
      denom: uakt
    state: closed
  pagination:
    next_key: null
    total: "0"
  ```
As you can notice from the above, you lease will be marked `closed`.

