# Setup

Deepstream(Local deployment and operation)：

By following these steps, you will create an Ubuntu system in WSL and deploy DeepStream in a Docker container under the Ubuntu system，and the operating instructions for Deepstream.

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

### setup a docker container
```shell
 docker-compose up -d
 docker exec -i -t deepstream /bin/bash # connect to this docker
```

### clone deepstream-yolo
when in docker
```shell
 sudo apt-get update
 git clone https://github.com/marcoslucianops/DeepStream-Yolo.git
```

### running Deepstream official example
execute commands in WSL to enter a Docker container called `deepstream`
```shell
 docker exec -it deepstream /bin/bash
```
refer to the following website for a simple test run

https://github.com/marcoslucianops/DeepStream-Yolo

### run your own deepstream project
the .pt format model needs to be converted to .onnx format. The specific steps for conversion are as follows:

https://github.com/marcoslucianops/DeepStream-Yolo/blob/master/docs/YOLOv8.md

modify the following configuration items in `deepstream_app_comfig.txt`:
```shell
 [source0]
 ...
 uri= #The address of the video you want to use for detection
 ...
 [primary-gie]
 ...
 config-file= #Your modified configuration file(e.g. config.inpfer_primary_yoloV8.txt)
 ...
```
in your own config-file (e.g. config. inpfer_primary_yoloV8. txt), modify the following configuration items:
```shell
 [property]
 ...
 onnx-file= #Your .onnx format model path
 model-engine-file = model_b1_gpu0_fp32.engine #This will be generated automatically. If you change the model, you need to delete the previously generated one
 ...
```


### the combination of deepstream and arrtFronted

add the following content to the deepstream_app_comfig.txt file:
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
 enable=0
 type=6
 msg-conv-config=config_msgconv.txt #This file needs to be created by oneself
 msg-conv-payload-type=0
 multiple-payloads=1
 msg-conv-msg2p-new-api=1
 msg-broker-proto-lib=/opt/nvidia/deepstream/deepstream-7.0/lib/libnvds_amqp_proto.so
 msg-broker-config=config_amqp.txt #This file needs to be created by oneself
 new-api=1
```

if the network mode of your deepstream container is host, make sure that Docker Desktop has enabled host mode and check the following settings in Docker Desktop
```shell
 Setting → Resources → Network → Enable host networking
```

modify configuration items in the mediamtx configuration file
```shell
 ...
 rtspAddress: :8554 → rtspAddress: 0.0.0.0:8554
 ...
 source: #If Deepstream uses host mode, please write the address of the WSL here， for example: rtsp://172.21.81.242:8554/ds-test
 ...
```

### the combination of deepstream and arrtBackend
#### make sure you have installed rebbitmqq correctly. If not, follow the steps below to install RabbitMQ in Docker

execute the following command in WSI to download and run RabbitMQ
```shell
 docker run -d --name rabbitmq -p 5672:5672 -p 15672:15672 rabbitmq:management
```
visit RabbitMQ management interface to verify successful installation
```shell
 http://localhost:15672 #the default username and password are both: guest
```

check the network mode of RabbitMQ in your Docker container to ensure proper communication between RabbitMQ and arrtBackend, as well as between RabbitMQ and Deepstream

#### modify the following content in the backend. env file
```shell
 DEV_URL= #Your address
 RABBITMQ_PORT= #Your port
 RABBITMQ_USER= #Your user
 RABBITMQ_PASSWORD= #Your password
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
