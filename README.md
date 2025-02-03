# HPC-Mini-Cluster

Welcome! The following is my procedure for setting up an HPC Mini-Cluster made up of Raspberry Pi devices for ECE 4990. The goal of this project is to learn about HPC and prepare for SC25!

## Getting Started

Before beginning, collect the necessary materials of the Pi Cluster Kit, including the Raspberry Pi devices, micro SD card, power cables, ethernet cables, network switch, and power supply.

Then, begin cabling everything up. Make sure to plug in the power supply and network switch to the wall power, and connect the pis to the power supply and network switches. I recommend following this [link](https://epcced.github.io/wee_archlet/#intro) for the hardware setup.

## SD Card Setup

To install the OS, install the following [tool](https://www.raspberrypi.com/software/). Insert your microSD card into your computer (using an adapter or the microSD card slot) and open the Raspberry Pi Imager. In the GUI, select the version of Raspberry Pi you are using (in my case it was Raspberry Pi 4), click Raspberry Pi OS (other) to select Raspberry Pi OS Lite (64 bit) for our OS, and finally select the SD card that you are using. After clicking Next, you'll want to select Edit Settings to create a hostname, username, password, and enable ssh under the services tab for your OS. I also recommend setting up your Wi-Fi connection in the OS during this step. Finally, apply the settings you just configured and install the OS onto the SD card.

![raspberry pi imager](images/raspberryimager.png)

To make sure that it worked properly, insert your microSD card into the Raspberry Pi and connect a monitor, mouse, and keyboard to the Raspberry Pi. If all went well, you should see a terminal appear on the monitor! Go ahead and repeat these steps for each Raspberry Pi in your setup.

![my setup](images/helloworld.jpg)

## Networking

After setting up your Raspberry Pi devices and installing the operating systems, let's make it to where we can ssh into each device via Wi-Fi. To do this, if you're on-campus, you'll need to obtain the MAC addresses for each device and register them with resmedianet. To do this, you can run the commands `ifconfig` or `ip -c a` to display them while having the Raspberry Pi device connected to another monitor and keyboard. These commands are also extremely useful for debugging your connections by displaying the IP addresses of your ethernet (eth0) and Wi-Fi (wlan0). To actually set the connections up, you'll want to use the commands `sudo nmtui` and/or `sudo raspi-config` to add your Wi-Fi connection. During debugging, I had to disconnect my ethernet cable from my PC to the rest of the cluster to get everything working. Your experience will be different based on your location, but don't be discouarged! To test if your set-up works, you'll want to run the command `ssh <username>@<node IP>`, where the node IP is the IP of wlan0.

![ssh](images/ssh.png)

At this point, your setup should look like something above, so our next step will be to configure the static IPs for the ethernet! Simply type `sudo ntmui`, `Edit a connection`, select the ethernet option, and change `IPv4 CONFIGURATION` to `Manual`. From here, you can configure your own IP address for the setup. For mine, I used 192.168.0 as the base and appended 254 for the head node and 1-2 for the compute nodes. To configure it, it should loook something like this:

![nmtui](images/nmtui.png)

Go ahead and follow a similar process for each of your nodes. To test if it worked, you can use the command `ping <other_ip>` to test sending data between the nodes. You can also try to connect between them using `ssh <username>@<node_cluster_LAN_IP>`. Here's the result of `ip -c a` for my head node and sending data to another node in the network:

![ethernettest](images/ethernettest.png)