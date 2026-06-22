Utilities and Testing
=====================

This section provides a few examples for checking the mitigation and bridging throughput.

SYN Mitigation
--------------

In order to test SYN mitigation you can use one of the SYN flood generators available on internet, such as Juno:

.. code-block:: console

   wget http://packetstorm.wowhacker.com/DoS/juno.c
   gcc -O2 juno.c -o juno

Throughput
----------

In order to test the bridging throughput you can use iperf:

Target machine:

.. code-block:: console

   iperf -s --bind 192.168.1.254

Client machine:

.. code-block:: console

   iperf -c 192.168.1.254

