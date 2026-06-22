Models and Licenses
===================

The nScrub software comes in two versions, 1G and 10G. Each version comes in various flavours (S/M for 1G, L/XL/XXL for 10G) depending on speed and number of target servers that nScrub can handle.

The 10G version includes support for multithreaded packet capture and processing, with an internal architecture able to balance the load across multiple threads in a zero-copy fashion. This version can handle rates above 10 Gigabit.

Please find all nScrub flavours in the table below.

.. list-table:: Features by Model
   :widths: 25 15 15 15 15 15
   :header-rows: 1

   * - 
     - S
     - M
     - L
     - XL
     - XXL
   * - Link Speed
     - 1 Gigabit
     - 1 Gigabit
     - 10 Gigabit
     - 10 Gigabit
     - 10+ Gigabit
   * - Protected Servers
     - Up to 10
     - Unlimited
     - Up to 10
     - Up to 100
     - Unlimited

License Setup
-------------

nScrub needs a license in order to operate permanently. In order to  obtain a license, you need to go to the web shop and order it using the system ID and nScrub version and model that you want to enable.

Identity version and system ID:

.. code-block:: console

   nscrub -V
   nscrub 1.0 [SystemID: 1234567890]

Get a license and install it:

.. code-block:: console

   echo <LICENSE> > /etc/nscrub.license

You can now start your licensed nScrub copy. It is possible to check license status with:

.. code-block:: console

   nscrub --check-license

and maintenance expiration with:

.. code-block:: console

   nscrub --check-maintenance

Please note that in order to run nScrub on top of PF_RING ZC, you also need ZC licenses, one for each interface you are going to use. In order to obtain ZC licenses, you need to go to the web shop and order them using the MAC addresses as reported by:

.. code-block:: console

   ifconfig <interface>

the driver model as reported by:

.. code-block:: console

   ethtool -i <interface> | grep driver

and the PF_RING version as reported by:

.. code-block:: console

   zcount -h | grep PFRING_ZC

EULA
----

Licensee's use of this software is conditioned upon acceptance of the terms specified in https://svn.ntop.org/svn/ntop/trunk/legal/LicenseAgreement/Reseller/EULA.txt

