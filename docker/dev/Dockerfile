# Basic cross-build environment
FROM ubuntu:20.04 as cross-build

ARG UID=1000
ARG GID=1000

ENV \
 ARCH=amd64 \
 HOST=x86_64-unknown-linux \
 TOOL=x86_64-linux-gnu \
 TARGET=x86_64-unknown-linux-gnu \
 CARGO_TARGET_X86_64_UNKNOWN_LINUX_GNU_LINKER=x86_64-linux-gnu-gcc \
 CC_X86_64_UNKNOWN_LINUX_GNU=x86_64-linux-gnu-gcc \
 CXX_X86_64_UNKNOWN_LINUX_GNU=x86_64-linux-gnu-g++ \
 BUILD_MODE=release

ENV \
 TOOLCHAIN=stable \
 DEBIAN_FRONTEND=noninteractive \
 PKG_CONFIG_ALLOW_CROSS=1 \
 PKG_CONFIG_PATH=/usr/lib/$TOOL/pkgconfig:/usr/lib/pkgconfig \
 LD_LIBRARY_PATH=/usr/lib/$TOOL:/usr/$TOOL/lib \
 LD_RUN_PATH=/usr/lib/$TOOL:/usr/$TOOL/lib \
 LDFLAGS="-L/usr/lib/$TOOL -L/usr/$TOOL/lib -Wl,-rpath-link,/usr/lib/$TOOL -Wl,-rpath-link,/usr/$TOOL/lib" \
 CFLAGS="-I/usr/include/$TOOL -I/usr/$TOOL/include -I/usr/include" \
 CPPFLAGS="-I/usr/include/$TOOL -I/usr/$TOOL/include -I/usr/include"

RUN \
 sed 's/http:\/\/\(.*\).ubuntu.com\/ubuntu\//[arch-=amd64,i386] http:\/\/ports.ubuntu.com\/ubuntu-ports\//g' /etc/apt/sources.list > /etc/apt/sources.list.d/ports.list && \
 sed -i 's/http:\/\/\(.*\).ubuntu.com\/ubuntu\//[arch=amd64,i386] http:\/\/\1.archive.ubuntu.com\/ubuntu\//g' /etc/apt/sources.list && \
 addgroup --gid $GID build && \
 adduser \
    --disabled-password \
    --gecos "" \
    --ingroup build \
    --uid $UID \
    --home /opt/build \
    build && \
 dpkg --add-architecture $ARCH && \
 apt-get update && \
 apt-get upgrade -y && \
 apt-get install -y \
    pkg-config \
    build-essential \
    crossbuild-essential-$ARCH

WORKDIR /opt/build


# Environment for ImageMagick
FROM cross-build as imagemagick-builder

RUN \
 apt-get install -y \
    libltdl-dev:$ARCH \
    libjpeg-dev:$ARCH \
    libpng-dev:$ARCH \
    libwebp-dev:$ARCH \
    liblzma-dev:$ARCH \
    libxml2-dev:$ARCH

ADD --chown=build:build https://imagemagick.org/download/ImageMagick.tar.gz /opt/build/ImageMagick.tar.gz

USER build

RUN \
 tar zxf ImageMagick.tar.gz && \
 mv ImageMagick-* ImageMagick

WORKDIR /opt/build/ImageMagick

RUN \
 ./configure \
    CC=$TOOL-gcc \
    CXX=$TOOL-g++ \
    --enable-shared \
    --with-modules \
    --disable-static \
    --disable-docs \
    --prefix=/usr/local \
    --with-utilities=no \
    --with-magick-plus-plus=no \
    --without-perl \
    --with-xml=yes \
    --with-png=yes \
    --with-jpeg=yes \
    --with-webp=yes \
    --host=$HOST && \
 make

USER root

RUN \
 make install && \
 ldconfig /usr/local/lib


# Environment for Rust
FROM cross-build as rust

RUN \
 apt-get install -y curl

ENV \
 PATH=$PATH:/opt/build/.cargo/bin

ADD --chown=build:build https://sh.rustup.rs /opt/build/rustup.sh

USER build

RUN \
 chmod +x rustup.sh && \
 ./rustup.sh --default-toolchain $TOOLCHAIN --profile minimal -y && \
 rustup target add $TARGET

USER root


# Environment for pict-rs
FROM cross-build as pict-rs-builder

RUN \
 apt-get install -y \
    libgexiv2-dev:$ARCH \
    libxml2:$ARCH \
    libltdl7:$ARCH \
    llvm-dev \
    libclang-dev \
    clang \
    libpng16-16:$ARCH \
    libjpeg8:$ARCH \
    libwebp6:$ARCH \
    libwebpdemux2:$ARCH \
    libwebpmux3:$ARCH \
    libgomp1:$ARCH

ENV \
 PATH=$PATH:/opt/build/.cargo/bin \
 PKG_CONFIG_PATH=$PKG_CONFIG_PATH:/usr/local/lib/pkgconfig \
 LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/usr/local/lib \
 LD_RUN_PATH=$LD_RUN_PATH:/usr/local/lib \
 LDFLAGS="$LDFLAGS -L/usr/local/lib" \
 IMAGE_MAGICK_LIB_DIRS=/usr/local/lib \
 IMAGE_MAGICK_INCLUDE_DIRS=/usr/local/include/ImageMagick-7 \
 CFLAGS="$CFLAGS -I/usr/local/include/ImageMagick-7" \
 CPPFLAGS="$CPPFLAGS -I/usr/local/include/ImageMagick-7" \
 RUSTFLAGS="-L/usr/lib/$TOOL -C link-arg=-Wl,-rpath-link,/usr/lib/$TOOL -L/usr/$TOOL/lib -C link-arg=-Wl,-rpath-link,/usr/$TOOL/lib"

COPY --from=rust --chown=build:build /opt/build/.cargo /opt/build/.cargo
COPY --from=rust --chown=build:build /opt/build/.rustup /opt/build/.rustup
COPY --from=imagemagick-builder /usr/local/ /usr/local

USER build

RUN \
 mkdir -p /opt/build/repo

WORKDIR /opt/build/repo
