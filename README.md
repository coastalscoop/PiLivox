# PiLivox
The overview below is how to take a totally clean RPI 5 and configure it to run a Livox.

## Setting up the Pi
- In terms of physical modifications to the Pi, it is a good idea to install a cooling fan, and also add an RTC battery.
- Download Raspberry Pi Imager on another machine (i.e. your desktop) and use it to write the operating system for the Pi to a MicroSD card. I chose to write the latest Raspberry Pi Debian OS to a 64gb MicroSD card.
- Insert the MicroSD into the Pi and boot it up for the first time.
- Install some basice packages:
  ```
  sudo apt-get update
  sudo apt-get install
  sudo apt install cmake
  ```

## Install the Livox Software Development Kit
  ```
  git clone https://github.com/Livox-SDK/Livox-SDK.git
  ```

At this point, we need to edit some of the files and it appears there is an omission in what is downloaded from Livox. Go to **Livox-SDK/sdk_core/src/base** and edit both the *thread_base.cpp* and *thread_base.h* files to include the text 
```
#include <memory>
```
in the opening lines.

Now run the compilation and install code:
```
cd Livox-SDK
cd build && cmake ..
make
sudo make install
```

## Connect the Livox

Now provide power to the Livox unit, and connect it to the Rasberry Pi using a Cat 6 ethernet cable between the converter and the ethernet port of the RPI5.

The Livox units typically come with an IP address of 192.168.1.1XX (where XX stands for the last two numbers in the serial number). In order to communicate with the Livox, you need to set the Raspberry Pi's IP address to be on the same subnet. To do this, you can click on **Network Connections** in the top right corner, go to **Advanced Options**, and then click **Edit Connections**. Double click on **Wired Connection** and on the **IPv4 Settings** tab change the **Method** to *Static* and add a new connection, making the address **192.168.1.50**, the Netmask **24**, and the Gateway **192.168.1.0**.

Save these changes and close.

Open up a command terminal and navigate back to the livox sample folder:

```
cd Livox-SDK/build/sample/lidar_lvx_file/
```
and now try and run the sample script:
```
./lidar_lvx_sample
```

If all is working correctly, you will see some preliminary connection information displayed, followed by a number of lines that say:
``` 
Finish save XX frame to lvx file.
```

This is good news! It means a scan is being taken and saved! However, if you see:

```
LocalIp and DeviceIp are not in same subnet
```

that is a common error and just means that the IP address is not yet configured properly so revisit that step and ensure both the RPI and the Livox have IP addresses on the same subnet.

Once that code runs successfuly, it will store a .lvx file in the current directory. The lvx files contain the scan data and the associated timestamps. 

## Automating the process


