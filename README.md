# AWS IoT Greengrass V2 Docker-in-Docker

## Overview

AWS IoT Greengrass v2 does not allow running Docker application containers when Greengrass itself is running inside a Docker container. For example, if you want to run [AWS IoT SiteWise Edge](https://docs.aws.amazon.com/iot-sitewise/latest/userguide/configure-gateway-ggv2.html) inside a GGv2 container, AWS IoT SiteWise Edge requires Docker containers to run its components. 

This solution provides:
- **Latest AWS IoT Greengrass V2** (built from source)
- **Docker-in-Docker capabilities** for running containerized components
- **Multi-stage build** for optimized image size
- **Multi-architecture support** (x86_64, aarch64, armv7l)

This container image enables you to run your containerized applications within Greengrass. You can also leverage the [official AWS IoT Greengrass v2 guidelines](https://github.com/aws-greengrass/aws-greengrass-docker) for additional operations.

**⚠️ Production Warning**: Before using in production, please read this [blog about Docker-in-Docker considerations](https://jpetazzo.github.io/2015/09/03/do-not-use-docker-in-docker-for-ci/).

## Example Architecture Diagram

You can use this solution standalone or integrate it with container orchestration tools like Amazon ECS Anywhere. Here's an example target deployment architecture for Amazon ECS Anywhere:

![Architecture Diagram](docs/example-arch-diagram.png)

## Prerequisites

- Docker (version 18.09 or later)
- Docker Compose (version 1.22 or later)
- AWS credentials (if provisioning is enabled)

## Quick Start

### Using Docker Compose (Recommended)

```bash
# Navigate to docker directory
cd docker

# Build and run with default settings
docker-compose up --build

# Run in detached mode
docker-compose up --build -d
```

### Using Docker CLI

```bash
# Navigate to docker directory
cd docker

# Build the image
docker build -t x86_64/aws-iot-greengrass:latest .

# Run the container
docker run --init --privileged -it --name aws-iot-greengrass \
  x86_64/aws-iot-greengrass:latest
```

## Configuration

### Environment Variables

The following environment variables can be configured:

- `PROVISION`: Set to `true` to provision the device (default: `false`)
- `AWS_REGION`: AWS region (default: `us-east-1`)
- `THING_NAME`: IoT Thing name (default: `default_thing_name`)
- `THING_GROUP_NAME`: IoT Thing Group name (default: `default_thing_group_name`)
- `TES_ROLE_NAME`: Token Exchange Service role name (default: `default_tes_role_name`)
- `TES_ROLE_ALIAS_NAME`: Token Exchange Service role alias (default: `default_tes_role_alias_name`)
- `COMPONENT_DEFAULT_USER`: Default user for components (default: `default_component_user`)
- `DEPLOY_DEV_TOOLS`: Deploy development tools (default: `false`)

### AWS Credentials Setup (Optional)

AWS credentials are **optional** if you're running on an EC2 instance with an IAM role or have AWS credentials already configured in your environment. The container will automatically use available AWS credentials in the following order:

1. **IAM Role** (Recommended for EC2/ECS/EKS)
2. **Environment variables** 
3. **AWS credentials file** (`~/.aws/credentials`)

#### Option 1: Using IAM Role (Recommended for Production)

If running on EC2, ECS, or EKS with proper IAM roles, no additional configuration needed. Just ensure your role has the required Greengrass permissions.

#### Option 2: Using Environment Variables

Uncomment and update the credentials in `docker-compose.yml`:

```yaml
environment:
  - PROVISION=true
  - AWS_REGION=eu-west-1
  # Uncomment if not using IAM roles
  - AWS_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMPLE
  - AWS_SECRET_ACCESS_KEY=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
  - AWS_SESSION_TOKEN=AQoDYXdzEJr1K...o5OytwEXAMPLE=  # Optional for temporary credentials
  - THING_NAME=MyGreengrassCore
  - THING_GROUP_NAME=MyGreengrassCoreGroup
```

#### Option 3: Using AWS Credentials File (Not Recommended for Production)

Create credentials file at `path_to_dir/greengrass-v2-credentials/credentials`:

```
[default]
aws_access_key_id = <YourAWSAccessKey>
aws_secret_access_key = <YourSecretKey>
```

Then uncomment and update the volume mount in docker-compose.yml:

```yaml
volumes:
  # Uncomment if using ~/.aws/credentials file
  - /path/to/credential/directory/:/root/.aws/:ro
environment:
  - PROVISION=true
```

#### Option 3: Using Environment File (Legacy Method)

Create `env.cfg` file:

```bash
GGC_ROOT_PATH=/greengrass/v2
AWS_REGION=eu-west-1
PROVISION=true
THING_NAME=MyGreengrassCore
THING_GROUP_NAME=MyGreengrassCoreGroup
TES_ROLE_NAME=GreengrassV2TokenExchangeRole
TES_ROLE_ALIAS_NAME=GreengrassCoreTokenExchangeRoleAlias
COMPONENT_DEFAULT_USER=ggc_user:ggc_group
```

Then run:

```bash
docker run --privileged --rm --init -it --name aws-iot-gg \
  -v path_to_cred/greengrass-v2-credentials:/root/.aws:ro \
  --env-file env.cfg -p 8883 x86_64/aws-iot-greengrass:latest
```

**⚠️ Security Warning**: 
- For running this image, you need to provide a `--privileged` flag. Ensure your security threat model allows this action in your environment.

## Build Arguments

You can customize the Greengrass version during build:

```bash
# Build with specific Greengrass version
docker build --build-arg GREENGRASS_RELEASE_VERSION=2.12.0 \
  -t x86_64/aws-iot-greengrass:2.12.0 .

# Build with latest version (default)
docker build -t x86_64/aws-iot-greengrass:latest .
```

## Architecture Support

The Dockerfile supports multiple architectures:
- x86_64 (Intel/AMD 64-bit)
- aarch64 (ARM 64-bit)
- armv7l (ARM 32-bit)

## Volumes

- `/var/lib/docker`: Docker daemon storage
- `/greengrass/v2`: Greengrass installation directory
- `/greengrass/v2/logs`: Greengrass logs (can be mounted for persistence)

## Usage

### Open Shell in the Container

```bash
# Attach to running container
docker exec -it aws-iot-greengrass sh

# Run with shell access
docker run --init --privileged -it --entrypoint sh \
  x86_64/aws-iot-greengrass:latest
```

### Available Commands

Docker is already installed in the container image. After you log in, you can use Docker commands to manage application containers:

```bash
# List running containers inside Greengrass
docker ps

# Run a test container
docker run --rm hello-world

# Run application containers
docker run -d --name my-app nginx:alpine
```

## Debugging

### Persist Logs

```bash
# Mount logs directory to persist logs
docker run --init --privileged -it \
  -v $(pwd)/logs:/greengrass/v2/logs \
  x86_64/aws-iot-greengrass:latest
```

### Monitor Greengrass Logs

```bash
# View container logs
docker logs aws-iot-greengrass

# Follow logs in real-time
docker logs -f aws-iot-greengrass
```

## Stopping and Cleanup

```bash
# Stop the container
docker-compose stop
# or
docker stop aws-iot-greengrass

# Remove the container
docker-compose down
# or
docker rm aws-iot-greengrass
```

## Troubleshooting

1. **Permission Issues**: Ensure the container runs with `--privileged` flag for Docker-in-Docker functionality
2. **Credential Issues**: Verify AWS credentials are properly configured if provisioning is enabled
3. **Network Issues**: Check that required ports (2375, 2376) are available if using external Docker clients
4. **Storage Issues**: Ensure sufficient disk space for Docker images and Greengrass components
5. **Build Issues**: Ensure you're running commands from the `docker/` directory

## Security

See [CONTRIBUTING](CONTRIBUTING.md#security-issue-notifications) for more information.

## License

This library is licensed under the MIT-0 License. See the LICENSE file.

