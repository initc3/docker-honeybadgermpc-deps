FROM ubuntu

RUN apt-get update && apt-get install -y --no-install-recommends \
    curl \
    ca-certificates \
    make \
    g++ \
    libgmp-dev

# Install NTL
WORKDIR /
RUN curl -so - https://www.shoup.net/ntl/ntl-11.3.2.tar.gz | tar xzvf -
WORKDIR /ntl-11.3.2/src
RUN ./configure CXXFLAGS="-g -O2 -fPIC -march=native -pthread -std=c++11"
RUN make
RUN make install
