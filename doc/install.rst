Installation
============

In order to run nscrub the following major steps are needed:

1. Install and configure PF_RING and ZC drivers (later in this chapter).
2. Install nScrub (later in this chapter) and configure it to run with the optimal setup (Engine Configuration and Performance and Tuning chapters).
3. Configure nScrub to mitigate traffic (Traffic Enforcement Configuration chapter).

Prerequisites:

- PF_RING for packet capture acceleration
- redis-server for storing configuration and system state

Please note that installation packages are provided for some Linux distributions including Ubuntu, Debian and CentOS at http://packages.ntop.org/. 
Please follow the steps on the website for configuring the repository and install at least the *pfring-dkms*, *pfring-drivers-zc-dkms* and *nscrub*.

PF_RING
-------

After installing *pfring-dkms* and *pfring-drivers-zc-dkms*, it is possible to use the init script under /etc/init.d/pf_ring to automate the kernel module and drivers loading as explained in the PF_RING User’s Guide at http://www.ntop.org/guides/pf_ring/get_started/packages_installation.html

For the impatient, this is an example of a minimal configuration for a dual-port ixgbe card (replace ixgbe with your driver model) on a uniprocessor:

.. code-block:: console

   touch /etc/pf_ring/pf_ring.start
   mkdir -p /etc/pf_ring/zc/ixgbe
   touch /etc/pf_ring/zc/ixgbe/ixgbe.start
   echo "RSS=0,0" > /etc/pf_ring/zc/ixgbe/ixgbe.conf
   echo "node=0 hugepagenumber=1024" > /etc/pf_ring/hugepages.conf

Please note that RSS=0,0 will create as many interface queues as the number of cores, however since most Intel cards support up to 16 RSS queues for load balancing, in case of CPUs with many cores it’s a good practice to force it to 16 using RSS=16,16.

Please also note that the number of hugepages requires for memory allocation depends on the number of interface queues (and auxiliary queues if any) and the MTU. For instance if you configure 12 RSS queues with an MTU of 1500 bytes, you probably need up to 2048 hugepages, if you configure 16 RSS queues a common configuration is 4096 hugepages.

In order to load pf_ring, if your system is using systemd please run:

.. code-block:: console

   systemctl start pf_ring

Otherwise please use the init.d script:

.. code-block:: console

   /etc/init.d/pf_ring start

nScrub
------

The nscrub installation is pretty straightforward, just install the *nscrub* package from the repository, then move to the next chapters.
