# syntax=docker/dockerfile:1

FROM --platform=linux/amd64 ubuntu:jammy as openwrt-build-stage
ARG DEBIAN_FRONTEND=noninteractive
ARG USERNAME=sea
ARG PASSWORD=123456
ARG WORKSPACE=/home/${USERNAME}/ws

# Create a new user
RUN useradd -m -s /bin/bash ${USERNAME}

# Set the user's password
RUN echo "root:${PASSWORD}"        | chpasswd
RUN echo "${USERNAME}:${PASSWORD}" | chpasswd

# Add the new user to the sudo group
RUN usermod -aG sudo ${USERNAME}

# setup mirror sources
RUN <<EOF

mv /etc/apt/sources.list /etc/apt/sources.list.bak

cat << EOT > /etc/apt/sources.list
# 默认注释了源码镜像以提高 apt update 速度，如有需要可自行取消注释
deb http://mirrors.tuna.tsinghua.edu.cn/ubuntu/ jammy main restricted universe multiverse
# deb-src http://mirrors.tuna.tsinghua.edu.cn/ubuntu/ jammy main restricted universe multiverse
deb http://mirrors.tuna.tsinghua.edu.cn/ubuntu/ jammy-updates main restricted universe multiverse
# deb-src http://mirrors.tuna.tsinghua.edu.cn/ubuntu/ jammy-updates main restricted universe multiverse
deb http://mirrors.tuna.tsinghua.edu.cn/ubuntu/ jammy-backports main restricted universe multiverse
# deb-src http://mirrors.tuna.tsinghua.edu.cn/ubuntu/ jammy-backports main restricted universe multiverse

deb http://security.ubuntu.com/ubuntu/ jammy-security main restricted universe multiverse
# deb-src http://security.ubuntu.com/ubuntu/ jammy-security main restricted universe multiverse

# 预发布软件源，不建议启用
# deb http://mirrors.tuna.tsinghua.edu.cn/ubuntu/ jammy-proposed main restricted universe multiverse
# # deb-src http://mirrors.tuna.tsinghua.edu.cn/ubuntu/ jammy-proposed main restricted universe multiverse
EOT

apt update;
apt install -q -y --no-install-recommends \
    vim                                   \
    zip                                   \
    unzip                                 \
    sudo                                  \
    file                                  \
    make                                  \
    wget                                  \
    git                                   \
    cmake                                 \
    tzdata                                \
    net-tools                             \
    locales                               \
    ssh                                   \
    iputils-ping                          \
    bash-completion                       \
    python3-pip                           \
    python3-distutils                     \
    build-essential                       \
    clang                                 \
    flex                                  \
    bison                                 \
    g++                                   \
    gawk                                  \
    gcc-multilib                          \
    g++-multilib                          \
    gettext                               \
    libncurses-dev                        \
    libssl-dev                            \
    rsync                                 \
    zlib1g-dev                            \


echo 'Asia/Shanghai' > /etc/timezone

ln -snf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime

pip config set global.index-url https://pypi.tuna.tsinghua.edu.cn/simple

EOF

USER ${USERNAME}
WORKDIR ${WORKSPACE}
COPY --chown=${USERNAME}:${USERNAME} openwrt ./openwrt
#RUN <<EOF
## Download the sources
#git clone https://git.openwrt.org/openwrt/openwrt.git
#EOF

WORKDIR ${WORKSPACE}/openwrt
RUN <<EOF
<<COMMENT
# Update the sources
git pull

# Select a specific code revision
# OpenWrt 22.03.5 r20134-5f15225c1e
# LuCI openwrt-22.03 branch git-23.119.80898-65ef406
git checkout v22.03.5

# Update the feeds
./scripts/feeds update -a
./scripts/feeds install -a

# Configure the firmware image
make menuconfig

# Optional: configure the kernel (usually not required)
# Don't, unless have a strong reason to
make -j$(nproc) kernel_menuconfig

# Build the firmware image
make -j$(nproc) defconfig download clean world
COMMENT

EOF

USER root
RUN <<EOF

cat << EOT > /startup.sh
#!/bin/bash
set -e
while [ 1 ]; do
    date -R;
    sleep 1;
done
exec "\$@"
EOT

echo "" >> /root/.bashrc
echo "if [ -f /etc/bash_completion ] && ! shopt -oq posix; then" >> /root/.bashrc
echo "    . /etc/bash_completion" >> /root/.bashrc
echo "fi" >> /root/.bashrc
echo "" >> /root/.bashrc

EOF

USER ${USERNAME}
# exec
ENTRYPOINT ["/bin/bash", "/startup.sh"]
