# cannot use a musl based image, as the following expects dynamically linked libs
FROM rust:1-slim

# add build dependencies
RUN apt-get update && apt-get install -y \
    curl build-essential tcl-dev valgrind unzip \
    && rm -rf /var/lib/apt/lists/*

# create non-root user to execute the tests (some tests fail if executed as root)
RUN addgroup -gid 1000 --system sqlite && \
    adduser -uid 1000 --system --gid 1000 --home /home/sqlite --shell /bin/bash sqlite && \
    passwd -d sqlite
WORKDIR /home/sqlite

# download and extract sqlite source
RUN curl -LO https://www.sqlite.org/2022/sqlite-src-3370200.zip && unzip sqlite-src-3370200.zip

# build a stub of the test vfs, which allows to build and cache sqlite before building the actual
# test vfs
ADD --chown=sqlite docker/test-vfs /home/sqlite/test-vfs
RUN cd test-vfs && cargo build

# build sqlite test programs
COPY patch/Makefile.in.patch Makefile.in.patch
RUN patch -u sqlite-src-3370200/Makefile.in Makefile.in.patch
RUN mkdir build && cd build && ../sqlite-src-3370200/configure
COPY patch/test_ext.c sqlite-src-3370200/src/test_ext.c
# -DSQLITE_EXTRA_INIT=sqlite3_register_test_vfs: custom init method that initializes the custom
#  test vfs
# -DSQLITE_MAX_MMAP_SIZE=0: disable memory mapping as it is not supported for the custom vfs
# -DSQLITE_DISABLE_DIRSYNC: disable to skip some tests that are not relevant for a custom vfs
RUN cd build && make \
    OPTS="-DSQLITE_EXTRA_INIT=sqlite3_register_test_vfs -DSQLITE_MAX_MMAP_SIZE=0 -DSQLITE_DISABLE_DIRSYNC" \
    LIBS="-L/home/sqlite/test-vfs/target/debug/ -ltest_vfs" \
    USE_AMALGAMATION=0 \
    testfixture
RUN chown -R sqlite:sqlite /home/sqlite/build
RUN rm -rf /home/sqlite/test-vfs

# Patch/skip some tests
COPY patch/test patch
COPY patch.sh .
RUN cd sqlite-src-3370200 && ../patch.sh && rm -rf ../{patch,patch.sh}

RUN mkdir /home/sqlite/lib/
ENV LD_LIBRARY_PATH /home/sqlite/lib/

WORKDIR /github/workspace
COPY entrypoint.sh /
ENTRYPOINT ["/entrypoint.sh"]
