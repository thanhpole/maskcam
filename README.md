# MaskCam
MaskCam is a prototype of a Jetson Nano-based smart camera system that measures crowd face mask usage in real-time, with all AI computation performed at the edge. MaskCam detects and tracks people in its field of view and determines whether they are wearing a mask via an object detection, tracking, and voting algorithm. It uploads statistics (not videos) to the cloud, where a web GUI can be used to monitor the face mask compliance in the field of view. It saves interesting video snippets to local disk (e.g., a sudden influx of lots of people not wearing masks) and can optionally stream video via RTSP.

MaskCam can be run on a Jetson Nano Developer Kit, or on a Jetson Nano SOM with the ConnectTech Photon carrier board.  It is designed to use the Raspberry Pi High Quality Camera but will also work with pretty much any USB webcam.

The on-device software stack is mostly written in Python and runs under JetPack 4.4.1 or 4.5, Edge AI processing is handled by Nvidia’s DeepStream video analytics framework and YoloV4 Tiny, and Tryolab's Norfair tracker.  MaskCam reports statistics to and receives commands from the cloud using MQTT and a web-based GUI.
The software is containerized and can be easily installed on a Jetson Nano DevKit using docker with just a couple of commands.  For production purposes MaskCam can run under BalenaOS, which makes it easy to manage and deploy multiple devices.

We urge you to try it out! It’s easy to install on a Jetson Nano Dev Kit and requires only a web cam. (The cloud-based statistics server and web GUI are optional, but are also dockerized and easy to instal on any reasonable Linux system, or on a low-cost AWS ec2 instance.)  See below for installation instructions.

MaskCam was developed by Berkeley Design Technology, Inc. (BDTI) and Tryolabs S.A., with development funded by Nvidia. MaskCam is offered under the MIT License. For more information about MaskCam, please see the forthcoming white paper from BDTI.

## Running MaskCam from a Container on a Jetson Nano Developer Kit
The easiest and fastest way to get MaskCam running on your Jetson Nano Dev Kit is using our pre-built containers.  Make sure that you have JetPack 4.4.1 or 4.5 iinstalled on your Nano DevKit and then run:
```
docker pull maskcam/maskcam-beta
docker run --runtime nvidia --privileged --rm -it -p 1883:1883 -p 8080:8080 -p 8554:8554 maskcam/maskcam-beta
```
Note that the container is quite large, so it will take maybe 10 minutes to download and inistall.

## Building MaskCam from Source on Jetson Nano Developer Kit
1. Make sure these packages are installed at system level:
```
sudo apt install python3-opencv python3-libnvinfer
```

2. Clone this repo:
```
git clone git@github.com:tryolabs/bdti-jetson.git
```

3. Install the requirements listed on `requirements.in` (currently not freezed) **without dependencies** to avoid installing python-opencv (which is already installed system-level):
```
pip3 install --no-deps -r requirements.in
```

