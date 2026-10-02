# SPDX-FileCopyrightText: 2025 Contributors to the Media eXchange Layer project.
# SPDX-License-Identifier: Apache-2.0
FROM debian:trixie-slim AS base

RUN apt-get update && apt-get install -y \
    gstreamer1.0-plugins-good \
    gstreamer1.0-plugins-base \
    gstreamer1.0-plugins-bad \
    gstreamer1.0-plugins-ugly \
    gstreamer1.0-tools \
    gstreamer1.0-x \
    dumb-init \
    libibverbs1 \
    ibverbs-providers \
    librdmacm1 \
    && rm -r /var/lib/apt/lists/*

#-----------------------------------------------------------------
FROM base AS builder

ENV HOME="/root" 

ARG CLANG_VERSION=19
RUN apt-get update && apt-get install -y \
    "clang-${CLANG_VERSION}" \
    "clang-tools-${CLANG_VERSION}" \
    "clang-tidy-${CLANG_VERSION}" \
    "clang-format-${CLANG_VERSION}" \
    "libclang-rt-${CLANG_VERSION}-dev" \
    build-essential bison flex \
    cmake ninja-build \
    curl zip unzip tar \
    pkg-config \
    git \
    autoconf \
    automake \
    libtool \
    libgstreamer1.0-dev \
    libgstreamer-plugins-base1.0-dev \
    libfabric-dev \
    libibverbs-dev \
    librdmacm-dev

RUN git clone https://github.com/microsoft/vcpkg.git "${HOME}/vcpkg" \
    && cd "${HOME}/vcpkg" \
    && ./bootstrap-vcpkg.sh -disableMetrics
ENV VCPKG_ROOT="${HOME}/vcpkg"

ENV MXL_REPO_ROOT="${HOME}/mxl"
RUN mkdir "$MXL_REPO_ROOT"
COPY ./ "$MXL_REPO_ROOT"

# Install custom libfabric required by MXL
RUN ${MXL_REPO_ROOT}/.devcontainer/scripts/common/libfabric/install.sh

# Build MXL SDK
RUN cmake -S "$MXL_REPO_ROOT" --preset Linux-GCC-Release -DCMAKE_INSTALL_PREFIX=/usr -DMXL_ENABLE_FABRICS_OFI=ON
RUN ninja -C "${MXL_REPO_ROOT}/build/Linux-GCC-Release"
RUN ninja -C "${MXL_REPO_ROOT}/build/Linux-GCC-Release" install

# Install GST dependencies for rust plugins
RUN apt-get install -y --no-install-recommends \
    libglib2.0-dev \
    libcairo2-dev \
    libpango1.0-dev \
    libgraphene-1.0-dev \
    libgtk-4-dev \
    libgirepository1.0-dev \
    libatk1.0-dev \
    rustup

RUN ${MXL_REPO_ROOT}/.devcontainer/scripts/common/rust/install-rust.sh
ENV CARGO_HOME="${HOME}/.cargo/"
ENV RUSTUP_HOME="${HOME}/.rustup/"

RUN cd "${MXL_REPO_ROOT}/rust/" \
    && cargo build --release --features mxl-sys/mxl-not-built,mxl/mxl-not-built \
    && install -m 0644 "${MXL_REPO_ROOT}/rust/target/release/libgstmxl.so" \
         "/usr/lib/$(gcc -dumpmachine)/gstreamer-1.0/"
RUN git clone --depth 1 --branch 0.14.5 \
        https://gitlab.freedesktop.org/gstreamer/gst-plugins-rs.git \
        /tmp/gst-plugins-rs \
    && cd /tmp/gst-plugins-rs \
    && cargo build --release -p gst-plugin-closedcaption \
    && install -m 0644 target/release/libgstrsclosedcaption.so \
         "/usr/lib/$(gcc -dumpmachine)/gstreamer-1.0/" \
    && cd - \
    && rm -rf /tmp/gst-plugins-rs

# ------------------------------------------------------------
# Runtime images
FROM base AS mxl

ARG  LIB="/usr/lib/x86_64-linux-gnu"
COPY --from=builder "${LIB}/libmxl*" "${LIB}/"

# Copy compiled libfabric directly into multiarch directory to avoid system overrides
COPY --from=builder "/usr/lib/libfabric*" "${LIB}/"

# Copy compiled libfabric tools (fi_info, etc.) and MXL binaries
COPY --from=builder "/usr/bin/fi_*" "/usr/bin/"
COPY --from=builder "/usr/bin/mxl-*" "/usr/bin/"
COPY --from=builder "${LIB}/gstreamer-1.0/libgstmxl.so" "${LIB}/gstreamer-1.0/"
COPY --from=builder "${LIB}/gstreamer-1.0/libgstrsclosedcaption.so" "${LIB}/gstreamer-1.0/"

RUN useradd -m mxl
USER mxl
WORKDIR /home/mxl

FROM mxl AS gst-mxl-rs
ENTRYPOINT ["gst-launch-1.0"]

FROM mxl AS mxl-info
ENTRYPOINT ["/usr/bin/mxl-info"]

FROM mxl AS mxl-gst-testsrc
ENTRYPOINT ["/usr/bin/mxl-gst-testsrc"]

FROM mxl AS mxl-fake-reader
COPY examples/scripts/fake-reader.sh /home/mxl/fake-reader.sh
ENTRYPOINT ["/usr/bin/dumb-init", "--", "/home/mxl/fake-reader.sh"]
