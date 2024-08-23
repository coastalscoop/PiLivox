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

The next thing I do in the Livox-SDK folder is create a new file which I will use to call my script. I call that file LivoxScheduledSample, and the contents of that file are:

```
#!/bin/bash

# Wait for 60 seconds to give the lidar time to start up
sleep 60

# Change directory to where the Livox sample script is located
cd build/sample/lidar_lvx_file/

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
- cd build/sample/lidar_lvx_file/: Changes directory to where the Livox sample script is located.
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
@reboot /home/livox/LivoxScheduledSample.sh -t 60
```

in this case, the parameter '-t 60' refers to a 60 second scan, and you can change that to suit your needs. A list of parameters that you can tweak or change is in the Livox SDK documentation on Github.

## Important note

In setting this cron job, you are creating an infinite loop whereby every time you turn on the pi, it will fire up, do a scan, and automatically shut down. If you dont want this behaviour (for example, you want the pi to stay on so you can download data etc) then as soon as it boots up, you should go back into the cron scheduler:

```
crontab -e
```

and put a # in front of the @reboot line. Then, next time the Pi boots up it will not automatically start this script.

There are cleaner ways of handling this, such as making the shutdown line of the script conditional on a flag, but that's a matter of preference.