4. Install Nvidia DeepStream:
Aside from the system requirements of th previous step, you also need to install
[DeepStream 5.0](https://docs.nvidia.com/metropolis/deepstream/dev-guide/text/DS_Quickstart.html#jetson-setup) 
(no need to install Kafka protocol adaptor)
and also make sure to install the corresponding **python bindings** for GStreamer
[gst-python](https://docs.nvidia.com/metropolis/deepstream/dev-guide/text/DS_Python_Sample_Apps.html#python-bindings),
and for DeepStream [pyds](https://docs.nvidia.com/metropolis/deepstream/dev-guide/text/DS_Python_Sample_Apps.html#metadata-access).

5. Compile YOLOv4 plugin for DeepStream:
After installing DeepStream, compile the YOLOv4 plugin for DeepStream:
```
cd <this repo path>/deepstream_plugin_yolov4
export CUDA_VER=10.2
make
```
If all went well, you should see a library `libnvdsinfer_custom_impl_Yolo.so` in that directory.

6. Download TensorRT engine file from [here](https://drive.google.com/file/d/1Qb6f2VNXE15EgIi6roebgSo8XZuAPQxi/view?usp=sharing) and save it as `yolo/facemask_y4tiny_1024_608_fp16.trt`.

7. Now you should be ready to run. By default, the device `/dev/video0` will be used, but other devices can be set as first argument:
```bash
# Use default input camera /dev/video0
python3 maskcam_run.py

# Equivalent as above:
python3 maskcam_run.py v4l2:///dev/video0

# Process an mp4 file instead (no network functions, MQTT and static file server disabled)
python3 maskcam_run.py file:///path/to/video.mp4

# Read from Raspi2 camera using device-id
python3 maskcam_run.py argus:///0
```

8. If you want to add an MQTT broker, set the following environment variables:
```bash
export MQTT_BROKER_IP=<server ip>
export MQTT_DEVICE_NAME=<unique identifier for this device>

python3 maskcam_run.py
```

### Sending MQTT messages
If you just want to test MQTT messages and be able to send commands to the device, you might run
the MQTT broker in your local machine and set your IP as the `MQTT_BROKER_IP`.

The MQTT broker is called `mosquitto` and can be run locally if you have `docker-compose` in your computer:
```
cd server
cp database.env.template database.env
cp frontend.env.template frontend.env
cp backend.env.template backend.env

docker-compose up mosquitto
```

Then you can run the MQTT commander script for debugging on your local computer or from the Jetson device itself:
```
export MQTT_BROKER_IP=<server ip (local or remote)>
export MQTT_DEVICE_NAME=<device to command>
python3 -m maskcam.mqtt_commander
```

Note that you'll need to set the IP of the MQTT Broker as `127.0.0.1` if you're running the commander in the
same computer where you're running docker-compose, or set it to your computer's network address if running
on the device (same `MQTT_BROKER_IP` that you need to run `maskcam_run.py`).

### Running MaskCam standalone services
When `maskcam_run.py` is run, it actually runs several processes which can be run individually:
```bash
# This process runs DeepStream and generates UDP video packages to be used in other processes
python3 -m maskcam.maskcam_inference
```

In another terminal, run simultaneously (this will start saving a file from UDP packages, until Ctrl+C is pressed):
```bash
python3 -m maskcam.maskcam_filesave
```

To visualize the stream remotely via RTSP, start the streaming service:
```bash
python3 -m maskcam.maskcam_streaming
```

The same concept applies to the static file server `maskcam_fileserver`.


## Running TensorRT engine on images
After following the steps to run `maskcam` (except that you don't need DeepStream for this part),
you might also want to test the object detector on a folder with images:
```
cd yolo/
python3 run_yolo_images.py path/to/input/folder path/to/output/folder
```

## Setting up the web server
Under the `server/` folder, it can be found a whole complete implementation of a server using docker-compose,
which contains a mosquitto broker, backend API, database, and streamlit frontend.
It can be deployed to any local or remote machine (tested on linux and OSX).

Note that you can also run only the MQTT broker service to test the device (see section above).

To create all the services for the web application, create the `.env` files using the default templates:
```
cd server
cp database.env.template database.env
cp frontend.env.template frontend.env
cp backend.env.template backend.env

docker-compose build
docker-compose up
```

## Running on Jetson Nano with Photon carrier board
Please see the setup instructions at [docs/Photon-Nano-Setup.md](docs/Photon-Nano-Setup.md) for how to set up and run MaskCam on the Photon Nano.

## Convert weights generated using the original darknet implementation to TRT
 1. Clone the pytorch implementation of YOLOv4:
```
git clone git@github.com:Tianxiaomo/pytorch-YOLOv4.git
```
 2. Convert the Darknet model to ONNX using the script in `tool/darknet2onnx.py`, e.g:
```
PYTHONPATH='pytorch-YOLOv4:$PYTHONPATH' python3 pytorch-YOLOv4/tool/darknet2onnx.py yolo/facemask-yolov4-tiny.cfg yolo/facemask-yolov4-tiny_best.weights <optional batch size>
```
 3. Convert the ONNX model to TRT (on the Jetson Nano, `trtexec` can be found under `/usr/src/tensorrt/bin/trtexec`):
```
/usr/src/tensorrt/bin/trtexec --fp16 --onnx=../yolo/yolov4_1_3_608_608_static.onnx --explicitBatch --saveEngine=tensorrt_fp16.trt
```
