# Dockerfile — Android Reverse Engineering Worker Container
# 预装 jadx/apktool/Ghidra/Frida/radare2/androguard/Android SDK
# 支持静态+动态分析，Frida Hook自动化

FROM ubuntu:22.04

ENV DEBIAN_FRONTEND=noninteractive
ENV HOME=/home/reverser
ENV ANDROID_HOME=/home/reverser/android-sdk
ENV GHIDRA_HOME=/home/reverser/tools/ghidra_11.2.1_PUBLIC

# ── 基础工具 + 换阿里云镜像源 (HTTP!) ──
# Ubuntu 22.04 基础镜像没有 ca-certificates，必须用 HTTP 不是 HTTPS
RUN sed -i 's|http://archive.ubuntu.com|http://mirrors.aliyun.com|g; s|http://security.ubuntu.com|http://mirrors.aliyun.com|g' /etc/apt/sources.list && \
    apt-get update && \
    apt-get install -y --no-install-recommends \
        build-essential git curl wget unzip sudo \
        python3 python3-pip python3-dev \
        openjdk-17-jdk-headless \
        socat \
        && rm -rf /var/lib/apt/lists/*

# ── 创建非root用户 ──
RUN useradd -m -s /bin/bash reverser && \
    echo "reverser ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers

USER reverser
WORKDIR /home/reverser

# ── Node.js v20 (官方tarball) ──
# dpkg --print-architecture 输出 amd64，但 Node tarball 用 x64
RUN ARCH=$(dpkg --print-architecture | sed 's/amd64/x64/') && \
    curl -fsSL "https://nodejs.org/dist/v20.18.0/node-v20.18.0-linux-${ARCH}.tar.xz" | sudo tar -xJ -C /usr/local --strip-components=1 && \
    node --version && npm --version

# ── jadx (Java反编译器) ──
RUN mkdir -p ~/tools && \
    cd ~/tools && \
    wget -q https://github.com/skylot/jadx/releases/download/v1.5.0/jadx-1.5.0.zip && \
    unzip jadx-1.5.0.zip -d jadx && rm jadx-1.5.0.zip && \
    chmod +x jadx/bin/jadx jadx/bin/jadx-gui
ENV PATH="/home/reverser/tools/jadx/bin:$PATH"

# ── apktool ──
RUN sudo wget -q https://raw.githubusercontent.com/iBotPeaches/Apktool/master/scripts/linux/apktool -O /usr/local/bin/apktool && \
    sudo wget -q "https://bitbucket.org/iBotPeaches/apktool/downloads/apktool_2.10.0.jar" -O /usr/local/bin/apktool.jar && \
    sudo chmod +x /usr/local/bin/apktool /usr/local/bin/apktool.jar

# ── Ghidra ──
ARG GHIDRA_URL=https://github.com/NationalSecurityAgency/ghidra/releases/download/Ghidra_11.2.1_build/ghidra_11.2.1_PUBLIC_20241105.zip
RUN wget -q "$GHIDRA_URL" -O /tmp/ghidra.zip && \
    cd ~/tools && unzip /tmp/ghidra.zip && rm /tmp/ghidra.zip
ENV PATH="/home/reverser/tools/ghidra_11.2.1_PUBLIC:$PATH"

# ── Frida + frida-tools ──
RUN python3 -m pip install --upgrade pip && \
    pip3 install frida-tools frida

# ── androguard (Python APK分析) ──
RUN pip3 install androguard

# ── unicorn + capstone (加壳 .so 模拟脱壳, 供 r0re_tools/android 的 runner 使用) ──
RUN pip3 install unicorn capstone

# ── radare2 ──
RUN cd ~/tools && \
    git clone --depth=1 https://github.com/radareorg/radare2 && \
    cd radare2 && ./sys/install.sh && \
    rm -rf ~/tools/radare2
ENV PATH="/usr/local/bin:$PATH"

# ── Android SDK platform-tools (ADB) ──
# 使用腾讯镜像
RUN mkdir -p ${ANDROID_HOME} && \
    cd ${ANDROID_HOME} && \
    wget -q "https://mirrors.cloud.tencent.com/AndroidSDK/platform-tools_r37.0.0-linux.zip" && \
    unzip platform-tools_r37.0.0-linux.zip && rm platform-tools_r37.0.0-linux.zip
ENV PATH="${ANDROID_HOME}/platform-tools:$PATH"

# ── Android SDK cmdline-tools (aapt等) ──
ARG ANDROID_CMDLINE_TOOLS_URL=https://dl.google.com/android/repository/commandlinetools-linux-11076708_latest.zip
RUN wget -q "$ANDROID_CMDLINE_TOOLS_URL" -O /tmp/cmdline-tools.zip && \
    mkdir -p ${ANDROID_HOME}/cmdline-tools && \
    cd ${ANDROID_HOME}/cmdline-tools && \
    unzip /tmp/cmdline-tools.zip && mv cmdline-tools latest && rm /tmp/cmdline-tools.zip
ENV PATH="${ANDROID_HOME}/cmdline-tools/latest/bin:$PATH"

# ── 辅助脚本 ──
COPY scripts/bin/*.sh /usr/local/bin/
RUN sudo chmod +x /usr/local/bin/*.sh

# ── Ghidra headless scripts ──
COPY scripts/ghidra-scripts/*.java ~/tools/ghidra-scripts/
RUN chmod +x ~/tools/ghidra-scripts/*.java

# ── Frida hook templates ──
COPY scripts/frida-hooks/*.js /home/reverser/tools/frida-hooks/
COPY scripts/frida-hooks/frida-auto.sh /home/reverser/tools/bin/
RUN sudo chmod +x /home/reverser/tools/bin/frida-auto.sh && \
    sudo ln -sf /home/reverser/tools/bin/frida-auto.sh /usr/local/bin/frida-auto

# ── device-bridge.sh (容器内备用，但主要在宿主机运行) ──
# 容器内可以用 socat 做二级转发，但通常不需要
COPY scripts/bin/device-bridge.sh /home/reverser/tools/bin/
RUN sudo chmod +x /home/reverser/tools/bin/device-bridge.sh

# ── 知识库 ──
RUN mkdir -p ~/knowledges && cd ~/knowledges && \
    git clone --depth=1 https://github.com/AsteroidZone/Android-RE-reference.git 2>/dev/null || true && \
    git clone --depth=1 https://github.com/AsteroidZone/AndroidSecurityStudy.git 2>/dev/null || true && \
    git clone --depth=1 https://github.com/AsteroidZone/frida-all-in-one.git 2>/dev/null || true

# ── AI Agent CLI (pi only — Cairn uses pi adapter; npm registry = taobao mirror) ──
RUN sudo npm config set registry https://registry.npmmirror.com && \
    sudo npm install -g @mariozechner/pi-coding-agent@0.73.0

# ── Kimi Code CLI (独立二进制官方安装脚本;容器 Node 20 不满足 npm 版要求的 >=22.19) ──
# 装到 /opt 并放开读权限:容器以 reverser 用户运行,/root 不可读
RUN curl -fsSL https://code.kimi.com/kimi-code/install.sh | sudo bash && \
    sudo mkdir -p /opt/kimi-code && sudo cp -a /root/.kimi-code/bin /opt/kimi-code/bin && \
    sudo chmod -R a+rX /opt/kimi-code && \
    sudo ln -sf /opt/kimi-code/bin/kimi /usr/local/bin/kimi && \
    /usr/local/bin/kimi --version

# ── r0re 黑板 skill (kimi 经 --skills-dir /opt/r0re/skills 加载) ──
COPY skills/r0re-blackboard /opt/r0re/skills/r0re-blackboard

# ── 工作目录 ──
RUN mkdir -p ~/workspace/analysis
WORKDIR /home/reverser/workspace

# ── 环境变量 (帮助容器检测自身环境) ──
ENV IN_DOCKER=true
ENV HOST_GATEWAY=host.docker.internal
ENV ADB_SERVER_SOCKET=tcp:host.docker.internal:5037
ENV FRIDA_SERVER_PORT=14725
ENV FRIDA_HOST=host.docker.internal:14725
