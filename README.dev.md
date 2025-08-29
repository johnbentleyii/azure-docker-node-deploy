# Node Docker Development Environment
Provides a Docker container to run an instance of Node using nodemon to:
1. Reload elements of the node application as code is changed and saved
1. Enable remote debugging in VSCode

## Quick Start Guide
Once configured, the following steps will allow you to run your Node application and interact with the debugger.
1. Build and run the docker container, navigate to `docker-compose.dev.yml` and select `Compose Up` from the context menu.
1. To monitor `nodemon` activity, use the `Containers` extension from Docker, find the container, and select `Compose Logs`.
1. For debugging, go to the `Run and Debug`, select `Attach to Docker/Node` from the play menu.
1. Once attached, open any Node source file to place breakpoints, add log points, etc.

## Configuring for your Node Application
To connect the Docker configuration to your Node application, the following values are configured for your specific application and computing environment:
- PACKAGE_PATH - the path to the Node application's `package.json`
- SOURCE_DIR - the path to the root of your Node application's source code
- STARTUP_ENV - the relative path to your startup script, i.e. index.js, bin/www
- APP_PORT - the local port number you will use to connect to Node
- DEBUG_PORT - the local port number VSCode will use to connect for debugging

Note that the `APP_PORT` and `DEBUG_PORT` must be unique per Docker/Node image which is concurrently in operation.

In addition, there is a package and script requirement for the Node application's `package.json` file.

File modifications are as follows.

### .env File
Set the following variables in the `.env` file, located in the root of this repo: 
- PACKAGE_PATH - the path to the Node application's `package.json`
- SOURCE_DIR - the path to the root of your Node application's source code
- STARTUP_ENV - the relative path to your startup script, i.e. index.js, bin/www
- APP_PORT - the local port number you will use to connect to Node
- DEBUG_PORT - the local port number VSCode will use to connect for debugging

Note that the `APP_PORT` and `DEBUG_PORT` must be unique per Docker/Node image which is concurrently in operation.

The `.env` file is used by `docker-compose.dev.yml`, which also passes the values to  `docker/Dockerfile.env`. Note this configuration and usage if you need to integrated the Docker configuration into another Docker compose configuration.

Also, note that the use of a Docker volume to mount the source files is required to utilize the dynamic reloading feature.

### launch.json
VSCode is not able to use the values in `.env` so the `.vscode/launch.json` must be updated to match the values in `.env`.

First, set the `"port"` value. For example, if your `.env` file had the following setting:
```
DEBUG_PORT=59229
```
then the `launch.json` should have:
```
     "port": 59229,
```

Next, set `"localRoot"` to match the `SOURCE_DIR` in `.env`. For example, if your `.env` file had:
```
SOURCE_PATH=~/repos/my-node-application/
```

then the `launch.json` should have:
```
    "localRoot": "/home/myusername/repos/my-node-application"
```

Note that if the `launch.json` is in the same project folder as the Node appplication, e.g. in `/home/myusername/repos/my-node-application/.vscode` from the prior example, `localRoot` can use the `${workspaceFolder}`:
```
    "localRoot": "${workspaceFolder}"
```

## Embedding with the Node application
The most straightforward way of leverage the Docker/Node container by including as part of the Node application project/repository. To do this, copy the following files to the repo:
- `.vscode/launch.json`
- `docker/Dockerfile.dev`
- `docker-compose.dev.yml`
- `.env`

Follow the configuration instructions, setting `localRoot` to `${workspaceFolder}` if the Node application is in the root of the project. If the Node application is in a subfolder, append the path to `${workspaceFolder}`.

If there are multiple Node applications, copy the container configuration `app` in `docker-compose.dev.yml`, uniquely name for each Node application, and update the appropriate settings in the configuration. You will also need to set unique ports for each of the Node application's `APP_PORT` and `DEBUG_PORT`. You will also need to create uniquely named entries in `launch.json`, i.e. `"Attach Node App 1", "Attach Node App 2"`.