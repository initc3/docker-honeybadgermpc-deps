# - Base 
#   - Install baseline dependencies used to build our main dependencies (e.g. cffi,
#     make, etc.)
#   - Commands in this section should be changed as little as possible to improve 
#     cache performance
# - Dependencies
#   - This is composed of a bunch of different targets, each of which inherit from
#     the base target, and create a single dependency. 
#   - It's essential to only copy what's necessary from the build context in this 
#     stage to improve caching

##
# Base target:
# All used targets should be based off of this target, and as such, changes to this 
# should be kept to an absolute minimum, as it causes the longest builds.
# This should contain all setup required by all other targets, such as environment
# variables, and essential apt dependencies.
##
FROM python:3.8.0-slim-buster AS base

# Allows for log messages to be immediately dumped to the 
# stream instead of being buffered.
ENV PYTHONUNBUFFERED 1

# Path variables needed for Charm
ENV LIBRARY_PATH /usr/local/lib
ENV LD_LIBRARY_PATH /usr/local/lib
ENV LIBRARY_INCLUDE_PATH /usr/local/include

# NOTE: Could use /usr/local/lib/python3.8/site-packages instead
ENV PYTHON_LIBRARY_PATH /opt/venv
ENV PATH ${PYTHON_LIBRARY_PATH}/bin:${PATH}

# Make sh point to bash
# This is being changed since it will avoid any errors in the `launch_mpc.sh` script
# which relies on certain code that doesn't work in container's default shell.
RUN ln -sf bash /bin/sh

# Install apt dependencies
# Put apt dependencies here that are needed by all build paths
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    curl \
    git \
    iproute2 \
    libflint-dev \
    libgmp-dev \
    libffi-dev \
    libmpc-dev \
    libmpfr-dev \
    libssl-dev

# Setup virtualenv
RUN pip install --upgrade pip virtualenv
RUN python -m virtualenv ${PYTHON_LIBRARY_PATH}

# Install pip dependencies here that are absolutely required by setup.py for 
# better cache performance. These should be changed rarely, as they cause
# long rebuild times.
RUN pip install \
    cffi \
    Cython \
    gmpy2 \
    psutil \
    pycrypto \
    pyzmq \
    zfec

# This is needed otherwise the build for the power sum solver will fail.
# This is a known issue in the version of libflint-dev in apt.
# https://github.com/wbhart/flint2/issues/217
# This has been fixed if we pull the latest code from the repo. However, we want
# to avoid compiling the lib from the source since it adds 20 minutes to the build.
RUN sed -i '30c #include "flint/flint.h"' /usr/include/flint/flintxx/flint_classes.h

##
# Build Target:
# Isolated target to build heavy dependencies
# Built dependencies must be manually copied over in later stages.
##
FROM base AS build
WORKDIR /

# Install apt dependencies. These dependencies should only be those which are
# needed for building dependencies. Any other dependencies should be installed 
# in later targets
RUN apt-get install -y --no-install-recommends \
    bison \
    cmake \
    flex \
    wget

# Install NTL
WORKDIR /
RUN curl -so - https://www.shoup.net/ntl/ntl-11.3.2.tar.gz | tar xzvf -
WORKDIR /ntl-11.3.2/src
RUN ./configure CXXFLAGS="-g -O2 -fPIC -march=native -pthread -std=c++11"
RUN make
RUN make install

# Install better pairing
# Creates dependencies in /usr/local/include/pbc and /usr/local/lib
WORKDIR /
RUN curl -so - https://crypto.stanford.edu/pbc/files/pbc-0.5.14.tar.gz | tar xzvf -
WORKDIR /pbc-0.5.14/
RUN ./configure
RUN make
RUN make install

# Install charm
# Creates /charm/dist/Charm_Crypto...x86_64.egg, which gets copied into the venv
# /opt/venv/lib/python3.7/site-packages/Charm_crypto...x86_64.egg
WORKDIR /
RUN git clone https://github.com/JHUISI/charm.git
WORKDIR /charm
RUN git reset --hard be9587ccdd4d61c591fb50728ebf2a4690a2064f
RUN ./configure.sh
RUN make install

# Downloads rust and sets it up
RUN curl https://sh.rustup.rs -sSf | sh -s -- -y --default-toolchain nightly-2019-11-16
ENV PATH "/root/.cargo/bin:${PATH}"

FROM base
COPY --from=build ${PYTHON_LIBRARY_PATH} ${PYTHON_LIBRARY_PATH}
COPY --from=build /usr/local/include/ /usr/local/include/
COPY --from=build ${LIBRARY_PATH} ${LIBRARY_PATH}
COPY --from=build /root/.cargo /root/.cargo
COPY --from=build /root/.rustup /root/.rustup
ENV PATH "/root/.cargo/bin:${PATH}"
