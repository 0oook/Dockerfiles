# syntax=docker/dockerfile:1

FROM --platform=linux/amd64 ubuntu:jammy as openwrt-build-stage
ARG DEBIAN_FRONTEND=noninteractive

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
    curl                                  \
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
    software-properties-common            \
    lsb-release                           \
    gnupg

echo 'Asia/Shanghai' > /etc/timezone

ln -snf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime

pip config set global.index-url https://pypi.tuna.tsinghua.edu.cn/simple

EOF

# Setup ros humble resoures
RUN <<EOF

# set locale
locale
locale-gen en_US en_US.UTF-8
update-locale LC_ALL=en_US.UTF-8 LANG=en_US.UTF-8
export LANG=en_US.UTF-8
locale

# setup sources
add-apt-repository universe
curl -sSL https://raw.githubusercontent.com/ros/rosdistro/master/ros.key -o /usr/share/keyrings/ros-archive-keyring.gpg

#echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/ros-archive-keyring.gpg] http://packages.ros.org/ros2/ubuntu $(. /etc/os-release && echo $UBUNTU_CODENAME) main" | tee /etc/apt/sources.list.d/ros2.list > /dev/null
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/ros-archive-keyring.gpg] http://mirrors.tuna.tsinghua.edu.cn/ros2/ubuntu jammy main" | tee /etc/apt/sources.list.d/ros2.list > /dev/null

apt update

EOF

# Install ros packages
RUN <<EOF

apt install -q -y --no-install-recommends \
    ros-humble-ros-base                   \
    ros-humble-rviz2                      \
    ros-humble-navigation2                \
    ros-humble-nav2-bringup               \
    ros-humble-turtlebot3-gazebo

EOF

# Copy gazebo models files
ADD osrf/gazebo_models /root/.gazebo/models

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
echo "source /opt/ros/humble/setup.bash" >> /root/.bashrc

EOF

# exec
ENTRYPOINT ["/bin/bash", "/startup.sh"]
# docker run -it -v /tmp/.x11-unix:/tmp/.x11-unix --env DISPLAY=$DISPLAY --name gazebo --privileged --net host gazebo_image:latest /bin/bash
# ros2 launch ros_gz_sim_demos air_pressure.launch.py
# git clone https://github.com/osrf/gazebo_models.git

# source /opt/ros/humble/setup.bash
# export TURTLEBOT3_MODEL=waffle
# export GAZEBO_MODEL_PATH=$GAZEBO_MODEL_PATH:/opt/ros/humble/share/turtlebot3_gazebo/models

# ros2 launch nav2_bringup tb3_simulation_launch.py headless:=False
