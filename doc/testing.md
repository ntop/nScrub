---
title: "Utilities and Testing"
description: "This section provides a few examples for checking the mitigation and bridging throughput."
outline: [2, 3]
---
# Utilities and Testing {#utilities-and-testing}

This section provides a few examples for checking the mitigation and bridging throughput.

## SYN Mitigation {#syn-mitigation}

In order to test SYN mitigation you can use one of the SYN flood generators available on internet, such as Juno:

```console
wget http://packetstorm.wowhacker.com/DoS/juno.c
gcc -O2 juno.c -o juno
```

## Throughput {#throughput}

In order to test the bridging throughput you can use iperf:

Target machine:

```console
iperf -s --bind 192.168.1.254
```

Client machine:

```console
iperf -c 192.168.1.254
```
