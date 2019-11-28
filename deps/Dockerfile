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
FROM python:3.8.0-slim-buster

# Allows for log messages to be immediately dumped to the 
# stream instead of being buffered.
ENV PYTHONUNBUFFERED 1

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
        libssl-dev \
	&& rm -rf /var/lib/apt/lists/*

# Setup virtualenv
RUN python -m venv ${PYTHON_LIBRARY_PATH}

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


# NTL
COPY --from=sbellem/ntl:buster /usr/local/include/NTL /usr/local/include/NTL
COPY --from=sbellem/ntl:buster /usr/local/lib/libntl.a /usr/local/lib/libntl.a

# PBC
COPY --from=sbellem/pbc:0.5.14-slim-buster /usr/local/include/pbc /usr/local/include/pbc
COPY --from=sbellem/pbc:0.5.14-slim-buster /usr/local/lib/libpbc.* /usr/local/lib/

# Rust
ENV RUSTUP_HOME /usr/local/rustup
ENV CARGO_HOME /usr/local/cargo
COPY --from=sbellem/rust-frozen:2019.11.21-slim-buster $RUSTUP_HOME $RUSTUP_HOME
COPY --from=sbellem/rust-frozen:2019.11.21-slim-buster $CARGO_HOME $CARGO_HOME
ENV PATH "${CARGO_HOME}/bin:${PATH}"

# Charm Python library
COPY --from=sbellem/charm-crypto:slim-buster ${PYTHON_LIBRARY_PATH} ${PYTHON_LIBRARY_PATH}
