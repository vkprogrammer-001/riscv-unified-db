FROM ubuntu:24.04
LABEL org.opencontainers.image.source=https://github.com/riscv/riscv-unified-db

ENV DEBIAN_FRONTEND=noninteractive \
    MISE_DATA_DIR="/opt/mise" \
    MISE_CONFIG_DIR="/opt/mise/config" \
    MISE_CACHE_DIR="/opt/mise/cache" \
    MISE_STATE_DIR="/opt/mise/state" \
    MISE_INSTALL_PATH="/usr/local/bin/mise" \
    MISE_YES=1

WORKDIR /workspace

# please keep pkgs sorted
RUN apt-get update && \
  apt-get install -y --no-install-recommends --fix-missing \
  build-essential \
  clang-format \
  clang-tidy \
  cmake \
  curl \
  ditaa \
  g++ \
  gcc-riscv64-linux-gnu \
  gcc-riscv64-unknown-elf \
  gdb \
  gh \
  git \
  gpg \
  gpg-agent \
  less \
  libc6-dev-riscv64-cross \
  libelf-dev \
  libgmp-dev \
  libnewlib-dev \
  libyaml-dev \
  libz3-dev \
  minisat \
  parallel \
  rustc \
  shellcheck \
  sudo \
  zlib1g-dev \
  \
  && apt-get clean autoclean \
  && apt-get autoremove -y \
  && rm -rf /var/lib/{apt,dpkg,cache,log}/*

RUN curl https://mise.run | sh

COPY .mise.toml .ruby-version /workspace/

# - 'mise install -y': Installs tools defined in .mise.toml
RUN mise settings set trusted_config_paths '~' && \
    mise trust /workspace/.mise.toml && \
    mise install -y && \
    mise prune -y

# Set installed versions as global defaults (dynamically, no hardcoded versions!)
# This creates latest symlinks that we can use in PATH
RUN for tool in node python ruby; do \
        version=$(ls /opt/mise/installs/$tool | head -1); \
        mise use --global ${tool}@${version}; \
    done && \
    chmod -R 777 /opt/mise

# Set PATH using latest symlinks (avoids shim overhead and ARG_MAX limit)
# mise use --global created /opt/mise/installs/{tool}/latest symlinks for us
ENV PATH="/workspace/bin/container:/opt/node/node_modules/.bin:/opt/venv/bin:/opt/mise/installs/node/latest/bin:/opt/mise/installs/python/latest/bin:/opt/mise/installs/ruby/latest/bin:$PATH"

# build/install eqntott
RUN git clone --depth=1 https://github.com/TheProjecter/eqntott.git && \
    cd eqntott && \
    ./configure && \
    make && make install && \
    cd .. && \
    rm -rf eqntott

# build/install espresso
RUN git clone --depth=1 https://github.com/psksvp/espresso-ab-1.0.git && \
    cd espresso-ab-1.0 && \
    ./configure && \
    make && make install && \
    cd .. && \
    rm -rf espresso-ab-1.0

# build / install must
# mcsmus/mcsmus/control.cc is missing #include <cstdio>, thus the sed below
RUN git clone https://github.com/jar-ben/mustool.git && \
    cd mustool && \
    git checkout 17fa9f9542a9ce05328dfccd1cd410f05f741ab3 && \
    sed -i -e 's/#include <signal.h>/#include <signal.h>\n#include <cstdio>/' mcsmus/mcsmus/control.cc && \
    make -j && \
    install must -m 0777 /usr/bin/must && \
    cd .. && \
    rm -rf mustool

# Create Python virtual environment and install packages
COPY requirements.txt /opt/requirements.txt
RUN python -m venv /opt/venv && \
  /opt/venv/bin/pip install -r /opt/requirements.txt && \
  rm /opt/requirements.txt
ENV PATH="/opt/venv/bin:$PATH"

# Install npm packages globally in the container
COPY package.json package-lock.json /opt/node/
RUN cd /opt/node && \
  npm ci && \
  rm /opt/node/package.json /opt/node/package-lock.json
ENV NODE_PATH="/opt/node/node_modules"
ENV PATH="/opt/node/node_modules/.bin:$PATH"
ENV IN_UDB_CONTAINER=true

# install gems
COPY Gemfile /opt/Gemfile
COPY Gemfile.lock /opt/Gemfile.lock
COPY tools/ruby-gems/idlc/idlc.gemspec /opt/tools/ruby-gems/idlc/
COPY tools/ruby-gems/idlc/lib/gem_versions.rb /opt/tools/ruby-gems/idlc/lib/
COPY tools/ruby-gems/idlc/lib/idlc/version.rb /opt/tools/ruby-gems/idlc/lib/idlc/
COPY tools/ruby-gems/idl_highlighter/idl_highlighter.gemspec /opt/tools/ruby-gems/idl_highlighter/
COPY tools/ruby-gems/idl_highlighter/lib/idl_highlighter/version.rb /opt/tools/ruby-gems/idl_highlighter/lib/idl_highlighter/
COPY tools/ruby-gems/udb/udb.gemspec /opt/tools/ruby-gems/udb/
COPY tools/ruby-gems/udb/lib/gem_versions.rb /opt/tools/ruby-gems/udb/lib/
COPY tools/ruby-gems/udb/lib/udb/version.rb /opt/tools/ruby-gems/udb/lib/udb/
COPY tools/ruby-gems/udb-gen/udb-gen.gemspec /opt/tools/ruby-gems/udb-gen/
COPY tools/ruby-gems/udb-gen/lib/gem_versions.rb /opt/tools/ruby-gems/udb-gen/lib/
COPY tools/ruby-gems/udb-gen/lib/udb-gen/version.rb /opt/tools/ruby-gems/udb-gen/lib/udb-gen/
COPY tools/ruby-gems/udb_helpers/udb_helpers.gemspec /opt/tools/ruby-gems/udb_helpers/
COPY tools/ruby-gems/udb_helpers/lib/udb_helpers/version.rb /opt/tools/ruby-gems/udb_helpers/lib/udb_helpers/
RUN bundle install --gemfile /opt/Gemfile

# the binaries for gems need to be copied in
RUN mkdir -p /workspace/bin/container
COPY tools/ruby-gems/idlc/bin/idlc /workspace/bin/container
COPY tools/ruby-gems/udb/bin/udb /workspace/bin/container
COPY tools/ruby-gems/udb-gen/bin/udb-gen /workspace/bin/container
ENV PATH="/workspace/bin/container:$PATH"

# Grant the default ubuntu user sudo access
RUN echo ubuntu ALL=\(root\) NOPASSWD:ALL > /etc/sudoers.d/ubuntu \
    && chmod 0440 /etc/sudoers.d/ubuntu

USER ubuntu
