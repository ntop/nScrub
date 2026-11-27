---
title: "Engine Configuration"
description: "The engine can be configured using the REST API or the CLI tool nscrub-cli with auto-completion support (note this is also using the REST API as backend)."
outline: [2, 3]
---
# Engine Configuration {#engine-configuration}

The engine can be configured using the REST API or the CLI tool nscrub-cli with auto-completion support (note this is also using the REST API as backend).
In order to use the API over HTTPs you need to create a SSL certificate:

```console
openssl req -new -x509 -sha256 -extensions v3_ca -nodes -days 365 -out cert.pem
cat privkey.pem cert.pem > /usr/share/nscrub/ssl/ssl-cert.pem
rm -f privkey.pem cert.em
```

In order to run nScrub using systemd the configuration file should be placed under /etc/nscrub/nscrub.conf. Example:

```console
touch /etc/nscrub/nscrub.start
cat /etc/nscrub/nscrub.conf
--wan-interface=zc:eth1
--lan-interface=zc:eth2
--log-path=/var/tmp/nscrub.log
--pid-path=/var/tmp/nscrub.pid
--daemon
```

Note: if you are using the init.d script on old linux distributions, /etc/nscrub/nscrub.start should be created in order to enable the service.

If your system is using systemd, the application can be run with:

```console
systemctl start nscrub
```

Otherwise please use the init.d script:

```console
/etc/init.d/nscrub start
```

The following options can be specified in the configuration file. For default values please refer to the application help (--help|-h option).

## Basic Settings {#basic-settings}

*[--wan-interface|-i] \&lt;device>*
First device name (internet)

*[--lan-interface|-o] \&lt;device>*
Second device name (local network)

*[--asymmetric|-A]*
Asymmetric routing (wan to lan traffic only)

*[--sw-distribution|-w]*
Use SW Distribution + RSS TX Queues

*[--active-wait|-a]*
Active packet wait

## Host Connectivity/Routing {#host-connectivity-routing}

*[--connect-host|-X]*
Let the host be reachable on the wan/lan interfaces

*[--routing|-x]*
Act as a router (usually used with BGP traffic diversion)

## CPU Affinity {#cpu-affinity}

*[--balancer-affinity|-r] \&lt;id>:\&lt;id>*
Bind packet distribution threads to core ids (-w only)

*[--thread-affinity|-g] \&lt;id>\[:\&lt;id>[..]\]*
Bind processing threads to core id

*[--time-source-affinity|-T] \&lt;id>*
Bind time-source thread to core id

## Traffic Monitoring {#traffic-monitoring}

*[--aux-queues|-O] \&lt;num>*
Enable \&lt;num> sets of auxiliary egress queues for packet sampling (traffic analysis/dumping)

*[--event-script-dir|-Q] \&lt;dir>*
Event scripts directory

## Advanced Settings {#advanced-settings}

*[--cluster-id|-c] \&lt;id>*
ZC cluster ID

*[--ht-idle-timeout|-I] \&lt;usec>*
Flow idle timeout

*[--dyn-white-list-idle-timeout|-e] \&lt;usec>*
Auto-whitelist idle timeout

*[--gre-decapsulation|-E]*
Decapsulate GRE traffic

*[--redis|-D] \&lt;host[:port][@db-id]>*
Redis DB host[:port][@database id]

## REST Server {#rest-server}

*[--http-address|-G] \&lt;address>*
IP address for REST server socket binding

*[--http-port|-H] \&lt;port>*
HTTP port for REST server

*[--https-port|-s] \&lt;port>*
HTTPs port for REST server

*[--docs-root|-R] \&lt;dir>*
Docs root directory

## Logging {#logging}

*[--log-path|-l] \&lt;path>*
Log file path

*[--stats-log-path|-y] \&lt;path>*
Stats log file path

*[--debug-level|-t] \&lt;level>*
Trace level

## Service {#service}

*[--daemon|-d]*
Run as a daemon

*[--pid-path|-p] \&lt;path>*
PID file path

*[--user] \&lt;sys user>*
Run with the specified user

## Version and License {#version-and-license}

*[--version|-V]*
Print version

*[--system-id|-Y]*
Print system identifier

*[--check-license|-C]*
Checks if license is valid

*[--check-maintenance|-M]*
Checks maintenance expiration
