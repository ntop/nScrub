Hardware Bypass Support
=======================

When running the software in transparent bridge mode, nScrub supports hardware bypass
adapters that can be used for:

- forwarding the traffic when the system is under maintenance
- implementing an automatic failover mechanism in case of software failures

In bypass mode all packets received from one port are transmitted to the adjacent port.
This mode can be enabled both manually (usually used when the system is under maintenance) 
or leveraging on a heartbeat mechanism to automatically engage the bypass when the software
is not alive for some reason.

nScrub currently supports Silicom bypass adapters. No special configuration is required 
besides installing the Silicom driver as the adapter is automatically detected and the
heartbeat mechanism is automatically enabled.

The bypass mode can be manually anabled at any time by using the *bypass enable* command
in *nscrub-cli*. In the same way it can be disabled using *bypass disable* (this 
automatically enables the heartbeat mechanism). The actual bypass status can be checked
using the *status* command in *nscrub-cli*.

Enabling, disabling, or checking the status of the bypass is also possible through the
REST API.

Please refer to the following section for the adapter configuration.

Silicom Adapters
----------------

In order to enable the bypass support on Silicom adapters, the *bp_ctl* driver (provided
by Silicom) should be compiled and installed:

.. code-block:: console

   tar xvzf bp_ctl-xxx.tar.gz
   cd bp_ctl-xxx
   make && make install

The driver should be loaded using the installed tool (please make sure this is
persistent across system reboots):

.. code-block:: console

   bpctl_start

Please note that a nScrub restart is required after loading the bypass driver in order 
to make sure that the bypass support is detected and enabled successfully.
