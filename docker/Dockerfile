# Build these with the build.py script
# Example:
#   python3 docker/build.py --tag dev --arch amd64 --build-type docker build

# One of "docker", "hassio"
ARG BASEIMGTYPE=docker

# https://github.com/hassio-addons/addon-debian-base/releases
FROM ghcr.io/hassio-addons/debian-base:6.2.0 AS base-hassio
# https://hub.docker.com/_/debian?tab=tags&page=1&name=bullseye
FROM debian:bullseye-20221024-slim AS base-docker

FROM base-${BASEIMGTYPE} AS base

ARG TARGETARCH
ARG TARGETVARIANT

RUN \
    apt-get update \
    # Use pinned versions so that we get updates with build caching
    && apt-get install -y --no-install-recommends \
        python3 \
        python3-pip \
        python3-setuptools \
        python3-pil \
        python3-cryptography \
        iputils-ping \
        git \
        curl \
        openssh-client \
    && rm -rf \
        /tmp/* \
        /var/{cache,log}/* \
        /var/lib/apt/lists/*

ENV \
  # Fix click python3 lang warning https://click.palletsprojects.com/en/7.x/python3/
  LANG=C.UTF-8 LC_ALL=C.UTF-8 \
  # Store globally installed pio libs in /piolibs
  PLATFORMIO_GLOBALLIB_DIR=/piolibs

# Support legacy binaries on Debian multiarch system. There is no "correct" way
# to do this, other than using properly built toolchains...
# See: https://unix.stackexchange.com/questions/553743/correct-way-to-add-lib-ld-linux-so-3-in-debian
RUN \
    if [ "$TARGETARCH$TARGETVARIANT" = "armv7" ]; then \
        ln -s /lib/arm-linux-gnueabihf/ld-linux.so.3 /lib/ld-linux.so.3; \
    fi

RUN \
    # Ubuntu python3-pip is missing wheel
    pip3 install --no-cache-dir \
        wheel \
        platformio \
    # Change some platformio settings
    && platformio settings set enable_telemetry No \
    && platformio settings set check_platformio_interval 1000000 \
    && mkdir -p /piolibs


# First install requirements to leverage caching when requirements don't change
COPY requirements.txt requirements_optional.txt docker/platformio_install_deps.py platformio.ini /
RUN \
    pip3 install --no-cache-dir -r /requirements.txt -r /requirements_optional.txt \
    && /platformio_install_deps.py /platformio.ini


# ======================= docker-type image =======================
FROM base AS docker

# Copy esphome and install
COPY . /esphome
RUN pip3 install --no-cache-dir --no-use-pep517 -e /esphome

# Settings for dashboard
ENV USERNAME="" PASSWORD=""

# Expose the dashboard to Docker
EXPOSE 6052

COPY docker/docker_entrypoint.sh /entrypoint.sh

# The directory the user should mount their configuration files to
VOLUME /config
WORKDIR /config
# Set entrypoint to esphome (via a script) so that the user doesn't have to type 'esphome'
# in every docker command twice
ENTRYPOINT ["/entrypoint.sh"]
# When no arguments given, start the dashboard in the workdir
CMD ["dashboard", "/config"]




# ======================= hassio-type image =======================
FROM base AS hassio

RUN \
    apt-get update \
    # Use pinned versions so that we get updates with build caching
    && apt-get install -y --no-install-recommends \
        nginx-light \
    && rm -rf \
        /tmp/* \
        /var/{cache,log}/* \
        /var/lib/apt/lists/*

ARG BUILD_VERSION=dev

# Copy root filesystem
COPY docker/ha-addon-rootfs/ /

# Copy esphome and install
COPY . /esphome
RUN pip3 install --no-cache-dir --no-use-pep517 -e /esphome

# Labels
LABEL \
    io.hass.name="ESPHome" \
    io.hass.description="Manage and program ESP8266/ESP32 microcontrollers through YAML configuration files" \
    io.hass.type="addon" \
    io.hass.version="${BUILD_VERSION}"
    # io.hass.arch is inherited from addon-debian-base




# ======================= lint-type image =======================
FROM base AS lint

ENV \
  PLATFORMIO_CORE_DIR=/esphome/.temp/platformio

RUN \
    apt-get update \
    # Use pinned versions so that we get updates with build caching
    && apt-get install -y --no-install-recommends \
        clang-format-11 \
        clang-tidy-11 \
        patch \
        software-properties-common \
        nano \
        build-essential \
        python3-dev \
    && rm -rf \
        /tmp/* \
        /var/{cache,log}/* \
        /var/lib/apt/lists/*

COPY requirements_test.txt /
RUN pip3 install --no-cache-dir -r /requirements_test.txt

VOLUME ["/esphome"]
WORKDIR /esphome
