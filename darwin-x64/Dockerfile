FROM cross-compiler:darwin-x64

RUN mkdir -p /build
WORKDIR /build

# Fix Boost using wrong archiver / ignoring <archiver> flags
# https://svn.boost.org/trac/boost/ticket/12573
# https://github.com/boostorg/build/blob/boost-1.63.0/src/tools/clang-darwin.jam#L133
RUN mv /usr/bin/ar /usr/bin/ar.orig && \
    mv /usr/bin/strip /usr/bin/strip.orig && \
    mv /usr/bin/ranlib /usr/bin/ranlib.orig && \
    ln -sf ${CROSS_ROOT}/bin/${CROSS_TRIPLE}-ar /usr/bin/ar && \
    ln -sf ${CROSS_ROOT}/bin/${CROSS_TRIPLE}-strip /usr/bin/strip && \
    ln -sf ${CROSS_ROOT}/bin/${CROSS_TRIPLE}-ranlib /usr/bin/ranlib

# Install Boost.System
ENV BOOST_VERSION 1.63.0
RUN curl -L https://sourceforge.net/projects/boost/files/boost/${BOOST_VERSION}/boost_`echo ${BOOST_VERSION} | sed 's/\\./_/g'`.tar.bz2/download | tar xj && \
    cd boost_`echo ${BOOST_VERSION} | sed 's/\\./_/g'`/ && \
    ./bootstrap.sh --prefix=${CROSS_ROOT} --with-toolset=clang && \
    echo "using clang : darwin : ${CROSS_TRIPLE}-c++ ;" > ${HOME}/user-config.jam && \
    ./b2 --with-date_time --with-system --with-chrono --with-random --prefix=${CROSS_ROOT} toolset=clang-darwin link=static variant=release threading=multi target-os=darwin install 1>log 2>err && \
    rm -rf ${HOME}/user-config.jam && \
    rm -rf `pwd`

# Move back ar, strip and ranlib...
RUN mv /usr/bin/ar.orig /usr/bin/ar && \
    mv /usr/bin/strip.orig /usr/bin/strip && \
    mv /usr/bin/ranlib.orig /usr/bin/ranlib

# Install OpenSSL
ENV OPENSSL_VERSION 1.0.2k
ENV OPENSSL_SHA256 6b3977c61f2aedf0f96367dcfb5c6e578cf37e7b8d913b4ecb6643c3cb88d8c0
RUN wget -q https://www.openssl.org/source/openssl-${OPENSSL_VERSION}.tar.gz -O openssl.tar.gz && \
    echo "$OPENSSL_SHA256  openssl.tar.gz" | sha256sum -c - && \
    tar -xzf openssl.tar.gz && \
    rm openssl.tar.gz && \
    cd openssl-${OPENSSL_VERSION}/ && \
    CROSS_COMPILE=${CROSS_TRIPLE}- ./Configure threads no-shared darwin64-x86_64-cc --prefix=${CROSS_ROOT} 1>log 2>err && \
    make -j $(cat /proc/cpuinfo | grep processor | wc -l) 1>log 2>err && \
    make install 1>log 2>err && \
    rm -rf `pwd`

# Install SWIG
ENV SWIG_VERSION 3.0.12
RUN curl -L https://github.com/swig/swig/archive/rel-${SWIG_VERSION}.tar.gz | tar xz && \
    cd swig-rel-${SWIG_VERSION}/ && \
    ./autogen.sh && \
    ./configure 1>log 2>err && \
    make -j $(cat /proc/cpuinfo | grep processor | wc -l) 1>log 2>err && \
    make install 1>log 2>err && \
    rm -rf `pwd`

# Install Golang
ENV GOLANG_VERSION 1.8
ENV GOLANG_SRC_URL https://golang.org/dl/go$GOLANG_VERSION.src.tar.gz
ENV GOLANG_SRC_SHA256 406865f587b44be7092f206d73fc1de252600b79b3cacc587b74b5ef5c623596

ENV GOLANG_BOOTSTRAP_VERSION 1.4.3
ENV GOLANG_BOOTSTRAP_URL https://golang.org/dl/go$GOLANG_BOOTSTRAP_VERSION.src.tar.gz
ENV GOLANG_BOOTSTRAP_SHA1 486db10dc571a55c8d795365070f66d343458c48

RUN set -ex \
  && mkdir -p /usr/local/bootstrap \
  && wget -q "$GOLANG_BOOTSTRAP_URL" -O golang.tar.gz \
  && echo "$GOLANG_BOOTSTRAP_SHA1  golang.tar.gz" | sha1sum -c - \
  && tar -C /usr/local/bootstrap -xzf golang.tar.gz \
  && rm golang.tar.gz \
  && cd /usr/local/bootstrap/go/src \
  && ./make.bash 1>golang.log 2>golang.err \
  && export GOROOT_BOOTSTRAP=/usr/local/bootstrap/go \
  \
  && wget -q "$GOLANG_SRC_URL" -O golang.tar.gz \
  && echo "$GOLANG_SRC_SHA256  golang.tar.gz" | sha256sum -c - \
  && tar -C /usr/local -xzf golang.tar.gz \
  && rm golang.tar.gz \
  && cd /usr/local/go/src \
  && ./make.bash 1>golang.log 2>golang.err \
  \
  && CC_FOR_TARGET=${CROSS_TRIPLE}-cc CXX_FOR_TARGET=${CROSS_TRIPLE}-c++ GOOS=darwin GOARCH=amd64 CGO_ENABLED=1 ./make.bash --no-clean \
  && rm -rf /usr/local/bootstrap /usr/local/go/pkg/bootstrap
ENV PATH ${PATH}:/usr/local/go/bin

# Install libtorrent
ENV LIBTORRENT_VERSION 430ff6d7b45f4d30d51593a026ca4c7b3db0e9be
RUN curl -L https://github.com/arvidn/libtorrent/archive/`echo ${LIBTORRENT_VERSION} | sed 's/\\./_/g'`.tar.gz | tar xz && \
    cd libtorrent-`echo ${LIBTORRENT_VERSION} | sed 's/\\./_/g'`/ && \
    ./autotool.sh && \
    \
    sed -i 's/$PKG_CONFIG openssl --libs-only-/$PKG_CONFIG openssl --static --libs-only-/' ./configure && \
    \
    OSXCROSS_PKG_CONFIG_PATH=${CROSS_ROOT}/lib/pkgconfig/ \
    CC=${CROSS_TRIPLE}-cc CXX=${CROSS_TRIPLE}-c++ \
    CFLAGS="${CFLAGS} -O2" \
    CXXFLAGS="${CXXFLAGS} ${CFLAGS} -O2" \
    ./configure \
        --enable-static \
        --disable-shared \
        --disable-deprecated-functions \
        --host=${CROSS_TRIPLE} \
        --prefix=${CROSS_ROOT} \
        --with-boost=${CROSS_ROOT} --with-boost-libdir=${CROSS_ROOT}/lib && \
    \
    make -j $(cat /proc/cpuinfo | grep processor | wc -l) && make install && \
    rm -rf `pwd`

WORKDIR /
