# https://github.com/devcontainers/images
# https://github.com/devcontainers/features/tree/main/src/docker-in-docker
# base image for all devcontainers: https://github.com/docker-library/buildpack-deps

# use buildx syntax definition to use secrets
# syntax=docker/dockerfile:1.7-labs

# ============================================================================
# base
# ============================================================================
FROM ubuntu:24.04 AS base

ARG \
  DEVCONTAINER_VERSION=0.0.0
ENV \
  LANG="C.UTF-8" 

# set versions of installed packages =========================================

# NodeJS
ARG NODEJS_DEFAULT_RELEASE="22"
ARG NODEJS_16_RELEASE="16.20.1"
ARG NODEJS_18_RELEASE="18.16.0"
ARG NODEJS_20_RELEASE="20.12.2"
ARG PNPM_RELEASE="9.0.6"

# Java
ARG JAVA_DEFAULT_RELEASE="zulu-23.32.11"
ARG JAVA_17_RELEASE="zulu-17.42.19"
ARG JAVA_21_RELEASE="zulu-21.42.19"
ARG MAVEN_RELEASE="3.9"

# AWS
ARG AWSCLI_RELEASE="2.13.26"
ARG AWSVAULT_RELEASE="7.2.0"
ARG ANSIBLE_RELEASE="9.13.0"

# Docker
ARG DOCKERCLI_RELEASE="28"

RUN <<EOT
  set -eux
  export DEBIAN_FRONTEND=noninteractive
# restore manpages
# yes | unminimize 2>&1
# install minimal base packages
  apt-get update && apt-get install -y --no-install-recommends \
    ca-certificates curl wget \
    dialog gnupg sq \
    tzdata locales-all \
    procps apt-utils net-tools iputils-ping
# switch to https repos
  sed -i '/^[[:space:]]*#/!s/http/https/g' /etc/apt/sources.list.d/ubuntu.sources
# add additional repos
  install -dm 755 /etc/apt/keyrings
#  mise (https://mise.jdx.dev)
  wget -qO - https://mise.jdx.dev/gpg-key.pub | gpg --dearmor | tee /etc/apt/keyrings/mise-archive-keyring.gpg 1> /dev/null
  echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/mise-archive-keyring.gpg] https://mise.jdx.dev/deb stable main" | tee /etc/apt/sources.list.d/mise.list;
# install dev tooling
  apt-get update && apt-get install -y --no-install-recommends \
    mise \
    build-essential \
    sudo htop less bash-completion \
    bash zsh fish \
    zip mc ncdu tmux \
    git rsync openssh-client \
    ripgrep fzf fd-find jq eza
