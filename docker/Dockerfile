FROM ubuntu:22.04 as build

ARG TARGETOS=linux
ARG TARGETARCH=amd64
ARG RUNNER_VERSION=2.315.0
ARG RUNNER_CONTAINER_HOOKS_VERSION=0.6.0
ARG DOCKER_VERSION=25.0.4
ARG BUILDX_VERSION=0.13.1
ENV DOTNET_RUNNING_IN_CONTAINER=true

RUN apt update -y && apt install curl wget sudo unzip -y && \
    apt-get install -y --no-install-recommends \
        ca-certificates \
        \
        # .NET dependencies
        libc6 \
        libgcc1 \
        libgssapi-krb5-2 \
        libicu70 \
        libssl3 \
        libstdc++6 \
        zlib1g && \
    rm -rf /var/lib/apt/lists/*

WORKDIR /actions-runner
RUN export RUNNER_ARCH=${TARGETARCH} \
    && if [ "$RUNNER_ARCH" = "amd64" ]; then export RUNNER_ARCH=x64 ; fi \
    && curl -f -L -o runner.tar.gz https://github.com/actions/runner/releases/download/v${RUNNER_VERSION}/actions-runner-${TARGETOS}-${RUNNER_ARCH}-${RUNNER_VERSION}.tar.gz \
    && tar xzf ./runner.tar.gz \
    && rm runner.tar.gz

RUN curl -f -L -o runner-container-hooks.zip https://github.com/actions/runner-container-hooks/releases/download/v${RUNNER_CONTAINER_HOOKS_VERSION}/actions-runner-hooks-k8s-${RUNNER_CONTAINER_HOOKS_VERSION}.zip \
    && unzip ./runner-container-hooks.zip -d ./k8s \
    && rm runner-container-hooks.zip

RUN export RUNNER_ARCH=${TARGETARCH} \
    && if [ "$RUNNER_ARCH" = "amd64" ]; then export DOCKER_ARCH=x86_64 ; fi \
    && if [ "$RUNNER_ARCH" = "arm64" ]; then export DOCKER_ARCH=aarch64 ; fi \
    && curl -fLo docker.tgz https://download.docker.com/${TARGETOS}/static/stable/${DOCKER_ARCH}/docker-${DOCKER_VERSION}.tgz \
    && tar zxvf docker.tgz \
    && rm -rf docker.tgz \
    && mkdir -p /usr/local/lib/docker/cli-plugins \
    && curl -fLo /usr/local/lib/docker/cli-plugins/docker-buildx \
        "https://github.com/docker/buildx/releases/download/v${BUILDX_VERSION}/buildx-v${BUILDX_VERSION}.linux-${TARGETARCH}" \
    && chmod +x /usr/local/lib/docker/cli-plugins/docker-buildx

FROM ubuntu:22.04

ENV DEBIAN_FRONTEND=noninteractive
ENV RUNNER_MANUALLY_TRAP_SIG=1
ENV ACTIONS_RUNNER_PRINT_LOG_TO_STDOUT=1
ENV ImageOS=ubuntu22
ENV DOTNET_RUNNING_IN_CONTAINER=true

RUN apt-get update -y && apt-get install curl wget sudo unzip -y && \
    apt-get install -y --no-install-recommends \
    ca-certificates \
    jq \
    software-properties-common \
    git \
    gnupg2 \
    python2 \
    openssl \
    libc6 \
    libgcc1 \
    libgssapi-krb5-2 \
    libicu70 \
    libssl3 \
    libstdc++6 \
    zlib1g \
    sudo \
    make \
    lsb-release && \
    rm -rf /var/lib/apt/lists/*

RUN apt-get update && \
    wget -qO /usr/local/bin/yq https://github.com/mikefarah/yq/releases/latest/download/yq_linux_amd64 && \
    chmod a+x /usr/local/bin/yq && \
    apt-get install -y python3-pip && \
    apt-get install -y software-properties-common && \
    add-apt-repository --yes --update ppa:ansible/ansible && \
    apt-get install -y ansible && \
    apt-get install -y curl git wget && \
    curl -fsSL https://deb.nodesource.com/setup_lts.x | bash - && \
    apt-get install -y nodejs && \
    npm install --global yarn && \
    npm install -g typescript && \
    echo "deb https://ppa.launchpadcontent.net/brightbox/ruby-ng/ubuntu/ focal main"|tee  /etc/apt/sources.list.d/brightbox.list && \
    apt-key adv --keyserver keyserver.ubuntu.com --recv-keys  80F70E11F0F0D5F10CB20E62F5DA5F09C3173AA6 && \
    apt-get install -y ruby && \
    add-apt-repository ppa:longsleep/golang-backports --yes --update && \
    apt-get install -y golang-go && \
    mkdir -p ~/go/{bin,pkg,src} && \
    echo 'export GOPATH="$HOME/go"' >> ~/.bashrc && \
    echo 'export PATH="$PATH:${GOPATH//://bin:}/bin"' >> ~/.bashrc && \
    wget https://github.com/gruntwork-io/terragrunt/releases/download/v0.38.12/terragrunt_linux_amd64 && \
    mv terragrunt_linux_amd64 terragrunt && \
    chmod u+x terragrunt && \
    mv terragrunt /usr/local/bin/terragrunt && \
    install -m 0755 -d /etc/apt/keyrings && \
    curl -fsSL https://get.opentofu.org/opentofu.gpg | tee /etc/apt/keyrings/opentofu.gpg >/dev/null && \
    curl -fsSL https://packages.opentofu.org/opentofu/tofu/gpgkey | gpg --no-tty --batch --dearmor -o /etc/apt/keyrings/opentofu-repo.gpg >/dev/null && \
    chmod a+r /etc/apt/keyrings/opentofu.gpg && \
    echo "deb [signed-by=/etc/apt/keyrings/opentofu.gpg,/etc/apt/keyrings/opentofu-repo.gpg] https://packages.opentofu.org/opentofu/tofu/any/ any main" | \
    tee /etc/apt/sources.list.d/opentofu.list > /dev/null && \
    apt-get update && \
    apt-get install -y tofu && \
    apt-get install -yq --no-install-recommends \
    postgresql-client-14 \
    icu-devtools \
    libpq-dev \
    imagemagick \
    graphviz \
    fonts-open-sans \
    shared-mime-info && \
    apt-get clean && \
    apt-get autoclean && \
    rm -rf /var/lib/apt/lists/*


RUN adduser --disabled-password --gecos "" --uid 1001 runner \
    && groupadd docker --gid 123 \
    && usermod -aG sudo runner \
    && usermod -aG docker runner \
    && echo "%sudo   ALL=(ALL:ALL) NOPASSWD:ALL" > /etc/sudoers \
    && echo "Defaults env_keep += \"DEBIAN_FRONTEND\"" >> /etc/sudoers \
    && mkdir -p /opt/hostedtoolcache \
    && chown runner:docker /opt/hostedtoolcache \
    && chmod 775 /opt/hostedtoolcache

WORKDIR /home/runner

COPY --chown=runner:docker --from=build /actions-runner .
COPY --from=build /usr/local/lib/docker/cli-plugins/docker-buildx /usr/local/lib/docker/cli-plugins/docker-buildx

RUN install -o root -g root -m 755 docker/* /usr/bin/ && rm -rf docker

USER runner
