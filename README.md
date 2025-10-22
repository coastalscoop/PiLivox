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
- Enable charging of the RTC battery when mains are connected
   ```
    sudo nano /boot/firmware/config.txt
   ```
- Check the battery voltage (and periodically check to see that it is charging):
   ```
  vcgencmd pmic_read_adc BATT_V
    ```
## Download the Livox Software Development Kit
  ```
  git clone https://github.com/Livox-SDK/Livox-SDK.git
  ```
NOTE: The above repository has not been upodated for new compilers - if you receive errors instead clone the repository below:
```
git clone https://github.com/coastalscoop/Livox-SDK-24.04 Livox-SDK

```
At this point, we need to edit some of the files and it appears there is an omission in what is downloaded from Livox. Go to **Livox-SDK/sdk_core/src/base** and edit both the *thread_base.cpp* and *thread_base.h* files to include the text 
```
#include <memory>
```
in the opening lines.

## Edit the Livox Software Development Kit

We will be using the standard lidar_lvx_sample script to collect most data, but it is not entirely fit for purpose at the moment and we want to make some changes to optimise it, and also to save on power.

To save on power, we will turn the lidar unit off manually after the collection script. In our deployed remote settings, this means there is not massive power consumption from the lidar unit whilst we run our processing scripts or send data over the internet (the default is that the lidar stays powered up, even when not actively being called upon to collect data). To do this, navigate to:
```
/home/rpi/Livox-SDK/sample/lidar_lvx_file
```
Open up the main.cpp script in a text editor and look for the following section (around line 192):
```
if (type == kEventConnect) {
    LidarConnect(info);
    printf("[WARNING] Lidar sn: [%s] Connect!!!\n", info->broadcast_code);
  } else if (type == kEventDisconnect) {
    LidarDisConnect(info);
    printf("[WARNING] Lidar sn: [%s] Disconnect!!!\n", info->broadcast_code);
  } else if (type == kEventStateChange) {
    LidarStateChange(info);
    printf("[WARNING] Lidar sn: [%s] StateChange!!!\n", info->broadcast_code);
  }
```
we want to add a simple line of code immediately after connection to make sure the lidar is powered up: 

```
LidarSetMode(info->handle, kLidarModeNormal, nullptr, nullptr);
```

The new section becomes:
```
  if (type == kEventConnect) {
    LidarConnect(info);
    printf("[WARNING] Lidar sn: [%s] Connect!!!\n", info->broadcast_code);
    LidarSetMode(info->handle, kLidarModeNormal, nullptr, nullptr);
  } else if (type == kEventDisconnect) {
    LidarDisConnect(info);
    printf("[WARNING] Lidar sn: [%s] Disconnect!!!\n", info->broadcast_code);
  } else if (type == kEventStateChange) {
    LidarStateChange(info);
    printf("[WARNING] Lidar sn: [%s] StateChange!!!\n", info->broadcast_code);
  }
```

and we also want to make sure the lidar powers down at the end. Look for the following section at the very end of main.cpp:
```
/** Stop the sampling of Livox LiDAR. */
      LidarStopSampling(devices[i].handle, OnStopSampleCallback, nullptr);

    }
  }

/** Uninitialize Livox-SDK. */
  Uninit();
}
```

and update it as follows:
```
/** Stop the sampling of Livox LiDAR. */
      LidarStopSampling(devices[i].handle, OnStopSampleCallback, nullptr);

      LidarSetMode(devices[i].handle,kLidarModePowerSaving, nullptr, nullptr);
      printf("Set to power saving mode");
    }
  }

/** Uninitialize Livox-SDK. */
  Uninit();
}
```

This is also the point whereby we could edit the code to work in repetitive (line scan) mode through the addition of the following text right after our power up script above:
```
   // Set the scan pattern to kRepetitiveScanPattern (1) when connecting
    LidarSetScanPattern(handle, kRepetitiveScanPattern, nullptr, nullptr);
```
as follows:

```
  if (type == kEventConnect) {
    LidarConnect(info);
    printf("[WARNING] Lidar sn: [%s] Connect!!!\n", info->broadcast_code);
    LidarSetMode(info->handle, kLidarModeNormal, nullptr, nullptr);
    // Set the scan pattern to kRepetitiveScanPattern (1) when connecting
    LidarSetScanPattern(handle, kRepetitiveScanPattern, nullptr, nullptr);
  } else if (type == kEventDisconnect) {
    LidarDisConnect(info);
    printf("[WARNING] Lidar sn: [%s] Disconnect!!!\n", info->broadcast_code);
  } else if (type == kEventStateChange) {
    LidarStateChange(info);
    printf("[WARNING] Lidar sn: [%s] StateChange!!!\n", info->broadcast_code);
  }
```

