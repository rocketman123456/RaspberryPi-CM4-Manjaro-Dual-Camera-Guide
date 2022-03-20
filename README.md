# RaspberryPi-CM4-Manjaro-Dual-Camera-Guide
Raspberry Pi CM4; Manjaro; Dual Camera Guide

# 目的
为了使用微雪的CM4双目摄像头设备，结合ROS2进行带网络的机器人相关开发，折腾了一番<br>
尝试了很多系统与配置方法，终于在manjaro系统上成功使用双目摄像头，特此记录

# 1. 烧录系统
准备一张16G以上的tf卡，使用官方的[烧录软件](https://www.raspberrypi.com/documentation/computers/getting-started.html#using-raspberry-pi-imager)进行系统烧录，选择Manjaro系统。
烧录完成后，需要进行配置才能用鼠标键盘操作开发板

# (Optional)从NVME SSD启动系统
可以根据[这篇教程](https://dphacks.com/2021/11/21/how-to-boot-a-pi-cm4-from-nvme-ssd/)进行操作，即可改变默认的启动顺序

# 2. 修改boot.txt启动文件
树莓派的系统启动将从/boot/config.txt文件中读取配置，具体的内容可见[这篇官方介绍](https://www.raspberrypi.com/documentation/computers/config_txt.html)
在此贴出我的配置文件

```
# See /boot/overlays/README for all available options

gpu_mem=256
initramfs initramfs-linux.img followkernel
kernel=kernel8.img
arm_64bit=1
disable_overscan=1

#enable sound
#dtparam=audio=on
#hdmi_drive=2

#enable vc4
dtoverlay=vc4-fkms-v3d
max_framebuffers=2
#disable_splash=1

# Enable USB
dtoverlay=dwc2,dr_mode=host

# Set External Antenna
dtparam=ant1

# Overclock
#over_voltage=6
#arm_freq=2000
#gpu_freq=750

# Set Camera
start_x=1
```

# 3. 加入dt-blob.bin
该文件是特殊的Linux Device Tree，该文件在这里是树莓派GPU对GPIO口的配置，进入系统后，该部分的操作为：<br>
```
sudo wget https://datasheets.raspberrypi.com/cmio/dt-blob-dualcam.bin -O /boot/dt-blob.bin
```
从[这篇文章中找到](https://www.raspberrypi.com/documentation/computers/compute-module.html#quickstart-guide)

# 4. 进入系统，进行测试
最后在系统中安装cheese，即可对摄像头进行测试，从系统库中安装完opencv和opencv的python接口后，也进行了相应的测试，摄像头可以正常工作。
在cheese中采集的图像是正常的，但是在opencv中采集的图像有色调之类的问题，需要通过正确配置摄像头属性才能解决问题。
同时编写了简单的opencv测试程序，双目可以正常操作。

# 5. OpenCV Python测试程序
```
import cv2
cap0 = cv2.VideoCapture(0)
cap0.set(cv2.CAP_PROP_FRAME_WIDTH, 640)
cap0.set(cv2.CAP_PROP_FRAME_HEIGHT, 480)
cap1 = cv2.VideoCapture(1)
cap1.set(cv2.CAP_PROP_FRAME_WIDTH, 640)
cap1.set(cv2.CAP_PROP_FRAME_HEIGHT, 480)
while True:
    ret, frame0 = cap0.read()
    ret, frame1 = cap1.read()
    cv2.imshow("cap0", frame0)
    cv2.imshow("cap1", frame1)
    if cv2.waitKey(1) & 0xFF == ord('q'):
        break
cap0.release()
cap1.release()
cv2.destroyAllWindows()
```

# 5.1 OpenCV C++ 测试程序

```
#include <stdio.h>
#include <opencv2/opencv.hpp>

using namespace cv;

int main(int argc, char** argv ) {
    VideoCapture cap0(0, CAP_V4L);
    if (!cap0.isOpened()) {
        printf("cannot open cam0\n");
        return 1;
    }
    cap0.set(CAP_PROP_FRAME_WIDTH, 640);
    cap0.set(CAP_PROP_FRAME_HEIGHT, 480);

    VideoCapture cap1(1, CV_CAP_V4L);
    if (!cap1.isOpened()) {
        printf("cannot open cam1\n");
        return 1;
    }
    cap1.set(CAP_PROP_FRAME_WIDTH, 640);
    cap1.set(CAP_PROP_FRAME_HEIGHT, 480);

    bool ret0, ret1;
    Mat frame0, frame1;

    for(int i = 0; i < 10; ++i) {
        ret0 = cap0.read(frame0);
        if(!ret0) {
            printf("read cam0 error\n");
            break;
        }
        ret1 = cap1.read(frame1);
        if(!ret1) {
            printf("read cam1 error\n");
            break;
        }
        printf("Frame %d\n", i);
    }
    cap0.release();
    cap1.release();

    return 0;
```

# 6. 检测摄像头分辨率
[这篇博客](https://www.learnpythonwithrune.org/find-all-possible-webcam-resolutions-with-opencv-in-python/)
```
import pandas as pd
import cv2


url = "https://en.wikipedia.org/wiki/List_of_common_resolutions"
table = pd.read_html(url)[0]
table.columns = table.columns.droplevel()

cap = cv2.VideoCapture(0)
resolutions = {}

for index, row in table[["W", "H"]].iterrows():
    cap.set(cv2.CAP_PROP_FRAME_WIDTH, row["W"])
    cap.set(cv2.CAP_PROP_FRAME_HEIGHT, row["H"])
    width = cap.get(cv2.CAP_PROP_FRAME_WIDTH)
    height = cap.get(cv2.CAP_PROP_FRAME_HEIGHT)
    resolutions[str(width)+"x"+str(height)] = "OK"

print(resolutions)
```