# clean up after package installation
  apt-get autoremove -y
  apt-get clean -y
  rm -rf /var/lib/apt/lists/*
EOT

# Set up system ==============================================================
RUN <<EOT
  set -eux
# try to remove ubuntu user (if present)
  userdel --force --remove ubuntu || true
# create user and group
  groupadd devs 
  useradd --create-home --groups devs,sudo --shell /bin/bash user --uid 1000;
# sudo without password
  echo '%sudo ALL=(ALL) NOPASSWD:ALL' > /etc/sudoers.d/sudo_devcontainer
# keep some env vars after sudo
  echo 'Defaults env_keep += "ftp_proxy http_proxy https_proxy no_proxy "' >> /etc/sudoers.d/sudo_devcontainer
# activate mise also for root
  echo 'eval "$(mise activate bash)"' >> /root/.bashrc
# append version and build date to /etc/motd
  env LANG="C.UTF-8" echo " devcontainer-ubuntu-base V${DEVCONTAINER_VERSION}, build `date +\"%Y-%m-%dT%H:%M:%S%z\"` " | sudo tee /etc/motd
  echo '' | sudo tee -a /etc/motd
EOT

# user context ===============================================================
USER user
WORKDIR /home/user

# user: install mise base tooling
RUN --mount=type=secret,id=mise_github_token,uid=1000 <<EOT
#!/usr/bin/env /usr/bin/bash
  export MISE_GITHUB_TOKEN=$(cat /run/secrets/mise_github_token)
  eval "$(mise activate bash)"
  export MISE_VERBOSE=true
  set -eux
# install (and use) user-global packages
  mise use --global --yes \
    node@${NODEJS_DEFAULT_RELEASE} \
    neovim@latest \
    starship@latest \
    lazygit@latest \
    delta@latest \
    task@latest
  mise list
EOT

# user: set up shell env
RUN <<EOT
#!/usr/bin/env /usr/bin/bash
  eval "$(mise activate bash)"
  set -eux
# git
  sudo git config --system advice.detachedHead false
  sudo git config --system init.defaultBranch master
  sudo git config --system core.autocrlf input
  sudo git config --system core.safecrlf warn
# starship prompt (https://starship.rs)
  starship config add_newline true
  starship config command_timeout 2000
  starship config hostname.ssh_only false
  starship config username.show_always true
  starship config username.format '[$user]($style)@'
  starship config container.disabled true
  echo 'eval "$(mise activate bash --shims)"' | sudo tee /etc/profile.d/mise.sh
# bash shell
  echo 'eval "$(mise activate bash)"' >> /home/user/.bashrc
  echo 'eval "$(starship init bash)"' >> /home/user/.bashrc
# zsh shell
  echo 'eval "$(mise activate zsh)"' >> /home/user/.zshrc
  echo 'eval "$(starship init zsh)"' >> /home/user/.zshrc
# fish shell
  mkdir -p /home/user/.config/fish
  echo 'mise activate fish | source' >> /home/user/.config/fish/config.fish
  echo 'starship init fish | source' >> /home/user/.config/fish/config.fish
EOT

ENV \
  LANG=de_DE.UTF-8 \
  LANGUAGE=de_DE.UTF-8 \
  LC_ALL=de_DE.UTF-8 \
  TZ=Europe/Berlin \
  SHELL=/bin/fish \
  EDITOR=nvim \
  DOCKER_BUILDKIT=1

# Mount for docker-in-docker 
VOLUME [ "/var/lib/docker" ]

CMD ["/usr/bin/bash"]
# CMD [ "sleep", "infinity" ]

# ============================================================================
# CI
# ============================================================================
FROM base AS ci

RUN --mount=type=secret,id=mise_github_token,uid=1000 <<EOT
#!/usr/bin/env /usr/bin/bash
  eval "$(mise activate bash)"
  export MISE_VERBOSE=true
  set -eux

# install user-global
  mise use --global --yes \
    awscli@${AWSCLI_RELEASE} \
    aws-vault@${AWSVAULT_RELEASE} \
    python@3.12 \
    pipx@1.7.1 \
    ansible@${ANSIBLE_RELEASE} \
    docker-cli@${DOCKERCLI_RELEASE} \
    pipx:awslogs@latest
# install other frequently used versions
  mise install --yes \
    node@${NODEJS_16_RELEASE} \
    node@${NODEJS_18_RELEASE} \
    node@${NODEJS_20_RELEASE} \
    pnpm@${PNPM_RELEASE} \
    java@${JAVA_DEFAULT_RELEASE} \
    java@${JAVA_17_RELEASE} \
    java@${JAVA_21_RELEASE} \
    maven@${MAVEN_RELEASE}
# list installed versions
    mise list
# add version information to /etc/motd
  env LANG="C.UTF-8" echo " devcontainer-ubuntu-ci V${DEVCONTAINER_VERSION}, build `date +\"%Y-%m-%dT%H:%M:%S%z\"` " | sudo tee /etc/motd
  echo '' | sudo tee -a /etc/motd
EOT

# ============================================================================
# CODER
# ============================================================================
FROM base AS coder
# install base xfce4 desktop and tools
RUN <<EOT
#!/usr/bin/env /usr/bin/bash
  eval "$(mise activate bash)"
  set -eux
# install desktop base system and tools
  sudo apt-get update && sudo apt-get install -y --no-install-recommends \
    xfce4 xfce4-terminal \
    xterm xclip dbus-x11 \
    meld 
# add version information to /etc/motd
  echo " devcontainer-ubuntu-desktop V${DEVCONTAINER_VERSION}, build `date +\"%Y-%m-%dT%H:%M:%S%z\"` " | sudo tee /etc/motd
  echo '' | sudo tee -a /etc/motd
EOT
#TODO: install DBeaver (Andi)
#TODO: install kasmvnc
#TODO: install eclipse+lombok
#TODO: install vscode (gui)

