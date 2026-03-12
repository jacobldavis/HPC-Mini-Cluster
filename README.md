# HPC-Mini-Cluster

Welcome! The following is my procedure for setting up an HPC Mini-Cluster made up of Raspberry Pi devices. The goal of this project is to learn about HPC and prepare for SC25!

## Getting Started

Before beginning, collect the necessary materials of the Pi Cluster Kit, including the Raspberry Pi devices, micro SD card, power cables, ethernet cables, network switch, and power supply.

Then, begin cabling everything up. Make sure to plug in the power supply and network switch to the wall power, and connect the pis to the power supply and network switches. I recommend following this [link](https://epcced.github.io/wee_archlet/#intro) for the hardware setup.

## SD Card Setup

To install the operating system, install the following [tool](https://www.raspberrypi.com/software/). Insert your microSD card into your computer (using an adapter or the microSD card slot) and open the Raspberry Pi Imager. In the GUI, select the version of Raspberry Pi you are using (in my case it was Raspberry Pi 4), click Raspberry Pi OS (other) to select Raspberry Pi OS Lite (64 bit) for our OS, and finally select the SD card that you are using. After clicking Next, you'll want to select Edit Settings to create a hostname (a nametag for the device on the network), username, password, and enable ssh under the services tab for your OS. I also recommend setting up your Wi-Fi connection in the OS during this step. Finally, apply the settings you just configured and install the OS onto the SD card.

![raspberry pi imager](images/raspberryimager.png)

To make sure that it worked properly, insert your microSD card into the Raspberry Pi and connect a monitor, mouse, and keyboard to the Raspberry Pi. If all went well, you should see a terminal appear on the monitor! Go ahead and repeat these steps for each Raspberry Pi in your setup.

## Networking

After setting up your Raspberry Pi devices and installing the operating systems, let's make it to where we can ssh (Secure Shell, a way to remotely control another computer through the terminal, like texting commands to it from across the room) into each device via Wi-Fi. To do this, if you're on-campus, you'll need to obtain the MAC addresses (a unique hardware identifier burned into every network device) under `wlan0` for each device and register them with [resmedianet](https://hdkb.clemson.edu/phpkb/article.php?id=475). To do this, you can run the commands `ifconfig` or `ip -c a` to display them while having the Raspberry Pi device connected to another monitor and keyboard. These commands are also extremely useful for debugging your connections by displaying the IP addresses of your ethernet (`eth0`) and Wi-Fi (`wlan0`) interfaces. To actually set the connections up, you'll want to use the commands `sudo nmtui` and/or `sudo raspi-config` to add your Wi-Fi connection (`sudo` means "superuser do", similar to asking for admin privileges to run a command). During debugging, I had to disconnect my ethernet cable from my PC to the rest of the cluster to get everything working. Your experience will be different based on your location, but don't be discouraged! To test if your set-up works, you'll want to run the command `ssh <username>@<node IP>` on your computer, where the node IP is the IP of `wlan0`.

![ssh](images/ssh.png)

At this point, your setup should look like something above, so our next step will be to configure the static IPs (a static IP is a fixed, permanent address for a device on the network, as opposed to a dynamic IP that might change every time it reconnects) for the ethernet! Simply type `sudo nmtui`, `Edit a connection`, select the ethernet option, and change `IPv4 CONFIGURATION` to `Manual`. From here, you can configure your own IP address for the setup. For mine, I used 192.168.0 as the base and appended 254/24 for the head node and 1-2/24 for the compute nodes. To configure it, it should look something like this:

![nmtui](images/nmtui.png)

Go ahead and follow a similar process for each of your nodes. To test if it worked, you can use the command `ping <other_ip>` to test sending data between the nodes (ping sends a small packet of data to another device and waits for a response, like asking "hello" across the network!). You can also try to connect between them using `ssh <username>@<node_cluster_LAN_IP>`. Here's the result of `ip -c a` for my head node and sending data to another node in the network:

![ethernettest](images/ethernettest.png)

Finally, we'll set it up to where your nodes can connect to each other without passwords by sharing access keys (think of this like exchanging house keys with a trusted neighbor). Run the command `ssh-keygen` on one of your nodes and press enter for each prompt that follows. Then, run the command `ssh-copy-id <username>@<node2's IP>` to copy the ssh keys to each other node in your network. After doing this, try to `ssh` into each other node, and if it works, you're good to go! Repeat this process for each node. Also try rebooting each node to see if your connections still work. Here's some output of me connecting to another node passwordless:

![passwordless](images/passwordless.png)

## Head and Compute Nodes Setup

A quick note on terminology: the **head node** is the "boss" machine that coordinates everything, while **compute nodes** are the worker machines that perform computation. It's like having a head chef in a kitchen versus the line cooks that follow the head chef's orders.

Let's set up the head node first. On your head node, run the commands `sudo apt-get update` (fetches the latest list of available software) and `sudo apt-get upgrade` (installs those updates). Then, run `sudo nano /etc/modules` (`nano` is a simple terminal text editor) and append "i2c-dev" and "ipv6" to the file as shown below.

![etcmodules](images/etcmodules.png)

Next, run `sudo service rpcbind start` and `sudo apt-get install nfs-kernel-server` to install the NFS (Network File System) Kernel server, which allows multiple computers to share a folder over the network, like a shared Google Drive folder but for your local cluster. Then, we'll create a shared directory by running the following commands: `sudo mkdir -p /home/shared_dir` (creates the folder), `sudo chmod 777 /home/shared_dir` (gives read/write/execute permissions to everyone), and `sudo mount --bind /home/shared_dir/ /home/shared_dir/` to create the directory, add permissions, and mount it onto the NFS server. To ensure it is mounted every time the machine reboots, run `sudo nano /etc/fstab` and add `/home/shared_dir /home/shared_dir none bind 0 0` (`/etc/fstab` is a config file that tells Linux what to automatically mount on startup). Also, run `sudo cat /etc/default/nfs-kernel-server` to check that NEED_SVCGSSD is empty, which means the configuration should be correct.

![need](images/need.png)

Also run `sudo cat /etc/ipmapd.conf` to check that nobody and nogroup are configured for Nobody-User and Nobody-Group. Next, run `sudo nano /etc/exports` and add a line with `/home/shared_dir SubnetAddress(rw,nohide,insecure,no_subtree_check,async)`, where SubnetAddress is your network's subnet address (a subnet is a smaller sub-network carved out of a larger one). You can use this [tool](https://cidr.xyz/) to confirm your SubnetAddress (for my example I inputted 192.168.0.254/24). This part allows for read and write access for the shared directory for any address within the subnet.

![rw](images/rw.png)

Then, check that `/etc/init.d/nfs-kernel-server`, `/etc/init.d/nfs-common`, and `/etc/init.d/rpcbind` have `Default-Start: 2 3 4 5` set (these are runlevels or different "modes" the OS can boot into; 2-5 are the normal multi-user modes you want these services running in). Modify them if not. In my case, nfs-common and rpcbind had Default-Start set to S, so I changed them. Finally, run `sudo update-rc.d -f rpcbind remove`, `sudo update-rc.d rpcbind defaults`, `sudo update-rc.d -f nfs-common remove`, `sudo update-rc.d nfs-common defaults`, `sudo update-rc.d -f nfs-kernel-server remove`, and `sudo update-rc.d nfs-kernel-server defaults` (these commands re-register the services to start automatically at boot). If any fail, run `sudo apt-get purge rpcbind` and `sudo apt-get install nfs-kernel-server`.

Now, let's go ahead and install MPI! MPI (Message Passing Interface) is a standard that lets multiple processes running on different machines communicate with each other, operating as the backbone of most HPC programs. Run `sudo apt-get install libxml2-dev`, `sudo apt-get install zlib1g zlib1g-dev`, and `sudo apt-get install mpich`. You can verify the installation by running `mpiexec --version`. Then, open a hello.c file in a directory in your shared directory and paste the following code:

```c
#include <mpi.h>
#include <stdio.h>
int main(int argc, char** argv) {
    // Initialize the MPI environment
    MPI_Init(NULL, NULL);
    // Get the number of processes
    int world_size;
    MPI_Comm_size(MPI_COMM_WORLD, &world_size);
    // Get the rank of the process
    int world_rank;
    MPI_Comm_rank(MPI_COMM_WORLD, &world_rank);
    // Get the name of the processor
    char processor_name[MPI_MAX_PROCESSOR_NAME];
    int name_len;
    MPI_Get_processor_name(processor_name, &name_len);
    // Print off a hello world message
    printf("Hello world from processor %s, rank %d"
           " out of %d processors\n",
           processor_name, world_rank, world_size);
    // Finalize the MPI environment.
    MPI_Finalize();
}
```

Each MPI process gets a unique "rank" (basically a process ID number), so when you run this across multiple cores or nodes, each one will print its own hello world message identifying itself. Next, compile the program by running `mpicc -o hello hello.c` (`mpicc` is like `gcc` but automatically links in the MPI libraries), and create a hostfile with your node's IP address, and run the code with `mpiexec -n 1 -f hostfile ./hello`. If it worked, MPI is likely installed correctly! Check out this [page](https://rookiehpc.org/) to learn more about MPI and OpenMP.

![hellompi](images/hellompi.png)

Now we'll set up the compute nodes. Perform the following procedure on each compute node. Add the head node's IP address and hostname by running `sudo nano /etc/hosts` (this file maps hostnames to IP addresses locally, so your machine knows where to find "raspberrypi0" without needing to ask a DNS server. In my case, it was 192.168.0.254 raspberrypi0). Next, to install MPI, run the following: `sudo apt-get update && sudo apt-get upgrade`, `sudo apt-get install libxml2-dev`, `sudo apt-get install zlib1g zlib1g-dev`, and `sudo apt-get install mpich`. You can verify the installation by running `mpiexec --version`. Next, set up the shared directory by running `sudo mkdir /home/shared_dir` and `sudo chmod 777 /home/shared_dir/`. Also, make sure that `/etc/init.d/nfs-common` and `/etc/init.d/rpcbind` have Default-Start: 2 3 4 5 set. Then, run the following: `sudo update-rc.d -f rpcbind remove`, `sudo update-rc.d rpcbind defaults`, `sudo update-rc.d -f nfs-common remove`, and `sudo update-rc.d nfs-common defaults`. To mount the shared directory to the NFS server, run `sudo mount HeadNodeIP:/home/shared_dir /home/shared_dir`, where HeadNodeIP is your head node's IP. Additionally, go ahead and run `sudo nano /etc/fstab` to add `HeadNodeIP:/home/shared_dir /home/shared_dir nfs rw,hard,intr,noauto,x-systemd.automount 0 0` so that the directory automatically boots upon reboot.

Let's check to see if everything worked. On your head node, navigate to your `testprogram` directory and make sure the `hostfile` specifies the correct number of nodes with their IPs or aliases and processes you want to use when running MPI. You'll want to run the command `mpiexec -n x -f hostfile ./hello`, where x = # nodes * processes per node.

![hellomanyworlds](images/hellomanyworlds.png)

## Benchmarks!

Now, let's move on to benchmarking. At this point in the guide, I added a fourth raspberry pi to my cluster, so my setup may look different. First, we'll install Spack (Supercomputer PACKage Manager, which is a package manager for HPC software, handling all the complicated dependency management that scientific software requires). In your head node, run `git clone https://github.com/spack/spack.git` in the shared_dir directory (`git clone` downloads a copy of a code repository from the internet) and enter the newly created Spack directory. Next, navigate to the `spack/share/spack` directory and run `. setup-env.sh`. Then, for each node (including the compute nodes), to start the environment each time you reboot, add `source /home/shared_dir/spack/share/spack/setup-env.sh` to your `.bashrc` file (`.bashrc` is a script that runs automatically every time you open a new terminal), and go ahead and run `source ~/.bashrc`. Verify Spack is running by running `spack --version`. You can change your Spack version with `git checkout v<version_number>`.

![spackinstall](images/spackinstall.png)

Now, use Spack to install hpl+openmp by running `spack install hpl+openmp` in your head node. HPL (High Performance LINPACK) is the benchmark used to rank the TOP500 supercomputers in the world by measuring how fast a system can solve a dense linear system of equations, and the result is reported in GFLOPS (billions of floating point operations per second). I initially ran into an error where it would take too long to install openblas, so you may have to install that separately, but overall it should take around thirty minutes. Verify the installation by running `spack find -v hpl`. Next, run `spack install pmix` to install PMIx (Process Management Interface for Exascale, the software layer that manages launching and coordinating all the MPI processes across your cluster), and verify its installation with `spack find -v pmix`. Next, I ran the following commands to start a PMIx server: `export PATH="/home/shared_dir/spack/opt/spack/linux-debian12-aarch64/gcc-12.2.0/bin:$PATH"` and `export PATH="/home/shared_dir/spack/opt/spack/linux-debian12-aarch64/gcc-12.2.0/openmpi-5.0.5-k5blxd7isreqddpejrpul2svexxmhbd3/bin:$PATH"` (`export PATH=...:$PATH` adds a new directory to your PATH, the list of places your shell looks for executable programs without overwriting the existing ones). You may have to modify these based on the versions of gcc/openmpi that you have and other issues. You can verify that the server is running with `ps -ef | grep pmix` (`ps -ef` lists all running processes; the `|` pipes that output into `grep pmix`, which filters it to only show lines containing "pmix"), and if it worked, add those two commands to `~/.bashrc` to launch the PMIx server on startup. Now, run `find / -name "xhpl" 2>/dev/null` to find the path to the LINPACK benchmarks (`2>/dev/null` silences error messages by redirecting them to /dev/null, a special file that discards anything written to it). I recommend creating a script to cd into this directory efficiently (my path was `/home/shared_dir/spack/opt/spack/linux-debian12-aarch64/gcc-12.2.0/hpl-2.3-vf52csiunlnpoiwtee7o2dc22hjydpub/bin`). Finally, create a hostfile with your IP configuration and .env file with paths to your hostfile, xhpl, and HPL.dat (HPL.dat is the configuration file for the benchmark).

![xhplsetup](images/xhplsetup.png)

To start, we'll run one process on one node. Modify the Ns to a size that one processor could handle (N represents the problem size, specifically the dimension of the matrix being solved; a larger N means more work and typically better efficiency up to a point), and set the Ps and Qs to values that multiply to your desired processes (P and Q represent the number of process rows and columns in a 2D grid, and keeping them as square as possible generally performs best). Then, you can execute the benchmark with `mpiexec -n 1 ./xhpl`.

![oneprocessonenodehpldat](images/oneprocessonenodehpldat.png)
![oneprocessonenodetest](images/oneprocessonenodetest.png)

Next, let's run four processes on one node. Modify Ns to a size your processor can handle (I tried 8000) and change the Ps and Qs to values that multiply up to your desired number of total processes (I chose 2 and 2 to make the grid as square as possible). Execute the benchmark with `mpiexec -n 4 ./xhpl`.

![fourprocessesonenodetest](images/fourprocessesonenodetest.png)

Now, let's run one process on four nodes. We can keep the Ns, Ps, and Qs around what you had for the four processes on one node. Now, let's create a helper script to run Linpack across each node in the cluster. Add the following code to the script and make sure to add executing privileges to the script. Then, run `mpiexec -n 4 -hostfile hostfile ./xhpl_runscript` (where -n # is the number of processes corresponding to your Ps x Qs grid size).

```bash
#!/bin/bash
source .env
case ${OMPI_COMM_WORLD_LOCAL_RANK} in
[0])
  HOST=$(sed -n '1p' ${HOSTFILE_A100})
  ;;
[1])
  HOST=$(sed -n '2p' ${HOSTFILE_A100})
  ;;
[2])
  HOST=$(sed -n '3p' ${HOSTFILE_A100})
  ;;
[3])
  HOST=$(sed -n '4p' ${HOSTFILE_A100})
  ;;
esac
echo "Process ${OMPI_COMM_WORLD_LOCAL_RANK} started on ${HOST}"
${XHPL} ${DAT}
```

This script uses `OMPI_COMM_WORLD_LOCAL_RANK` (an environment variable MPI sets automatically for each process's rank number) to figure out which node each process should run on, pulling the corresponding hostname from the hostfile with `sed -n 'Np'` (which prints line N of a file).

![oneprocessfournodestest](images/oneprocessfournodestest.png)

Finally, we'll run multiple processes on four nodes. Add NUM_RANKS_A100=4 to your .env file, modify the Ns (I set it to around 12000 to start), and set the Ps and Qs to something else (I tried both at 3 and 4). You can then run the full cluster benchmark with `mpiexec -n 9 -hostfile hostfile ./xhpl_runscript` (where -n # is the number of processes corresponding to your Ps x Qs grid size).

![fournodesrunone](images/fournodesrunone.png)

## Experimentation

That's it for the main guide! In this section, I'll share some of my thoughts and findings throughout the rest of the project.

First, when we plot GFLOPS vs the number of processes while keeping the problem size static, notice how the GFLOPS increase to a certain extent and drop-off after so many processes. This is caused by a multitude of factors, including increased communication overhead across and within nodes. Similar to a group project, adding more people helps up to a point, but eventually everyone spends more time coordinating than actually working.

![5000](images/5000.png)

Keeping the processes the same and increasing N, the GFLOPS increase to a certain point until the problem size is too much to handle for the processors.

![16](images/16.png)

Here's the best run that I've gotten so far. I used a 3 x 4 setup to reserve some resources for OS processes in the nodes.

![currPr](images/currPr.png)
