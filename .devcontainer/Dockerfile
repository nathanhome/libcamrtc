FROM --platform=$BUILDPLATFORM debian:bookworm-slim AS webrtc_builder

 # webrtc release m125 as default,
 # see https://chromiumdash.appspot.com/branches for newer commit numbers of releases
ARG WEBRTC_STABLE=6422  
ARG LLVM_VERSION=18.1.5  

# Set environment variables
ENV DEBIAN_FRONTEND=noninteractive

# prepare arm64 build environment
# sudo is needed as a placeholder for the install-build-deps.sh
# apt-get install -y python-is-python3 python3 pulseaudio libpulse-dev libncurses5 libx11-dev && \
RUN \
    apt-get update && \
    apt-get install -y curl git lsb-release git file vim sudo build-essential && \
    apt-get install -y python-is-python3 python3 python3-pip python3-venv && \
    rm -rf /var/lib/apt/lists/* && \
    mkdir -p /build/webrtc/out
WORKDIR /build

# get google depot tools
RUN \
    git clone --depth 1 https://chromium.googlesource.com/chromium/tools/depot_tools.git
ENV PATH="/build/depot_tools:${PATH}"

# get current LLVM
RUN \
    curl -s -L https://github.com/llvm/llvm-project/releases/download/llvmorg-${LLVM_VERSION}/clang+llvm-${LLVM_VERSION}-aarch64-linux-gnu.tar.xz \
    | tar Jxvf - -C /build
ENV PATH=/build/clang+llvm-${LLVM_VERSION}-aarch64-linux-gnu/bin:${PATH}
ENV LLVM_DIR=/build/clang+llvm-${LLVM_VERSION}-aarch64-linux-gnu

# WebRTC build sequence

# get webrtc source, prepare build
WORKDIR /build/webrtc
RUN \
    fetch --nohooks webrtc 

WORKDIR /build/webrtc/src  
RUN \
    # ./build/linux/sysroot_scripts/install-sysroot.py --arch=arm64 && \
    ./build/install-build-deps.sh

RUN \
    git config --global user.name "doorstation" && \
    git config --global user.email "doorstation@users.noreply.github.com" && \
    git config --global core.autocrlf false && \
    git config --global core.filemode false && \
    gclient sync --with_branch_heads --with_tags && \
    git fetch && \
    git checkout -b branch_${WEBRTC_STABLE} branch-heads/${WEBRTC_STABLE} && \
    gclient sync --with_branch_heads --with_tags

# compile DEBUG version

RUN \
    gn gen /build/webrtc/out/Release --args="is_debug=false is_clang=true clang_use_chrome_plugins=false clang_base_path=\"$LLVM_DIR\" target_os=\"linux\" target_cpu=\"arm64\" use_custom_libcxx=true rtc_include_tests=false rtc_build_tools=false rtc_use_pipewire=false enable_iterator_debugging=false treat_warnings_as_errors=false "  && \
    ninja -C /build/webrtc/out/Release           


###
# collect libcamera dev files and webrtc static artefacts to form a proper
# build platform for the python module. The module eventually conatains verything
# to implement a python client with proper signalling for the use case
FROM --platform=$BUILDPLATFORM debian:bookworm-slim


RUN \
    apt-get update && \
    apt-get install -y curl git build-essential libcamera-dev && \
    apt-get install -y python-is-python3 python3 python3-pip python3-venv && \
    rm -rf /var/lib/apt/lists/* && \
    mkdir -p /webrtc
WORKDIR /webrtc

# install compiled webrtc library
RUN --mount=type=bind,from=webrtc_builder,source=/build,target=/build \
    find /build/webrtc/src -type f \( -name "*.h" -o -name "*.hpp" \) \
    | grep -vE "(/|^)(examples|test|testing|src\/build|rtc_tools|tools|buildtools|sdk)(/|$)" \
    | while read -r file; do\
        relative_path=${file#"/build/webrtc/src/"}; \
        mkdir -p "/webrtc/include/$(dirname "$relative_path")"; \
        cp $file /webrtc/include/$relative_path; \
    done \
    && mkdir -p /webrtc/lib /webrtc/examples \
    && cp /build/webrtc/out/Release/obj/libwebrtc.a /webrtc/lib \     
    && cp -R /build/webrtc/src/examples/ /webrtc/examples

# this allows either overriding CMD to start the container with a shell or `docker exec` a shell on running container
CMD [ "sleep", "infinity" ]