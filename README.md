# AWS IoT GreenGrass v2 Docker in Docker
## Overview
AWS IoT GreenGrass v2 does not allow to run any Docker application container if GreenGrass itself is running inside a Docker container. This container image built on the official AWS Iot GreenGrass version 2 docker image to run your containerized applications. Therefore, you can also follow official AWS IoT GreenGrass v2 guideline to use any AWS Iot GreenGrass related operations. Before using in production please read following [blog](https://jpetazzo.github.io/2015/09/03/do-not-use-docker-in-docker-for-ci/).

## Example Architecture Diagram for the Solution

![Architecture Diagram](docs/example-arch-diagram.png)

## Build Image
`docker-compose -f docker/docker-compose.yml build`

## Update AWS Credential
`path_to_dir/greengrass-v2-credentials/credentials`

Example Credential file:

```
[default]
aws_access_key_id = <YourAWSAccessKey>
aws_secret_access_key = <YourSecretKey>
```

## AWS IoT GreenGrass v2 Provisioning
You can provide your environment file like below. 
 
### Provide Your Environment File for Provisioning
Create env file and copy your config accordingly. 

`nano env.cfg`
```
GGC_ROOT_PATH=/greengrass/v2
AWS_REGION=eu-west-1
PROVISION=true
THING_NAME=MyGreengrassCore
THING_GROUP_NAME=MyGreengrassCoreGroup
TES_ROLE_NAME=GreengrassV2TokenExchangeRole
TES_ROLE_ALIAS_NAME=GreengrassCoreTokenExchangeRoleAlias
COMPONENT_DEFAULT_USER=ggc_user:ggc_group
```
## Run AWS IoT GreenGrass v2 Docker in Docker Image
* **WARNING**: For running this image, you need to provide "privileged" flag, therefore it is important to know whether your security threat model allows this action in your environment.

`docker run --privileged --rm --init -it --name aws-iot-gg -v path_to_cred/greengrass-v2-credentials:/root/.aws:ro --env-file env.cfg  -p 8883 x86_64/aws-iot-greengrass:2.5.3 `

## Open Shell in the Image
`docker exec -it CONTAINER_ID /bin/sh`

## Available Commands in Image
Docker is already installed in the container image, after you log in, you can use docker commands to manage application containers

## Security

See [CONTRIBUTING](CONTRIBUTING.md#security-issue-notifications) for more information.

## License

This library is licensed under the MIT-0 License. See the LICENSE file.

