# Base image to start from
FROM node:14-alpine as node_base

# Set the Working Directory the Containers Terminal Starts from
WORKDIR /usr/src/app/

# Install app dependencies
# A wildcard is used to ensure both package.json AND package-lock.json are copied
# where available (npm@5+)
COPY package*.json ./
COPY yarn.lock ./

# install dependencies
RUN yarn 

# Copy Project Over from this folder "." to "/usr/src/app/" in the container
COPY . .

FROM ubuntu:20.04 AS codeql_base
LABEL maintainer="Github codeql team, modified by G8"

# tzdata install needs to be non-interactive
ENV DEBIAN_FRONTEND=noninteractive

# install/update basics and python
RUN apt-get update && \
    apt-get upgrade -y && \
    apt-get install -y --no-install-recommends \
    software-properties-common \
    vim \
    curl \
    wget \
    git \
    build-essential \
    unzip \
    apt-transport-https \
    python3.8 \
    python3-venv \
    python3-pip \
    python3-setuptools \
    python3-dev \
    gnupg \
    g++ \
    make \
    gcc \
    apt-utils \
    rsync \
    file \
    dos2unix \
    gettext && \
    apt-get clean && \
    ln -sf /usr/bin/python3.8 /usr/bin/python && \
    ln -sf /usr/bin/pip3 /usr/bin/pip 

# Install .NET Core for tools/builds
RUN cd /tmp && \
    wget https://packages.microsoft.com/config/ubuntu/20.04/packages-microsoft-prod.deb -O packages-microsoft-prod.deb && \
    dpkg -i packages-microsoft-prod.deb && \
    apt-get update; \
    apt-get install -y apt-transport-https && \
    apt-get update && \
    rm packages-microsoft-prod.deb
RUN apt-get install -y dotnet-sdk-3.1

RUN mkdir -p /usr/local/nvm
# nvm environment variables
ENV NVM_DIR /usr/local/nvm
ENV NODE_VERSION 14.17.0

# install nvm
# https://github.com/creationix/nvm#install-script
RUN curl --silent -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.38.0/install.sh | bash

# install node and npm
RUN . $NVM_DIR/nvm.sh \
    && nvm install $NODE_VERSION \
    && nvm alias default $NODE_VERSION \
    && nvm use default

# add node and npm to path so the commands are available
ENV NODE_PATH $NVM_DIR/v$NODE_VERSION/lib/node_modules
ENV PATH $NVM_DIR/versions/node/v$NODE_VERSION/bin:$PATH

# Clone our setup and run scripts
#RUN git clone https://github.com/microsoft/codeql-container /usr/local/startup_scripts
# RUN mkdir -p /usr/local/startup_scripts
# RUN ls -al /usr/local/startup_scripts
# COPY container /usr/local/startup_scripts/
# RUN pip3 install --upgrade pip \
#     && pip3 install -r /usr/local/startup_scripts/requirements.txt
COPY --from=node_base /usr/src/app /usr/src/app

# Install latest codeQL
# ENV CODEQL_HOME /usr/local/codeql-home
ENV CODEQL_HOME=${CODEQL_HOME:-/usr/local/codeql-home}
# record the latest version of the codeql-cli

ARG CODEQL_VERSION
RUN if [ -z "$CODEQL_VERSION" ] ; \
    then echo "DEFAULTING TO LATEST CODEQL CLI VERSION" && \
    curl --silent "https://api.github.com/repos/github/codeql-cli-binaries/releases/latest" | grep '"tag_name":' | sed -E 's/.*"([^"]+)".*/\1/' > /tmp/codeql_version; \
    else echo "USING CODEQL CLI $CODEQL_VERSION" && \
    echo "$CODEQL_VERSION" > /tmp/codeql_version; fi
# RUN curl --silent "https://api.github.com/repos/github/codeql-cli-binaries/releases/latest" | grep '"tag_name":' | sed -E 's/.*"([^"]+)".*/\1/' > /tmp/codeql_version

RUN mkdir -p ${CODEQL_HOME} \
    ${CODEQL_HOME}/codeql-repo \
    ${CODEQL_HOME}/codeql-go-repo \
    /opt/codeql

# get the latest codeql queries and record the HEAD
RUN git clone --depth 1 https://github.com/github/codeql ${CODEQL_HOME}/codeql-repo && \
    git --git-dir ${CODEQL_HOME}/codeql-repo/.git log --pretty=reference -1 > /opt/codeql/codeql-repo-last-commit
RUN git clone --depth 1 https://github.com/github/codeql-go ${CODEQL_HOME}/codeql-go-repo && \
    git --git-dir ${CODEQL_HOME}/codeql-go-repo/.git log --pretty=reference -1 > /opt/codeql/codeql-go-repo-last-commit

RUN CODEQL_VERSION=$(cat /tmp/codeql_version) && \
    wget -q https://github.com/github/codeql-cli-binaries/releases/download/${CODEQL_VERSION}/codeql-linux64.zip -O /tmp/codeql_linux.zip && \
    unzip /tmp/codeql_linux.zip -d ${CODEQL_HOME} && \
    rm /tmp/codeql_linux.zip

ENV PATH="${CODEQL_HOME}/codeql:${PATH}"

ARG PRECOMPILE_QUERIES=false

RUN if [ "$PRECOMPILE_QUERIES" = "true" ] ; \
    # then codeql query compile --threads=0 ${CODEQL_HOME}/codeql-repo/*/ql/src/codeql-suites/*.qls && \
    # query compile --threads=0 ${CODEQL_HOME}/codeql-go-repo/ql/src/codeql-suites/*.qls; \
    # then codeql query compile --threads=0 ${CODEQL_HOME}/codeql-repo/javascript/ql/src/codeql-suites/*.qls; \
    then codeql query compile --threads=0 ${CODEQL_HOME}/codeql-repo/javascript/ql/src/codeql-suites/javascript-security-extended.qls; \
    else echo 'WARNING: NO PRECOMPILE_QUERIES, Pre-compiling CodeQL queries can save time later'; fi
# Pre-compile our queries to save time later
# RUN codeql query compile --threads=0 ${CODEQL_HOME}/codeql-repo/*/ql/src/codeql-suites/*.qls
# RUN codeql query compile --threads=0 ${CODEQL_HOME}/codeql-go-repo/ql/src/codeql-suites/*.qls

ENV PYTHONIOENCODING=utf-8
# ENTRYPOINT ["python3", "/usr/local/startup_scripts/startup.py"]
WORKDIR /usr/src/app/

EXPOSE 8080
CMD [ "node", "app.js" ]