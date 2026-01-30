FROM ubuntu:24.04
LABEL org.opencontainers.image.source=https://github.com/riscv/riscv-unified-db

ENV DEBIAN_FRONTEND=noninteractive

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

ENV MISE_DATA_DIR="/mise"
ENV MISE_CONFIG_DIR="/mise"
ENV MISE_CACHE_DIR="/mise/cache"
# MISE_STATE_DIR needed to prevent permission errors in CI when mise tries to track configs
# Without this, mise defaults to ~/.local/state which isn't writable by non-root CI users
ENV MISE_STATE_DIR="/mise/state"
ENV MISE_INSTALL_PATH="/usr/local/bin/mise"
ENV MISE_YES=1

RUN curl https://mise.run | sh

COPY .mise.toml /opt/.mise.toml
COPY .ruby-version /opt/.ruby-version

WORKDIR /opt
RUN mise trust /opt/.mise.toml && \
    mise settings set trusted_config_paths '~' && \
    mise install --yes

RUN chmod -R 777 /mise

# Verify tools work after mise installation
RUN cd /opt && \
    mise env --shell bash | grep '^export PATH=' > /etc/profile.d/mise-path-temp.sh && \
    bash -c 'source /etc/profile.d/mise-path-temp.sh && ruby --version && node --version && npm --version && python3 --version'

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
RUN bash -l -c 'python3 -m venv /opt/venv && \
  /opt/venv/bin/pip install --no-cache-dir -r /opt/requirements.txt && \
  rm /opt/requirements.txt'
ENV PATH="/opt/venv/bin:$PATH"

# Install npm packages globally in the container
COPY package.json package-lock.json /opt/node/
RUN bash -l -c 'cd /opt/node && \
  npm ci --no-audit --no-fund && \
  chown -R root:root /opt/node/node_modules && \
  rm /opt/node/package.json /opt/node/package-lock.json'
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
RUN bash -l -c 'bundle install --gemfile /opt/Gemfile'

# Remove mise binary and unset MISE_* env vars to prevent workspace .mise.toml detection
# Replace mise wrapper scripts (npm/npx) with direct symlinks to avoid "mise: command not found"
# Tools remain accessible via the PATH extracted earlier to /etc/profile.d/mise-path.sh
RUN rm -f /usr/local/bin/mise && \
    rm -rf /mise/cache /mise/downloads && \
    echo 'unset MISE_DATA_DIR MISE_CONFIG_DIR MISE_CACHE_DIR MISE_STATE_DIR MISE_INSTALL_PATH MISE_YES' >> /etc/bash.bashrc && \
    for node_dir in /mise/installs/node/*/bin; do \
        rm -f "$node_dir/npm" "$node_dir/npx" && \
        ln -s ../lib/node_modules/npm/bin/npm-cli.js "$node_dir/npm" && \
        ln -s ../lib/node_modules/npm/bin/npx-cli.js "$node_dir/npx"; \
    done

# the binaries for gems need to be copied in
RUN mkdir -p /workspace/bin/container
COPY tools/ruby-gems/idlc/bin/idlc /workspace/bin/container
COPY tools/ruby-gems/udb/bin/udb /workspace/bin/container
COPY tools/ruby-gems/udb-gen/bin/udb-gen /workspace/bin/container
ENV PATH="/workspace/bin/container:$PATH"

# Grant the default ubuntu user sudo access
RUN echo ubuntu ALL=\(root\) NOPASSWD:ALL > /etc/sudoers.d/ubuntu \
    && chmod 0440 /etc/sudoers.d/ubuntu

# Set complete PATH as ENV for Singularity compatibility (doesn't support ENTRYPOINT)
# We use mise's 'latest' symlinks for Node and Ruby to avoid hardcoding versions
# Python uses /opt/venv (with packages) instead of mise Python (without packages)
ENV PATH="/workspace/bin/container:/opt/node/node_modules/.bin:/opt/venv/bin:/opt/bin:/mise/installs/node/latest/bin:/mise/installs/ruby/latest/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"

# Create mise-path.sh with the final PATH for Docker ENTRYPOINT (matches ENV above)
RUN echo "export PATH='/workspace/bin/container:/opt/node/node_modules/.bin:/opt/venv/bin:/opt/bin:/mise/installs/node/latest/bin:/mise/installs/ruby/latest/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin'" > /etc/profile.d/mise-path.sh && \
    chmod +x /etc/profile.d/mise-path.sh

# Create a wrapper script that sources profile and executes commands
RUN echo '#!/bin/bash' > /entrypoint.sh && \
    echo 'source /etc/profile.d/mise-path.sh' >> /entrypoint.sh && \
    echo 'exec "$@"' >> /entrypoint.sh && \
    chmod +x /entrypoint.sh

# Set ENTRYPOINT to the wrapper script
ENTRYPOINT ["/entrypoint.sh"]
CMD ["/bin/bash"]
