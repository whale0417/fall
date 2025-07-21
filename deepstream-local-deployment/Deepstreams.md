# Setup

Deepstream(Local deployment and operation)：

Through the following steps, you will deploy the deepstream project in your Ubuntu system of wsl2. Specifically, by configuring the environment and running the docker-compose.yml file, mediamtx containers, rabitmq containers, and deepstream containers will be created in Ubuntu.

## Prerequisites

1.Ensure that WSL is installed on your Windows system

2.make sure your [NVIDIA Driver version >= 535.161.08]

https://www.nvidia.com/Download/index.aspx

3.make sure your cuda version >= 12.2 (12.2 recommanded) by using command `nvcc -V`

## Start:

### install Ubuntu wsl：

```shell
 wsl --install Ubuntu-22.04 # install Ubuntu-22.04 on windows
 wsl -d Ubuntu-22.04 # activate Ubuntu-22.04
 nvidia-smi # Verify Driver Installation from within WSL environment
```

In Ubuntu WSL, if your CUDA version is<12.12，install it using the following command:

```shell
 wget https://developer.download.nvidia.com/compute/cuda/repos/wsl-ubuntu/x86_64/cuda-wsl-ubuntu.pin
 sudo mv cuda-wsl-ubuntu.pin /etc/apt/preferences.d/cuda-repository-pin-600
 wget https://developer.download.nvidia.com/compute/cuda/12.2.2/local_installers/cuda-repo-wsl-ubuntu-12-2-local_12.2.2-1_amd64.deb
 sudo dpkg -i cuda-repo-wsl-ubuntu-12-2-local_12.2.2-1_amd64.deb
 sudo cp /var/cuda-repo-wsl-ubuntu-12-2-local/cuda-*-keyring.gpg /usr/share/keyrings/
 sudo apt-get update
 sudo apt-get -y install cuda
```

### install nvidia tools

#### install CUDA Toolkit

if your WSL already support GPU acceleration, there is no need to install the Nvidia CUDA toolkit separately, check if CUDA toolkit is available

```shell
 nvcc --version
```

if not , visit the official download page, select the appropriate installation package type for your system, and follow the official instructions to install.

https://developer.nvidia.com/cuda-toolkit

after installation, check if CUDA toolkit is available:

```shell
 nvcc --version
```

#### install NVIDIA Container Toolkit

add NVIDIA container toolkit source:

```shell
 distribution=$(. /etc/os-release;echo $ID$VERSION_ID)
 curl -s -L https://nvidia.github.io/libnvidia-container/gpgkey | sudo apt-key add -
 curl -s -L https://nvidia.github.io/libnvidia-container/$distribution/libnvidia-container.list | sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
 sudo apt-get update
```

install NVIDIA Container Toolkit:

```shell
 sudo apt-get install -y nvidia-container-toolkit
```

restart Docker service to apply changes

##### attention: if you still have doubts about the above steps, you can also refer to the official Nvidia tutorial

https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html#installing-with-apt

### install Gstreamer

```shell
 sudo apt-get install libgstreamer1.0-dev libgstreamer-plugins-base1.0-dev libgstreamer-plugins-bad1.0-dev gstreamer1.0-plugins-base gstreamer1.0-plugins-good gstreamer1.0-plugins-bad gstreamer1.0-plugins-ugly gstreamer1.0-libav gstreamer1.0-tools gstreamer1.0-x gstreamer1.0-alsa
 gstreamer1.0-gl gstreamer1.0-gtk3 gstreamer1.0-qt5 gstreamer1.0-pulseaudio # install
 gst-inspect-1.0 --version # verify
```

### install docker

if Docker Desktop is installed on a Windows system, the following commands do not need to be executed in WSL:

```shell
 sudo apt-get install -y apt-transport-https ca-certificates curl gnupg-agent software-properties-common
 curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
 sudo apt-key fingerprint 0EBFCD88
 sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
 sudo apt-get update
 sudo apt-get install -y docker-ce docker-ce-cli containerd.io --fix-missing
```

### update docker-desktop config

open docker-desktop --> settings --> Docker Engine, add these and restart docker

```json
  "runtimes": {
      "nvidia": {
        "path": "nvidia-container-runtime",
        "runtimeArgs": []
      }
    },
    "default-runtime": "nvidia"
```

### setup docker container

copy the Deepstreams folder to the Ubuntu system, and then enter the file directory.

```shell
 docker-compose up -d
 docker exec -i -t deepstream /bin/bash # connect to this docker, the deepstream here can be replaced with the name of the container you want to link to.
```

### clone and run deepstream-yolo

when in deepstream container

```shell
 sudo apt-get update
 git clone https://github.com/marcoslucianops/DeepStream-Yolo.git
```

run the example of the deepstream-yolo project according to the tutorial of the official deepstream-yolo repository (this step is necessary, and it will configure the basic environment for deepstream-yolo)

https://github.com/marcoslucianops/DeepStream-Yolo