## Compile the code


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

If, however, you need to receive internet and livox data over ethernet then instead keep IPv4 settings as automatic, and manually add a new static IP address. This has currently been hard-coded into the startUp.sh script, so that this is set every time the rpi is powered on. 

Save these changes and close.
## Run the Livox

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

## Connection to AWS
A connection to Amazon Web Service is required in order to upload the data:
```
sudo apt install s3cmd
S3cmd --configure
```
Access key:  

Secret Access key:  

Encryption password [ignore]

Path to GPG [ignore]

Yes to HTTPS

Test: yes

Save: yes

## Automating the process

The next thing I do in the Livox-SDK folder is create a new file which I will use to call my script. I call that file LivoxScheduledSample, and the contents of that file are:

```
#!/bin/bash

# Wait for 60 seconds to give the lidar time to start up
sleep 60

# Change directory to where the Livox sample script is located
cd /home/livox/Livox-SDK/build/sample/lidar_lvx_file/

# Run the Livox sample program with the passed parameters
./lidar_lvx_sample "$@"

# Check if the .lvx file was created
lvx_file=$(ls *.lvx 2>/dev/null)
if [ -z "$lvx_file" ]; then
    echo "No .lvx file found!"
    exit 1
fi

# Move the .lvx file to the external hard drive (assuming it's mounted at /mnt/harddrive)
mv "$lvx_file" /mnt/harddrive/

# Check if the move was successful
if [ $? -ne 0 ]; then
    echo "Failed to move the .lvx file!"
    exit 1
fi

echo ".lvx file successfully moved to /mnt/harddrive/"

# Initiate shutdown of the Raspberry Pi
sudo shutdown -h now
```
In the above the only thing you need to edit is the name and location of the hard drive '/mnt/hardrive/' to whatever it appears as on your RPI.

Explanation of the code:
- sleep 60: Waits for 60 seconds before proceeding.
- cd /home/livox/Livox-SDK/build/sample/lidar_lvx_file/: Changes directory to where the Livox sample script is located.
- ./lidar_lvx_sample "$@": Runs the Livox sample script, allowing you to pass any parameters. The "$@" captures all arguments passed to the bash script.
- lvx_file=$(ls *.lvx 2>/dev/null): Looks for the .lvx file in the directory. If no .lvx file is found, the script will exit with an error message.
- mv "$lvx_file" /mnt/harddrive/: Moves the .lvx file to an external hard drive (assumed to be mounted at /mnt/harddrive/).
- sudo shutdown -h now: Initiates a shutdown of the Raspberry Pi.

You'll need to make sure that the script is executable:

```
chmod +x LivoxScheduledSample.sh
```

and then you can try executing it. It should run the Livox as per before, and if you execute from within a command window you will see the same output as before.

To automate this script as something that executes every time the RPI boots up, I use a cron job:

```
crontab -e
```

and then add the following line to the end of the file:

```
@reboot /home/livox/LivoxScheduledSample -t 60 >> /home/livox/log.txt 2>&1
```

in this case, the parameter '-t 60' refers to a 60 second scan, and you can change that to suit your needs. A list of parameters that you can tweak or change is in the Livox SDK documentation on Github. '/home/livox/log.txt 2>&1' creates a log file that you can check for debugging purposes if the script does not appear to run.

## Important note

In setting this cron job, you are creating an infinite loop whereby every time you turn on the pi, it will fire up, do a scan, and automatically shut down. If you dont want this behaviour (for example, you want the pi to stay on so you can download data etc) then as soon as it boots up, you should go back into the cron scheduler:

```
crontab -e
```

and put a # in front of the @reboot line. Then, next time the Pi boots up it will not automatically start this script.

There are cleaner ways of handling this, such as making the shutdown line of the script conditional on a flag, but that's a matter of preference.

## Troubleshooting

# Unable to use wifi when livox connected
This is a routing priority issue - rpi favours ethernet for internet over wifi, even when internet is not being supplied by the connected ethernet device. You can change the priorities of these connections as follows:
```
sudo nmcli connection modify "Ethernet connection 1" ipv4.route-metric 300
sudo nmcli connection modify "[wifi network name]" ipv4.route-metric 200
sudo nmcli connection up "Ethernet connection 1"
sudo nmcli connection up "[wifi network name]"
```

then use

```
ip route
```
to verify the wifi has been given a lower number [higher priority]
