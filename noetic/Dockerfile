# syntax=docker/dockerfile:1

# This is an auto generated Dockerfile for ros:desktop-full
# generated from docker_images/create_ros_image.Dockerfile.em
FROM osrf/ros:noetic-desktop-focal

# install ros packages
RUN apt-get update && apt-get install -y --no-install-recommends \
    ros-noetic-desktop-full

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
echo "source /opt/ros/noetic/setup.bash" >> /root/.bashrc

EOF

# exec
ENTRYPOINT ["/bin/bash", "/startup.sh"]
