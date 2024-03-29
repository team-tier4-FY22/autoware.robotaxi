ARG CUDA_IMAGE_TAG
FROM nvidia/cuda:$CUDA_IMAGE_TAG as devel
SHELL ["/bin/bash", "-o", "pipefail", "-c"]

ARG ROS_DISTRO
ARG CUDNN_VERSION
ARG TENSORRT_VERSION

## Install apt packages
# hadolint ignore=DL3008
RUN apt-get update && DEBIAN_FRONTEND=noninteractive apt-get -y install --no-install-recommends \
  git \
  ssh \
  && apt-get clean \
  && rm -rf /var/lib/apt/lists/*

## Copy files
COPY autoware.repos setup-dev-env.sh ansible-galaxy-requirements.yaml amd64.env arm64.env /autoware/
COPY ansible/ /autoware/ansible/
WORKDIR /autoware
RUN ls /autoware

## Add GitHub to known hosts for private repositories
RUN mkdir -p ~/.ssh \
  && ssh-keyscan github.com >> ~/.ssh/known_hosts

## Set up development environment
RUN --mount=type=ssh \
  ./setup-dev-env.sh -y --no-nvidia universe \
  && pip uninstall -y ansible ansible-core \
  && mkdir src \
  && vcs import src < autoware.repos \
  && rosdep update \
  && DEBIAN_FRONTEND=noninteractive rosdep install -y --ignore-src --from-paths src --rosdistro "$ROS_DISTRO" \
  && apt-get clean \
  && rm -rf /var/lib/apt/lists/*

## Install cuDNN and TensorRT
RUN apt-get update && DEBIAN_FRONTEND=noninteractive apt-get -y install --no-install-recommends \
  libcudnn8="$CUDNN_VERSION" \
  libcudnn8-dev="$CUDNN_VERSION" \
  libnvinfer8="$TENSORRT_VERSION" \
  libnvinfer-dev="$TENSORRT_VERSION" \
  libnvinfer-plugin8="$TENSORRT_VERSION" \
  libnvinfer-plugin-dev="$TENSORRT_VERSION" \
  libnvonnxparsers8="$TENSORRT_VERSION" \
  libnvonnxparsers-dev="$TENSORRT_VERSION" \
  libnvparsers8="$TENSORRT_VERSION" \
  libnvparsers-dev="$TENSORRT_VERSION" \
  && apt-get clean \
  && rm -rf /var/lib/apt/lists/*

## Clean up unnecessary files
RUN rm -rf \
  "$HOME"/.cache \
  /etc/apt/sources.list.d/cuda.list \
  /etc/apt/sources.list.d/docker.list \
  /etc/apt/sources.list.d/nvidia-docker.list

## Create entrypoint
# hadolint ignore=DL3059
RUN echo "source /opt/ros/${ROS_DISTRO}/setup.bash" > /etc/bash.bashrc
CMD ["/bin/bash"]

FROM devel as prebuilt
SHELL ["/bin/bash", "-o", "pipefail", "-c"]

## Build and change permission for runtime data conversion
RUN source /opt/ros/"$ROS_DISTRO"/setup.bash \
  && colcon build --cmake-args -DCMAKE_BUILD_TYPE=Release \
  && find /autoware/install -type d -exec chmod 777 {} \;

## Create entrypoint
RUN echo "source /autoware/install/setup.bash" > /etc/bash.bashrc
CMD ["/bin/bash"]