in the official deepstream-yolo repository tutorial, there are several points to note.

1.in the second step: "Download the cfg and weights files from Darknet repo to the DeepStream Yolo folder". The .cfg files are already located in the deepstream under the Deepstreams folder. Just copy them to the DeepStream Yolo folder.The .weights file is too large. You need to contact the administrator to obtain it.

2.in the third step: "Set the CUDA_VER according to your DeepStream version". If you follow the document environment completely, set export CUDA_VER=12.2  #If not, you should combine your specific environment

3.if the device running deepstream does not have a display to display the detected content, you need to modify the following configuration items in the deepstream_app_comfig.txt. The key is that when type=2, its corresponding enable=0
```shell
 ...
 [sink0]
 enable=0
 type=2 # show on screen (graphic system only)
 sync=0
 gpu-id=0
 nvbuf-memory-type=0
 ...
```

4.you may use the following command to copy user files.
```shell
 docker cp ...\fall.onnx deepstream:/.../DeepStream-Yolo #the function of this command is to copy files under a path in the Windows system to a container named deepstream in the wsl
```

### run your own deepstream-yolo project

the .pt format model needs to be converted to .onnx format. The specific steps for conversion are as follows:

https://github.com/marcoslucianops/DeepStream-Yolo/blob/master/docs/YOLOv8.md

modify the following configuration items in `deepstream_app_config.txt`:

```shell
 [source0]
 ...
 uri= #The address of the video you want to use for detection. e.g.when use a camera, change here to rtsp, replace with your camera ip uri=rtsp://[username]:[password]@[rtspIp]:[rtspPort].
 ...
 [primary-gie]
 ...
 config-file= #Your modified configuration file(e.g. config_infer_primary_yoloV8.txt)
 ...
```

in your own config-file (e.g. config_infer_primary_yoloV8. txt), modify the following configuration items:

```shell
 [property]
 ...
 onnx-file=fall.onnx # fall.onnx file is already located in the deepstream under the Deepstreams folder. Just copy it to the DeepStream Yolo folder. You can also copy your own .onnx file to use.
 model-engine-file = model_b1_gpu0_fp32.engine #This will be generated automatically. If you change the model, you need to delete the previously generated one
 labelfile-path=fall_labels.txt #fall_labels.txt file is already located in the deepstream under the Deepstreams folder. Just copy it to the DeepStream Yolo folder. You can also copy your own .txt file to use.
 ...
```

### the combination of deepstream and arrtFronted

add the following content to the deepstream_app_config.txt file:

```shell
 [sink1]
 enable=1
 type=4
 codec=1
 enc-type=0
 sync=0
 bitrate=800000
 profile=0
 rtsp-port=8554
 udp-port=5400

 [sink3]
 enable=1
 type=6
 msg-conv-config=config_msgconv.txt #The config_msgconv.txt file is already in the deepstream under the Deepstreams folder. Just copy it to the DeepStream Yolo folder. You can also copy your own config_msgconv.txt file to use
 msg-conv-payload-type=0
 multiple-payloads=1
 msg-conv-msg2p-new-api=1
 msg-broker-proto-lib=/opt/nvidia/deepstream/deepstream-7.0/lib/libnvds_amqp_proto.so
 msg-broker-config=config_amqp.txt #The config_amqp.txt file is already in the deepstream under the Deepstreams folder. Just copy it to the DeepStream Yolo folder. You can also copy your own config_amqp.txt file to use
 new-api=1
```

modify the following code in the project frontend relative path: src\components\Monitor\CameraFeed.tsx:

```shell
const videoSrc = ' http://127.0.0.1:8888/stream1/index.m3u8 ' #your address
```

### the combination of deepstream and arrtBackend

#### modify the following content in the backend .env file

```shell
 DEV_URL= #Your address, eg：localhost
 RABBITMQ_PORT= "5672"
 RABBITMQ_USER= "admin"
 RABBITMQ_PASSWORD= "admin"
```

# other problems

1.Can Deepstream set the IP address for RTSP streaming?

No, currently DeepStream 7.0 does not support setting RTSP streaming IP addresses, but supports modifying RTSP streaming ports. The localhost address released by Deepstream can be changed to another address through NGork technology.

2.How to use Deepstream's built-in tools to detect its RTSP video streams?

By using the underlying plugin GStreamer of the Deepstream framework to detect, the specific commands are as follows:

```shell
 gst-launch-1.0 rtspsrc location=rtsp://localhost:8554/ds-test protocols=tcp ! decodebin ! autovideosink
```

3.How to confirm if RabbitMQ has been successfully integrated into the project?

The simplest way is to run Deepstream and check if the relevant information detected by Deepstream is written in the backend database.

4.What technologies are involved in the project? Or, what is the technical schematic diagram of the project?

![Deepstream](https://github.com/user-attachments/assets/c9a86a93-62a2-441f-856b-8553691e125c)

