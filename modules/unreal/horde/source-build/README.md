# Horde Server Source Build

This guide documents how to build Horde Server from source and use the build as part of your Horde deployment with the Cloud Game Development Toolkit.

## Prerequisites

- **Docker**: Docker Desktop installed and running
- **Git**: For cloning repositories
- **.NET SDK**: .NET 8.0 or later

## Fork and clone the Unreal Engine Github repository

You can follow the steps documented in the [Unreal Engine Source Code](https://github.com/EpicGames/UnrealEngine/tree/release). Be sure to run both Setup and GenerateProjectFiles scripts before proceeding.

## Authenticate to your ECR registry
In a terminal window, run the following command to authenticate to your ECR registry:
```bash
    aws ecr get-login-password --region <region> | docker login --username AWS --password-stdin <account-id>.dkr.ecr.<region.>.amazonaws.com
```

## Create a repsoitory in Amazon ECR for storing your Horde Server container image:
In a terminal window, run the following command:
```bash
aws ecr create-repository --repository-name horde-server --image-scanning-configuration scanOnPush=true
```

## Build Horde Server container image from source and publish to Amazon ECR

### For devices using ARM64 architecture
1. In an IDE, open the Horde Server Dockerfile located at `/Engine/Source/Programs/Horde/HordeServer/Dockerfile`.
2. Replace the code between `COPY --from=redis /usr/local/bin/redis-server /usr/local/bin/redis-server` and `COPY Source/Programs/Shared/EpicGames.Core/*.csproj ./Source/Programs/Shared/EpicGames.Core/` with the following:

    ```dockerfile
    # Since the .deb does not install in this image, just download it and extract the static binary
    RUN ARCH=$(dpkg --print-architecture) && \
        if [ "$ARCH" = "arm64" ]; then \
            wget https://repo.mongodb.org/apt/ubuntu/dists/jammy/mongodb-org/7.0/multiverse/binary-arm64/mongodb-org-server_7.0.4_arm64.deb && \
            dpkg -x mongodb-org-server_7.0.4_arm64.deb /tmp/mongodb; \
        else \
            wget https://repo.mongodb.org/apt/debian/dists/bookworm/mongodb-org/7.0/main/binary-amd64/mongodb-org-server_7.0.4_amd64.deb && \
            dpkg -x mongodb-org-server_7.0.4_amd64.deb /tmp/mongodb; \
        fi && \
        cp /tmp/mongodb/usr/bin/mongod /usr/local/bin/mongod
    ```
3. Locate the code block with the comment `# Remove native libs not used on Linux x86_64`. We need to modify this so that the build has the libraries associated with the operating system and CPU architecture we are using. To do this, replace that block with the following:
```dockerfile
# Remove native libs not used for current architecture
RUN ARCH=$(dpkg --print-architecture) && \
    if [ "$ARCH" = "amd64" ]; then \
        rm -rf /app/out/runtimes/osx* && \
        rm -rf /app/out/runtimes/win-x86 && \
        rm -rf /app/out/runtimes/win-arm* && \
        rm -rf /app/out/runtimes/linux-arm* && \
        rm -rf /app/out/runtimes/linux-x86 && \
        rm -rf /app/out/runtimes/linux/native/libgrpc_csharp_ext.x86.so; \
    elif [ "$ARCH" = "arm64" ]; then \
        rm -rf /app/out/runtimes/osx-x64 && \
        rm -rf /app/out/runtimes/win-x64 && \
        rm -rf /app/out/runtimes/win-x86 && \
        rm -rf /app/out/runtimes/linux-x64* && \
        rm -rf /app/out/runtimes/linux-x86 && \
        rm -rf /app/out/runtimes/linux/native/libgrpc_csharp_ext.x64.so && \
        rm -rf /app/out/runtimes/linux/native/libgrpc_csharp_ext.x86.so; \
    fi
```

## Update the Horde BuildGraph
1. In an IDE, open the file `Engine/Build/BatchFiles/BuildGraph.xml`.
2. Create a new build target to build the Hord Server with the Dashboard and push the image to your Amazon ECR repository. Be sure to replace the values for `<account-id>` and `<region>` with your own values.
    ```xml
    <Node Name="Build and Publish HordeServer to ECR" Requires="Build HordeServer;Build HordeDashboard">
        <!-- Create a new image by combining server and dashboard image into one -->
        <Docker-Build BaseDir="Engine/Source/Programs/Horde/HordeServer" Files="Dockerfile*" UseBuildKit="true" Tag="horde-server" DockerFile="Engine/Source/Programs/Horde/HordeServer/Dockerfile.dashboard" />

        <!-- Publish the docker image to Amazon ECR -->
        <Docker-Push Repository="<account-id>.dkr.ecr.<region>.amazonaws.com" Image="horde-server" TargetImage="horde/horde-server:$(Version)" AwsEcr="True" />
    </Node>
    ```

4. In your terminal window, and navigate to `Engine/Build/BatchFiles`.
    ```bash
    cd Engine/Build/BatchFiles
    ```

6. Run the following command to build the Horde Server and Dashboard, then publish your docker image to Amazon ECR.
    ### Windows
    ```powershell
    RunUAT.bat BuildGraph -Script=Engine/Source/Programs/Horde/BuildHorde.xml -Target="Build and Publish HordeServer to ECR"
    ```

    ### Linux and Mac
    ```bash
    ./RunUAT.sh BuildGraph -Script=Engine/Source/Programs/Horde/BuildHorde.xml -Target="Build and Publish HordeServer to ECR"
    ```

7. Note the URI of the image that was published to Amazon ECR. It should be in the format of `<account-id>.dkr.ecr.<region>.amazonaws.com/horde/horde-server:<version>`

8. You can also locate the URI of the image in the AWS Console under Amazon ECR, or using the AWS CLI command:
    ```bash
    aws ecr describe-images --repository-name horde-server
    ```

## Update your Cloud Game Development Toolkit configuration
In order to use your newly created container image as your Horde Server image, you will need to specify the URI of the image as the value for the `image` variable. For this example we will be using the example deployment located at `modules/unreal/horde/examples/complete`
1. In your IDE, open the main.tf located at `modules/unreal/horde/examples/complete/main.tf`.
2. Under the module declaration `"unreal_engine_horde"` add the following line with the value of the image URI you noted earlier. Be sure to also set the value of `is_source_build` to `true` and set the value of `horde_server_architecture` to `ARM64` or `X_86` by passing the value with the `var.horde_server_architecture` variable as shown below:
    ```python
    module "unreal_engine_horde" {
        ...
        image = "<image-uri>"
        is_source_build = true
        horde_server_architecture = "<horde-server-architecture>"
        ...
    }
    ```
3. In a new terminal window, navigate to the directory `modules/unreal/horde/examples/complete` and run the following commands:
    ```bash
    terraform init
    terraform apply
    ```

## Update the server.json configuration file
