ARG BUILD_FROM
ARG BUILD_ARCH
FROM --platform=linux/amd64 debian:bullseye AS cross-builder-base

ENV \
    LANG="C.UTF-8" \
    DEBIAN_FRONTEND="noninteractive" \
    CURL_CA_BUNDLE="/etc/ssl/certs/ca-certificates.crt"

WORKDIR /usr/src

# Allow to reuse downloaded packages (these are only staged build images)
# hadolint ignore=DL3009
RUN \
    set -x \
    && apt-get update \
    && apt-get install -y --no-install-recommends \
       bash \
       curl \
       ca-certificates \
       build-essential \
       git

FROM --platform=linux/amd64 cross-builder-base AS cross-builder-amd64

COPY debian-amd64.cmake /usr/src/debian.cmake

ENV DEBIAN_ARCH=amd64
ENV DEBIAN_CROSS_PREFIX=x86_64-linux-gnu
ENV SLC_ARCH=linux_arch_64

RUN \
    set -x \
    && dpkg --add-architecture armhf \
    && apt-get update \
    && apt-get install -y --no-install-recommends \
       crossbuild-essential-amd64

FROM --platform=linux/amd64 cross-builder-base AS cross-builder-armv7

COPY debian-armv7.cmake /usr/src/debian.cmake

ENV DEBIAN_ARCH=armhf
ENV DEBIAN_CROSS_PREFIX=arm-linux-gnueabihf
ENV SLC_ARCH=linux_arch_32

RUN \
    set -x \
    && dpkg --add-architecture armhf \
    && apt-get update \
    && apt-get install -y --no-install-recommends \
       crossbuild-essential-armhf

FROM --platform=linux/amd64 cross-builder-base AS cross-builder-aarch64

COPY debian-arm64.cmake /usr/src/debian.cmake

ENV DEBIAN_ARCH=arm64
ENV DEBIAN_CROSS_PREFIX=aarch64-linux-gnu
ENV SLC_ARCH=linux_arch_64

RUN \
    set -x \
    && dpkg --add-architecture arm64 \
    && apt-get update \
    && apt-get install -y --no-install-recommends \
       crossbuild-essential-arm64


FROM --platform=linux/amd64 cross-builder-${BUILD_ARCH} AS cpcd-builder

ARG CPCD_VERSION=v4.1.1

RUN \
    set -x \
    && apt-get install -y --no-install-recommends \
       cmake \
       "libmbedtls-dev:${DEBIAN_ARCH}" \
       "libmbedtls12:${DEBIAN_ARCH}" \
    && git clone --depth 1 -b "${CPCD_VERSION}" \
       https://github.com/SiliconLabs/cpc-daemon.git \
    && mkdir cpc-daemon/build && cd cpc-daemon/build \
    && cmake ../ \
       -DCMAKE_TOOLCHAIN_FILE=../debian.cmake \
       -DENABLE_ENCRYPTION=FALSE \
    && make \
    && make install

FROM --platform=linux/amd64 cross-builder-${BUILD_ARCH} AS gecko-sdk-builder

ARG GECKO_SDK_VERSION=v4.1.1

RUN \
    set -x \
    && apt-get install -y --no-install-recommends \
       python3 \
       python3-jinja2 \
       python3-pip \
       openjdk-11-jre \
       git-lfs \
       unzip \
    && curl -O https://www.silabs.com/documents/login/software/slc_cli_linux.zip \
    && unzip slc_cli_linux.zip \
    && cd slc_cli/ && chmod +x slc
 
ENV PATH="/usr/src/slc_cli/:$PATH"

RUN \
    set -x \
    && git clone --depth 1 -b ${GECKO_SDK_VERSION} \
       https://github.com/SiliconLabs/gecko_sdk.git

FROM --platform=linux/amd64 cross-builder-${BUILD_ARCH} AS ezspd-builder

# ezspd links against libcpc.so
COPY --from=cpcd-builder /usr/local/ /usr/${DEBIAN_CROSS_PREFIX}/

RUN \
    set -x \
    && git clone https://github.com/agners/silabs-cpc-ezspd \
    && cd silabs-cpc-ezspd \
    && make \
        CC="${DEBIAN_CROSS_PREFIX}-gcc"

FROM $BUILD_FROM

RUN \
    set -x \
    && apt-get update \
    && apt-get install -y --no-install-recommends \
       libmbedtls12 \
       socat \
    && rm -rf /var/lib/apt/lists/* \
    && rm -rf /usr/src/*


COPY --from=gecko-sdk-builder \
     /usr/src/gecko_sdk/util/third_party/ot-br-posix /usr/src/ot-br-posix
COPY --from=gecko-sdk-builder \
     /usr/src/gecko_sdk/util/third_party/openthread /usr/src/openthread
COPY --from=cpcd-builder /usr/local/ /usr/local/

ENV BORDER_ROUTING=1
ENV BACKBONE_ROUTER=1
ENV OTBR_OPTIONS "-DOTBR_DBUS=OFF -DOT_MULTIPAN_RCP=ON -DOT_POSIX_CONFIG_RCP_BUS=CPC -DOT_LINK_RAW=1 -DOTBR_VENDOR_NAME=HomeAssistant -DOTBR_PRODUCT_NAME=OpenThreadBorderRouter"
ENV WEB_GUI=1
ENV DOCKER 1

COPY 0001-Avoid-writing-to-system-console.patch /usr/src

# Required and installed during build (script/bootstrap), could be removed
ENV OTBR_BUILD_DEPS build-essential ninja-build cmake wget ca-certificates \
  libreadline-dev libncurses-dev libcpputest-dev libdbus-1-dev libavahi-common-dev \
  libavahi-client-dev libboost-dev libboost-filesystem-dev libboost-system-dev \
  libnetfilter-queue-dev

# Build OTBR natively from Gecko SDK sources
WORKDIR /usr/src
RUN \
    set -x \
    && apt-get update \
    # Install npm/nodejs for WebUI manually to avoid systemd getting pulled in \
    && apt-get install -y --no-install-recommends \
       nodejs \
       npm \
       iproute2 \
       lsb-release \
       sudo \
    && cd ot-br-posix \
    && chmod +x ./script/bootstrap \
    && ./script/bootstrap \
    && ln -s ../../../openthread/ third_party/openthread/repo \
    && chmod +x ./script/* \
    && patch -p1 < /usr/src/0001-Avoid-writing-to-system-console.patch \
    && ./script/setup \
    && apt-get purge -y --auto-remove \
       build-essential \
       git \
       git-lfs \
       libmbedtls-dev \
       ${OTBR_BUILD_DEPS} \
    && rm -rf /var/lib/apt/lists/* \
    && rm -rf /usr/src/*

COPY --from=ezspd-builder \
     /usr/src/silabs-cpc-ezspd/ezspd \
     /usr/local/bin

RUN ldconfig && touch /accept_silabs_msla

COPY rootfs /

# use s6-overlay as init system
WORKDIR /
ENTRYPOINT ["/init"]
