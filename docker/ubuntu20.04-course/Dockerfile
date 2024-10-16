FROM sydr/ubuntu20.04-sydr-fuzz

ARG DEBIAN_FRONTEND=noninteractive

# Build SymQEMU

RUN apt-get update && \
    apt-get -y install libz3-dev zlib1g-dev && \
    pip3 install lit

RUN git clone https://github.com/eurecom-s3/symqemu.git && \
    cd symqemu && \
    git submodule update --init --recursive subprojects/symcc-rt/
RUN git clone https://github.com/eurecom-s3/symcc.git && \
    cd symcc && \
    git submodule update --init --recursive

RUN mkdir /symqemu/build
WORKDIR /symqemu/build
RUN ../configure --audio-drv-list= --disable-sdl --disable-gtk --disable-vte --disable-opengl --disable-virglrenderer --disable-werror --target-list=x86_64-linux-user
RUN make -j
RUN make check

COPY symcc_helper.patch /
WORKDIR /symcc
RUN cd util/symcc_fuzzing_helper && git apply /symcc_helper.patch
RUN cargo install --path util/symcc_fuzzing_helper

# Build FUZZOLIC

WORKDIR /

RUN apt-get update && \
    apt-get -y install qemu-user libglib2.0-dev libfdt-dev \
    libpixman-1-dev libcapstone-dev strace libprotobuf-dev \
    libibverbs-dev libjpeg62-dev libpng16-16 libjbig-dev \
    libtool-bin automake flex bison python3-setuptools libtool

RUN git clone https://github.com/season-lab/fuzzolic.git && cd fuzzolic && \
    git submodule update --init && cd solver/fuzzy-sat && git fetch && \
    git submodule sync && git submodule update --init

WORKDIR /fuzzolic

# Build QEMU tracer
RUN cd tracer && ./configure --prefix=`pwd`/../build --target-list=x86_64-linux-user && make -j `nproc`

# Build custom Z3
RUN cd solver/fuzzy-sat/fuzzolic-z3 && mkdir build && cd build && cmake .. -DCMAKE_BUILD_TYPE=Release -DCMAKE_INSTALL_PREFIX=`pwd`/dist && make -j `nproc` && make install

ENV C_INCLUDE_PATH=/fuzzolic/solver/fuzzy-sat/fuzzolic-z3/build/dist/include
ENV LIBRARY_PATH=/fuzzolic/solver/fuzzy-sat/fuzzolic-z3/build/dist/lib
ENV LD_LIBRARY_PATH=/fuzzolic/solver/fuzzy-sat/fuzzolic-z3/build/dist/lib

RUN cd solver/fuzzy-sat && \
       git rev-parse HEAD > /tmp/revision && \
       git checkout master && \
       git submodule update && \
       cd ../.. && \
       cp -r solver/fuzzy-sat solver/fuzzy-sat-cli && \
       rm solver/fuzzy-sat-cli/.git && \
       cd solver/fuzzy-sat && \
       git checkout `cat /tmp/revision` && \
       git submodule update

# Build fuzzy-sat-CLI
RUN cd solver/fuzzy-sat-cli && make -j `nproc`

# Build fuzzy-sat
RUN cd solver/fuzzy-sat && make -j `nproc`

# Build solver frontend
RUN rm `find /usr -name "libz3.so*"`
RUN cd solver && cmake . && make -j `nproc`

ENV AFL_PATH=/usr/local/bin
RUN mkdir /AFLplusplus && cp /usr/local/bin/afl-showmap /AFLplusplus

# Build fuzzolic tests
RUN cd tests && make

# Fix symqemu
RUN apt reinstall -y libz3-4 libz3-dev
RUN ln -s /symqemu/build/qemu-x86_64 /symqemu/build/symqemu-x86_64
RUN ln -s /symqemu/build/symqemu-x86_64 /usr/local/bin/symqemu-x86_64

COPY *.README /
COPY StandaloneFuzzTargetMainStdin.c /opt/.

WORKDIR /
