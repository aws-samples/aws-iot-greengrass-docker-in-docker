# Copyright Amazon.com, Inc. or its affiliates. All Rights Reserved.
# SPDX-License-Identifier: MIT-0

# Multi-stage build: First stage builds the latest Greengrass image
FROM amazonlinux:2023 as greengrass-builder

# Replace the args to lock to a specific version
ARG GREENGRASS_RELEASE_VERSION=latest
ARG GREENGRASS_ZIP_FILE=greengrass-${GREENGRASS_RELEASE_VERSION}.zip
ARG GREENGRASS_RELEASE_URI=https://d2s8p88vqu9w66.cloudfront.net/releases/${GREENGRASS_ZIP_FILE}

# Author
LABEL maintainer="AWS IoT Greengrass"

# Greengrass Version
LABEL greengrass-version=${GREENGRASS_RELEASE_VERSION}

# Set up Greengrass v2 execution parameters
# TINI_KILL_PROCESS_GROUP allows forwarding SIGTERM to all PIDs in the PID group so Greengrass can exit gracefully
ENV TINI_KILL_PROCESS_GROUP=1 \
    GGC_ROOT_PATH=/greengrass/v2 \
    PROVISION=false \
    AWS_REGION=us-east-1 \
    THING_NAME=default_thing_name \
    THING_GROUP_NAME=default_thing_group_name \
    TES_ROLE_NAME=default_tes_role_name \
    TES_ROLE_ALIAS_NAME=default_tes_role_alias_name \
    COMPONENT_DEFAULT_USER=default_component_user \
    DEPLOY_DEV_TOOLS=false \
    INIT_CONFIG=default_init_config \
    TRUSTED_PLUGIN=default_trusted_plugin_path \
    THING_POLICY_NAME=default_thing_policy_name

RUN env

# Install Greengrass v2 dependencies
RUN yum update -y && \
    yum install -y tar unzip wget sudo procps which shadow-utils python3 java-11-amazon-corretto-headless && \
    wget $GREENGRASS_RELEASE_URI && \
    rm -rf /var/cache/yum && \
    mkdir -p /opt/greengrassv2 $GGC_ROOT_PATH && \
    unzip $GREENGRASS_ZIP_FILE -d /opt/greengrassv2 && \
    rm $GREENGRASS_ZIP_FILE

# Second stage: Add Docker-in-Docker capabilities to the Greengrass image
FROM greengrass-builder

# Install Docker-in-Docker dependencies
RUN yum install -y ca-certificates \
    openssh-clients

ENV DOCKER_VERSION 20.10.12

# Install Docker binaries
RUN set -eux; \
	\
	arch="$(uname -m)"; \
	case "$arch" in \
		'x86_64') \
			url='https://download.docker.com/linux/static/stable/x86_64/docker-20.10.12.tgz'; \
			;; \
		'armv7l') \
			url='https://download.docker.com/linux/static/stable/armhf/docker-20.10.12.tgz'; \
			;; \
		'aarch64') \
			url='https://download.docker.com/linux/static/stable/aarch64/docker-20.10.12.tgz'; \
			;; \
		*) echo >&2 "error: unsupported architecture ($arch)"; exit 1 ;; \
	esac; \
	\
	wget -O docker.tgz "$url"; \
	\
	tar --extract \
		--file docker.tgz \
		--strip-components 1 \
		--directory /usr/local/bin/ \
	; \
	rm docker.tgz; \
	\
	dockerd --version; \
	docker --version

# Copy Greengrass entrypoint scripts
COPY "greengrass-entrypoint.sh" /
COPY "modify-sudoers.sh" /

# Copy Docker entrypoint scripts
COPY "modprobe.sh" /usr/local/bin/modprobe
COPY docker-entrypoint.sh /usr/local/bin/
COPY dockerd-entrypoint.sh /usr/local/bin/

# Make scripts executable and modify sudoers
RUN chmod +x /greengrass-entrypoint.sh && \
    chmod +x /modify-sudoers.sh && \
    ./modify-sudoers.sh

# https://github.com/docker-library/docker/pull/166
#   dockerd-entrypoint.sh uses DOCKER_TLS_CERTDIR for auto-generating TLS certificates
#   docker-entrypoint.sh uses DOCKER_TLS_CERTDIR for auto-setting DOCKER_TLS_VERIFY and DOCKER_CERT_PATH
# (For this to work, at least the "client" subdirectory of this path needs to be shared between the client and server containers via a volume, "docker cp", or other means of data sharing.)
ENV DOCKER_TLS_CERTDIR=/certs
# also, ensure the directory pre-exists and has wide enough permissions for "dockerd-entrypoint.sh" to create subdirectories, even when run in "rootless" mode
RUN mkdir /certs /certs/client && chmod 1777 /certs /certs/client
# (doing both /certs and /certs/client so that if Docker does a "copy-up" into a volume defined on /certs/client, it will "do the right thing" by default in a way that still works for rootless users)

# Install additional Docker dependencies
RUN set -eux; \
	yum install -y \
		e2fsprogs \
		iproute \
		iptables \
		openssl \
		xfsprogs \
		xz \
		pigz \
		device-mapper \
	; \
# Install optional packages if available
	yum install -y btrfs-progs || echo "btrfs-progs not available, skipping"; \
	if yum list available zfs 2>/dev/null | grep -q zfs; then \
		yum install -y zfs; \
	fi

# set up subuid/subgid so that "--userns-remap=default" works out-of-the-box
RUN set -eux; \
	groupadd -r dockremap; \
	useradd -g dockremap dockremap; \
	echo 'dockremap:165536:65536' >> /etc/subuid; \
	echo 'dockremap:165536:65536' >> /etc/subgid

# https://github.com/docker/docker/tree/master/hack/dind
ENV DIND_COMMIT 42b1175eda071c0e9121e1d64345928384a93df1

RUN set -eux; \
	wget -O /usr/local/bin/dind "https://raw.githubusercontent.com/docker/docker/${DIND_COMMIT}/hack/dind"; \
	chmod +x /usr/local/bin/dind

VOLUME /var/lib/docker
EXPOSE 2375 2376
 
# Make Docker entrypoint scripts executable
RUN chmod +x /usr/local/bin/docker-entrypoint.sh && \
    chmod +x /usr/local/bin/dockerd-entrypoint.sh

ENTRYPOINT ["/bin/sh", "-c"]

CMD ["docker-entrypoint.sh && dockerd-entrypoint.sh & exec /greengrass-entrypoint.sh"]
