# buildkite agent versions https://github.com/buildkite/agent/releases ... latest as of now is buildkite-agent-linux-amd64-3.50.4.tar.gz
# look at official buildkite docker agent build script: https://github.com/buildkite/agent/tree/main/packaging/docker
# -- in above subdir ubuntu-22.04 contains image we need: https://github.com/buildkite/agent/tree/main/packaging/docker/ubuntu-22.04
### start by pasting contents of dockerfile in here.

# docker build -t buildkite/agent:ubuntu-lts .

# syntax=docker/dockerfile:1.4

##    docker build -t buildkite-custom-agent \
##    --build-arg="USER_ID=$(id -u $(whoami))" \
##    --build-arg="GROUP_ID=$(id -g $(whoami))" \
##    --build-arg="HOST_USERNAME=$(whoami)" \
##    --build-arg="TARGETOS=linux" \
##    --build-arg="TARGETARCH=amd64" \
##    --build-arg="AGENT_USER_HOMEDIR=/home/$(whoami)" \
##    --build-arg="DOCKERHOST_GID=$(getent group docker | cut -d: -f3)" .

FROM ubuntu:22.04


ARG USER_ID
ARG GROUP_ID
ARG HOST_USERNAME
ARG TARGETOS
ARG TARGETARCH
ARG AGENT_USER_HOMEDIR
ARG DOCKERHOST_GID

ENV AGENT_USER_HOMEDIR=${AGENT_USER_HOMEDIR}


RUN <<BASH
#!/usr/bin/env bash

set -eufo pipefail

export DEBIAN_FRONTEND=noninteractive

apt-get update
apt-get install -y --no-install-recommends \
  apt-transport-https \
  bash \
  ca-certificates \
  curl \
  git \
  gnupg-agent \
  jq \
  openssh-client \
  perl \
  python3 \
  python3-pip \
  rsync \
  software-properties-common \
  tini \
  unzip

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | apt-key add -
add-apt-repository "deb [arch=$TARGETARCH] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
apt-get update
apt-get install -y --no-install-recommends docker-ce-cli docker-compose-plugin docker-buildx-plugin
rm -rf /var/lib/apt/lists/*

ln -s /usr/bin/tini /usr/sbin/tini

curl -Lfs -o /usr/local/bin/ssh-env-config.sh https://raw.githubusercontent.com/buildkite/docker-ssh-env-config/master/ssh-env-config.sh
chmod +x /usr/local/bin/ssh-env-config.sh
BASH

RUN curl -fsSLO https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip \
  && unzip awscli-exe-linux-x86_64.zip \
  && ./aws/install -i /usr/local/aws -b /usr/local/bin \
  && rm awscli-exe-linux-x86_64.zip

RUN if [ ${USER_ID:-0} -ne 0 ] && [ ${GROUP_ID:-0} -ne 0 ] ; then \
      if [ "$(getent passwd ${USER_ID} | cut -d: -f3)" = "${USER_ID}" ] ; then \
        userdel $(getent passwd ${USER_ID} | cut -d: -f1) ; \
      fi ; \
      if [ "$(getent group ${GROUP_ID} | cut -d: -f3)" = "${GROUP_ID}" ] ; then \
        groupdel $(getent group ${GROUP_ID} | cut -d: -f1) ; \
      fi ; \
      groupadd -g ${GROUP_ID} ${HOST_USERNAME} && \
      useradd -l -u ${USER_ID} -g ${HOST_USERNAME} ${HOST_USERNAME} -d ${AGENT_USER_HOMEDIR} && \
      install -d -m 0755 -o ${HOST_USERNAME} -g ${HOST_USERNAME} ${AGENT_USER_HOMEDIR} ; \
      fi

RUN mkdir ${AGENT_USER_HOMEDIR}/.buildkite-agent \
    && mkdir ${AGENT_USER_HOMEDIR}/.buildkite-agent/bin \
    && mkdir ${AGENT_USER_HOMEDIR}/.buildkite-agent/builds \
    && mkdir ${AGENT_USER_HOMEDIR}/.buildkite-agent/buildkite-secrets \
    && mkdir ${AGENT_USER_HOMEDIR}/.buildkite-agent/hooks \
    && mkdir ${AGENT_USER_HOMEDIR}/.buildkite-agent/plugins

RUN if [ getent group ${DOCKERHOST_GID} | cut -d: -f3 ] ; then groupdel $(getent group ${DOCKERHOST_GID} | cut -d: -f1) ; fi \
&& groupadd -g ${DOCKERHOST_GID} docker \
&& usermod -aG docker ${HOST_USERNAME}

RUN mkdir -p /tmp/agent-tmp \
    && curl -fsSLO "https://github.com/buildkite/agent/releases/download/v3.50.4/buildkite-agent-linux-amd64-3.50.4.tar.gz" \
    && tar -zxvf buildkite-agent-linux-amd64-3.50.4.tar.gz -C /tmp/agent-tmp/ \
    && mv /tmp/agent-tmp/buildkite-agent ${AGENT_USER_HOMEDIR}/.buildkite-agent/bin/buildkite-agent \
    && mv /tmp/agent-tmp/bootstrap.sh ${AGENT_USER_HOMEDIR}/.buildkite-agent/bootstrap.sh \
    && rm -fr /tmp/agent-tmp

RUN chown -R ${USER_ID}:${GROUP_ID} ${AGENT_USER_HOMEDIR}

ENV PATH="${AGENT_USER_HOMEDIR}/.buildkite-agent/bin:${AGENT_USER_HOMEDIR}/.buildkite-agent:/usr/local/bin:/usr/local/lib/docker/cli-plugins:${PATH}"

COPY ./common/docker-compose /usr/local/bin/docker-compose
COPY ./entrypoint.sh ${AGENT_USER_HOMEDIR}/.buildkite-agent/bin/buildkite-agent-entrypoint

RUN chmod +x /usr/local/bin/docker-compose \
    ${AGENT_USER_HOMEDIR}/.buildkite-agent/bin/buildkite-agent-entrypoint

ENTRYPOINT ["buildkite-agent-entrypoint"]

USER ${HOST_USERNAME}

CMD ["start"]
