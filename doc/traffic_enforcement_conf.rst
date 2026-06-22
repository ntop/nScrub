Traffic Enforcement Configuration
=================================

Once the application is up and running, it’s time to configure it for enabling traffic mitigation. This means we need to create virtual scrubbers (objects containing protection policies based on target), each virtual scrubber inspects the traffic matching one, or more, destination subnets. Each virtual scrubber, identified by a target ID, has its own traffic enforcement profiles that can be configured/changed/inspected at runtime using the API.

This sections provides some basic knowledge for configuring the engine using the REST API. Please refer to the API documentation for the full API specifications. 

Note that although this section covers the configuration using the REST API, a few command line tools (see Appendix A) are also available to ease the configuration, including:

- *nscrub-cli*, implementing a console with autocompletion with all the functionalities implemented by the REST API.
- *nscrub-add*, a wizard tool for creating new victims with a basic configuration (further customisations are usually needed using nscrub-cli or the API).
- *nscrub-export*, a tool for dumping the current configuration for a specific victim.

Default credentials for configuring nscrub:

+-----------------+-----------+
|  Username       |     admin |
+-----------------+-----------+
|  Password       |     admin |
+-----------------+-----------+
|  HTTP port      |      8880 |
+-----------------+-----------+
|  HTTPs port     |      4443 |
+-----------------+-----------+
|  Socket binding | localhost |
+-----------------+-----------+

.. note:: nScrub listens on localhost by default, please configure a different address (-G option) to use the REST API from a remote machine.

User Management
---------------

Changing the Password
~~~~~~~~~~~~~~~~~~~~~

The password can be changed while nScrub is running using the REST API or the CLI.

**Using nscrub-cli:**

.. code-block:: console

   nscrub-cli> usermod <username> <group> <new_password>

Example to change the admin password:

.. code-block:: console

   nscrub-cli> usermod admin administrator newpassword

**Using the REST API directly:**

.. code-block:: console

   curl -u admin:admin "http://localhost:8880/users?action=update&username=admin&group=administrator&password=newpassword"

Adding and Removing Users
~~~~~~~~~~~~~~~~~~~~~~~~~

**Using nscrub-cli:**

.. code-block:: console

   nscrub-cli> useradd <username> <group> <password>    # add a new user
   nscrub-cli> userdel <username>                       # remove a user
   nscrub-cli> users                                    # list all users

**Using the REST API:**

.. code-block:: console

   curl -u admin:admin "http://localhost:8880/users?action=add&username=john&group=administrator&password=secret"
   curl -u admin:admin "http://localhost:8880/users?action=del&username=john"
   curl -u admin:admin "http://localhost:8880/users?action=list"

Password Reset (Emergency)
~~~~~~~~~~~~~~~~~~~~~~~~~~

Passwords are stored as MD5 hashes in Redis under the key ``nscrub.user.<username>.password``.
If you lost the admin password, you can reset it following the instructions below:

1. Shutdown nscrub
2. Run ``redis-cli del nscrub.user.admin.password``
3. Restart nscrub to recreate the admin account with the default password ``admin``

Traffic Enforcement Logic
-------------------------

Understanding how nScrub processes and enforces traffic policies is essential for proper configuration.
This section explains the core traffic enforcement mechanism and how profiles interact to provide
DDoS protection while allowing legitimate traffic.

Profile-Based Traffic Inspection
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Each virtual scrubber (victim) has four profiles that define traffic enforcement policies:

- **DEFAULT profile**: Applied to traffic from IP addresses not in any list (unknown sources)
- **WHITE profile**: Applied to whitelisted IP addresses (verified legitimate sources)
- **BLACK profile**: Applied to blacklisted IP addresses (known attackers)
- **GRAY profile**: Applied to IP addresses requiring special handling

WAN to LAN Traffic Flow (Inbound Traffic Inspection)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Traffic arriving from the WAN interface destined to protected subnets (victims) is inspected based on
the source IP address and the matching profile:

1. **Unknown Sources (DEFAULT profile)**:

   - Traffic from sources not in any list (WHITE, BLACK, or GRAY) is evaluated by the DEFAULT profile
   - Typically configured with a "drop by default" policy since traffic should only pass after verification
   - The DEFAULT profile contains validation checks (SYN checks, DNS checks, rate limits, etc.)
   - When a packet passes all checks and is verified as legitimate, the source IP is **automatically
     added to the WHITE list**
   - Subsequent packets from that source will then be processed by the WHITE profile

2. **Whitelisted Sources (WHITE profile)**:

   - Traffic from sources in the WHITE list is processed by the WHITE profile
   - Should be configured with an "accept all" policy to allow legitimate traffic to flow freely
   - Sources are added to WHITE list either:

     - Automatically after successful verification by DEFAULT profile checks
     - Manually via API/CLI
     - From LAN to WAN traffic (see below)

3. **Blacklisted Sources (BLACK profile)**:

   - Traffic from sources in the BLACK list is processed by the BLACK profile
   - Typically configured to drop all traffic
   - Sources are added to BLACK list either:

     - Manually via API/CLI
     - Automatically by auto-blacklist features when sources fail validation or exceed rate limits

4. **Gray-listed Sources (GRAY profile)**:

   - Traffic from sources in the GRAY list requires special policies
   - Useful for applying custom rules to specific IP addresses or subnets
   - Typically configured with default "drop" and specific allow rules for certain traffic types

LAN to WAN Traffic Flow (Outbound Traffic)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Traffic from the LAN (protected network) to WAN is handled differently:

- **All LAN to WAN traffic is always passed** without inspection
- When a packet is sent from the LAN, the **destination IP is automatically whitelisted**
- This ensures return traffic (where the whitelisted destination becomes the source on WAN)
  is accepted through the WHITE profile
- This mechanism allows legitimate bidirectional communication initiated from the protected network

**Important**: If return traffic is not working as expected, this typically indicates a misconfiguration
that requires debugging (e.g., incorrect WHITE profile policy, routing issues, or asymmetric routing problems).

Traffic Verification and Promotion Example
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Here's a typical flow for a new TCP connection:

1. External client (1.2.3.4) sends SYN packet to victim (10.0.0.1)
2. Source 1.2.3.4 is not in any list → DEFAULT profile applies
3. DEFAULT profile has SYN check enabled (e.g., RFC compliance check)
4. If SYN packet passes validation:

   - Packet is forwarded to victim
   - Source 1.2.3.4 is added to WHITE list

5. Subsequent packets from 1.2.3.4 → WHITE profile applies
6. WHITE profile has "accept all" → packets flow freely
7. If idle timeout expires (with autopurging enabled), 1.2.3.4 may be removed from WHITE list

Configuration Best Practices
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

For proper traffic enforcement:

1. **WHITE profile**: Set "accept all" policy

   .. code-block:: console

      curl -u admin:admin "http://localhost:8880/profile/all/accept?target_id=VICTIM&profile=white&action=enable"

2. **BLACK profile**: Set "drop all" policy

   .. code-block:: console

      curl -u admin:admin "http://localhost:8880/profile/all/drop?target_id=VICTIM&profile=black&action=enable"

3. **DEFAULT profile**: Configure with validation checks and "drop by default"

   .. code-block:: console

      curl -u admin:admin "http://localhost:8880/profile/default?target_id=VICTIM&profile=default&action=update&value=drop"
      curl -u admin:admin "http://localhost:8880/profile/tcp/syn/check_method?target_id=VICTIM&profile=default&action=update&value=rfc"

4. **Dynamic list management**: Enable autopurging to automatically remove idle whitelisted IPs

   .. code-block:: console

      curl -u admin:admin "http://localhost:8880/attackers/dynamic/autopurging?target_id=VICTIM&action=enable"
      curl -u admin:admin "http://localhost:8880/attackers/dynamic/expiration?target_id=VICTIM&action=update&value=3600"

Please check the `Step-By-Step Guide <https://www.ntop.org/a-step-by-step-guide-for-protecting-your-network-with-nscrub/>`__ for a full (basic) configuration example.

Victims definition
------------------

Victims can be dynamically added, removed and configured at runtime.
This section shows some examples of the most common runtime settings, focusing on the REST API.
The below sections describe all the available API calls to configure the engine using both the CLI and
the REST API.

Read active victims:

.. code-block:: console

   curl -u <user>:<password> "http://<host>:<port>/targets?action=list"

or do the same using the command line tool:

.. code-block:: console

   nscrub-cli 
   localhost:8880> list targets

Add a new victim:

.. code-block:: console

   curl -u <user>:<password> "http://<host>:<port>/targets?action=add&target_id=<victim name>&subnet=<subnet (CIDR notation)>"

Each victim is bound to four profiles: default, black, white, and gray. These profiles define how traffic is processed based on the source IP address. For a detailed explanation of how profiles work and the traffic enforcement logic, see the "Traffic Enforcement Logic" section above.

.. note:: This section provides only a few examples of victim configuration, for the full settings please refer to the API documentation.

It is a common practice to set the "drop all" policy to the black profile:

.. code-block:: console

   curl -u <user>:<password> "http://<host>:<port>/profile/all/drop?target_id=<victim name>&profile=black&action=enable"

It is also a common practice to set the "accept all" policy to the white profile:

.. code-block:: console

   curl -u <user>:<password> "http://<host>:<port>/profile/all/accept?target_id=<victim name>&profile=white&action=enable"

The gray profile is usually used for applying special policies to "special" IPs. For instance it is a common practice to set the "default" policy to "drop" and then specify more specific policies to let specific traffic types through.

.. code-block:: console

   curl -u <user>:<password> "http://<host>:<port>/profile/default?target_id=<victim name>&profile=gray&action=update&value=drop"

The default profile is where the real traffic enforcement policies go, for checking unknown traffic. For instance it is also a common practice to set the default policy to drop:

.. code-block:: console

   curl -u <user>:<password> "http://<host>:<port>/profile/default?target_id=<victim name>&profile=default&action=update&value=drop"

Accept ICMP:

.. code-block:: console

   curl -u <user>:<password> "http://<host>:<port>/profile/icmp/accept?target_id=<victim name>&profile=default&action=enable"

Drop UDP:

.. code-block:: console

   curl -u <user>:<password> "http://<host>:<port>/profile/udp/drop?target_id=<victim name>&profile=default&action=enable"

Accept UDP port 53 (DNS):

.. code-block:: console

   curl -u <user>:<password> "http://<host>:<port>/profile/udp/src/53/accept?target_id=<victim name>&profile=default&action=enable"

Check TCP traffic:

.. code-block:: console

   curl -u <user>:<password> "http://<host>:<port>/profile/tcp/syn/check_method?target_id=<victim name>&profile=default&action=update&value=rfc"

It is also possible to set a rate limiter (in this example per source) to set a threshold to the traffic rate.

.. code-block:: console

   curl -u <user>:<password> "http://<host>:<port>/profile/rate?target_id=<victim name>&profile={black, white, gray, default}[&action=update&value=<pkts/s>]"

Many more policies are available, please refer to the full API documentation. 

Please note all the settings can also be read, omitting the action (and value) parameter.

In order to temporarily disable traffic checks, it is possible to put the system in bypass state, both globally:

.. code-block:: console

   curl -u <user>:<password> "http://<host>:<port>/bypass?[action={enable, disable}]"

or per victim:

.. code-block:: console

   curl -u <user>:<password> "http://<host>:<port>/profile/bypass?target_id=<victim name>&profile=default[&action={enable, disable}]"


Global Settings
---------------

Application version, configuration and status
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

*CLI*

.. code-block:: console

  > status

*REST*

.. code-block:: console

  curl -u <user>:<password> http://<host>:<port>/status

Example:

.. code-block:: console

  curl -u admin:admin http://localhost:8880/

Stats
~~~~~

*CLI*

.. code-block:: console

  > stats

*REST*

.. code-block:: console

  curl -u <user>:<password> http://<host>:<port>/stats


Example:


.. code-block:: console

  curl -u admin:admin http://localhost:8880/stats

Configure system name
~~~~~~~~~~~~~~~~~~~~~

*CLI*

.. code-block:: console

  > hostname [NAME]

*REST*

.. code-block:: console

  curl -u <user>:<password> http://<host>:<port>/hostname?[action=update\&value=<NAME>]

Example:

.. code-block:: console

  curl -u admin:admin http://localhost:8880/hostname

Configure system description
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

*CLI*

.. code-block:: console

  > desc [DESCRIPTION]

*REST*

.. code-block:: console

  curl -u <user>:<password> http://<host>:<port>/desc?[action=update\&value=<DESCRIPTION>]

Example:

.. code-block:: console

  curl -u admin:admin http://localhost:8880/desc


Configure global bypass
~~~~~~~~~~~~~~~~~~~~~~~

Hardware bypass is used when available.
Note: this is a full bypass, does not handle routing (when enabled).

*CLI*

.. code-block:: console

  > bypass [enable|disable]

*REST*

.. code-block:: console

  curl -u <user>:<password> http://<host>:<port>/bypass?[action={enable, disable}]

Example:


.. code-block:: console

  curl -u admin:admin http://localhost:8880/bypass

.. code-block:: console

  curl -u admin:admin http://localhost:8880/bypass?action=enable

.. code-block:: console

  curl -u admin:admin http://localhost:8880/bypass?action=disable

Read the neighbor table
~~~~~~~~~~~~~~~~~~~~~~~

Read the ARP Table.
Note that nScrub automatically learns neighbors reading the system arp table,
thus you can manage neighbors using the standard arp commands. 
Example of manually adding an entry:   $ arp -i eth1 -s 192.168.1.85 00:b1:ac:50:17:00
Example of manually deleting an entry: $ arp -d 192.168.1.85

*CLI*

.. code-block:: console

  > neigh

*REST*

.. code-block:: console

  curl -u <user>:<password> http://<host>:<port>/neigh?action=list

Example:

.. code-block:: console

  curl -u admin:admin http://localhost:8880/neigh?action=list

Set WAN/LAN IP
~~~~~~~~~~~~~~

Set the IP address for the WAN or LAN interfaces (changes are applied on nscrub restart)

*CLI*

.. code-block:: console

  > ip
  > ip WAN|LAN set IP

*REST*

.. code-block:: console

  curl -u <user>:<password> http://<host>:<port>/ip?action=update&interface={WAN, LAN}&value=<IP>

Example:

.. code-block:: console

  curl -u admin:admin http://localhost:8880/ip

.. code-block:: console

  curl -u admin:admin http://localhost:8880/ip?action=update\&interface=WAN\&value=10.10.10.1

Read the routing table
~~~~~~~~~~~~~~~~~~~~~~

Routing mode only.

*CLI*

.. code-block:: console

  > route

*REST*

.. code-block:: console

  curl -u <user>:<password> http://<host>:<port>/route?action=list

Example:

.. code-block:: console

  curl -u admin:admin http://localhost:8880/route?action=list

Update the routing table
~~~~~~~~~~~~~~~~~~~~~~~~

Routing mode only.

*CLI*

.. code-block:: console

  > route add SUBNET gw IP
  > route del SUBNET

*REST*

.. code-block:: console

  curl -u <user>:<password> http://<host>:<port>/route?[action={add, del}\&destination=<CIDR>[\&gw=<IP>]]

Example:

.. code-block:: console

  curl -u admin:admin http://localhost:8880/route

.. code-block:: console

  curl -u admin:admin http://localhost:8880/route?action=add\&destination=default\&gw=192.168.1.1

.. code-block:: console

  curl -u admin:admin http://localhost:8880/route?action=add\&destination=10.10.10.0/24\&gw=10.10.10.1

.. code-block:: console

  curl -u admin:admin http://localhost:8880/route?action=del\&destination=10.10.10.0/24

Configure VLAN reforging
~~~~~~~~~~~~~~~~~~~~~~~~

This is used to map ingress VLAN to egress VLAN.
Note: to remove a mapping set Src-ID = Dest-ID

*CLI*

.. code-block:: console

  > vlan id SRC-VLAN-ID reforge [DST-VLAN-ID]

*REST*

.. code-block:: console

  curl -u <user>:<password> http://<host>:<port>/vlan/id/<Src-ID>/reforge?[action=update\&value=<Dest-ID>]

Example:

.. code-block:: console

  curl -u admin:admin http://localhost:8880/vlan/id/2/reforge

.. code-block:: console

  curl -u admin:admin http://localhost:8880/vlan/id/2/reforge?action=update\&value=3

Read the VLAN reforging list
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

*CLI*

.. code-block:: console

  > vlan

*REST*

.. code-block:: console

  curl -u <user>:<password> http://<host>:<port>/vlan/id?action=list

Example:

.. code-block:: console

  curl -u admin:admin http://localhost:8880/vlan/id?action=list

Configure traffic mirroring
~~~~~~~~~~~~~~~~~~~~~~~~~~~

Define how traffic is sent to the auxiliary queues.

*CLI*

.. code-block:: console

  > mirror ID
  > mirror ID type [forwarded|discarded|injected|all]
  > mirror ID sampling [RATE]

*REST*

.. code-block:: console

  curl -u <user>:<password> http://<host>:<port>/mirror/<queue set id>/type[?action=update\&value={forwarded, discarded, injected, all}]

.. code-block:: console

  curl -u <user>:<password> http://<host>:<port>/mirror/<queue set id>/sampling[?action=update\&value=<sampling rate (0 for no traffic)>]

Example:

.. code-block:: console

  curl -u admin:admin http://localhost:8880/mirror/0/type?action=update\&value=all

.. code-block:: console

  curl -u admin:admin http://localhost:8880/mirror/0/sampling?action=update\&value=1

Configure the runtime debug level
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

*CLI*

.. code-block:: console

  > debug [LEVEL]

*REST*


.. code-block:: console

  curl -u <user>:<password> http://<host>:<port>/debug[?action=update\&value=<level>]

Example:

.. code-block:: console

  curl -u admin:admin http://localhost:8880/debug

.. code-block:: console

  curl -u admin:admin http://localhost:8880/debug?action=update\&value=2

Configure peer (MAC) policy 
~~~~~~~~~~~~~~~~~~~~~~~~~~~

This is usually not needed unless you want to blacklist a peer.

*CLI*

.. code-block:: console

  > peer
  > peer add MAC policy pass|drop
  > peer del MAC

*REST*

.. code-block:: console

  curl -u <user>:<password> http://<host>:<port>/peers?action={add,del,list}[\&address=<mac>[\&value={pass, drop}]]

Example:

.. code-block:: console

  curl -u admin:admin http://localhost:8880/peers?action=list

.. code-block:: console

  curl -u admin:admin http://localhost:8880/peers?action=add\&address=00:11:22:33:44:55\&policy=drop

.. code-block:: console

  curl -u admin:admin http://localhost:8880/peers?action=del\&address=00:11:22:33:44:55

Targets Management
------------------

Read targets list
~~~~~~~~~~~~~~~~~

*CLI*

.. code-block:: console

  > list targets

*REST*

.. code-block:: console

  curl -u <user>:<password> http://<host>:<port>/targets?action=list

Example:

.. code-block:: console

  curl -u admin:admin http://localhost:8880/targets?action=list

Add/del subnets from targets
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

If the target does not exists, it creates a new target.

*CLI*

.. code-block:: console

  > add target ID SUBNET
  > del target ID SUBNET

*REST*

.. code-block:: console

  curl -u <user>:<password> http://<host>:<port>/targets?action={add, del}\&target_id=<target id>\&subnet=<CIDR>

Example:

.. code-block:: console

  curl -u admin:admin http://localhost:8880/targets?action=add\&target_id=SCRBR1\&subnet=10.10.11.1/32

.. code-block:: console

  curl -u admin:admin http://localhost:8880/targets?action=del\&target_id=SCRBR1\&subnet=10.10.11.1/32

Delete a target by name
~~~~~~~~~~~~~~~~~~~~~~~

Delete a target and its configuration.
Use * for all targets.

*CLI*

.. code-block:: console

  > purge target ID|*

*REST*

.. code-block:: console

  curl -u <user>:<password> http://<host>:<port>/targets?action=target_del\&target_id={<target id>,*}

Example:

.. code-block:: console

  curl -u admin:admin http://localhost:8880/targets?action=target_del\&target_id=SCRBR1

Set a description for the target
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

*CLI*

.. code-block:: console

  > target ID desc [DESCRIPTION]

*REST*


.. code-block:: console

  curl -u <user>:<password> http://<host>:<port>/targets/desc?target_id=<target id>[\&action=update\&value=<DESCRIPTION>]

Example:

.. code-block:: console

  curl -u admin:admin http://localhost:8880/targets/desc?target_id=SCRBR1 

Configure VLAN reforging for traffic towards the target
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

This can be used as an alternative to the global mapping /vlan/id/{Src-ID}/reforge
Note: to disable reforging set Dest-ID = 0

*CLI*

.. code-block:: console

  > target ID vlan reforge [DST-VLAN-ID]

*REST*

.. code-block:: console

  curl -u <user>:<password> http://<host>:<port>/targets/vlan/reforge?target_id=<target id>[\&action=update\&value=<Dest-ID>]

Example:

.. code-block:: console

  curl -u admin:admin http://localhost:8880/targets/vlan/reforge?target_id=SCRBR1 

.. code-block:: console

  curl -u admin:admin http://localhost:8880/targets/vlan/reforge?target_id=SCRBR1\&action=update\&value=16

Configure target type
~~~~~~~~~~~~~~~~~~~~~

Target types (Web server, Game server, DNS server, ISP clients, etc) are used to give hints to the engine and optimise the protection algorithms.

*CLI*

.. code-block:: console

  > target ID type [web|dns|game|isp]

*REST*

.. code-block:: console

  curl -u <user>:<password> http://<host>:<port>/targets/type?target_id=<target id>[\&action=update\&value={web,dns,game,isp}]

Example:

.. code-block:: console

  curl -u admin:admin http://localhost:8880/targets/type?target_id=SCRBR1

.. code-block:: console

  curl -u admin:admin http://localhost:8880/targets/type?target_id=SCRBR1\&action=update\&value=web

Read target stats
~~~~~~~~~~~~~~~~~

Read (inbound traffic only) stats for a target.
Note: this accepts regexp (e.g. 'webserver_[0-9]*') as target id.

*CLI*

.. code-block:: console

  > target ID stats

*REST*

.. code-block:: console

  curl -u <user>:<password> http://<host>:<port>/targets/stats?target_id=<target id>

Example:

.. code-block:: console

  curl -u admin:admin http://localhost:8880/targets/stats?target_id=SCRBR1

Attackers Management
--------------------

Read attackers
~~~~~~~~~~~~~~

Read the attackers for a specific target, specifying the list name, and filtering by profile
Note:

 - profile=* means all the attackers, profile=white/black/gray to select IPs matching a profile
 - list=* means all the lists
 - up to 100 items are returned by default, or limit if provided. The offset parameter can be used to handle pagination.

*CLI*

.. code-block:: console

  > target ID attackers show LISTNAME|* white|gray|black|*

*REST*

.. code-block:: console

  curl -u <user>:<password> http://<host>:<port>/attackers?target_id=<target id>\&action=list\&profile={black, white, gray, *}\&list={<list name>, *}[\&offset=<offset>][\&limit=<max items>]

Example:

.. code-block:: console

  curl -u admin:admin http://localhost:8880/attackers?target_id=SCRBR1\&action=list\&profile=*\&list=Test

.. code-block:: console

  curl -u admin:admin http://localhost:8880/attackers?target_id=SCRBR1\&action=list\&profile=*\&list=*\&offset=0\&limit=500

Add/Delete attackers
~~~~~~~~~~~~~~~~~~~~

Add/del items from an attacker list (optionally you can specify a lifetime for adding attackers to the dynamic list, in this case attackers are not persistent on application restart)

*CLI*

.. code-block:: console

  > target ID attackers add LISTNAME SUBNET white|gray|black [SEC]
  > target ID attackers del LISTNAME SUBNET

*REST*

.. code-block:: console

  curl -u <user>:<password> http://<host>:<port>/attackers?target_id=<target id>\&action={add, del}\&list=<list name>\&subnet=<CIDR>[\&profile={black, white, gray}][\&lifetime=<seconds>]

Example:

.. code-block:: console

  curl -u admin:admin http://localhost:8880/attackers?target_id=SCRBR1\&action=add\&list=Test\&subnet=10.10.11.1/32\&profile=black

.. code-block:: console

  curl -u admin:admin http://localhost:8880/attackers?target_id=SCRBR1\&action=del\&list=Test\&subnet=10.10.11.1/32

Delete an attacker list
~~~~~~~~~~~~~~~~~~~~~~~

*CLI*

.. code-block:: console

  > 

*REST*

.. code-block:: console

  curl -u <user>:<password> http://<host>:<port>/attackers?target_id=<target id>\&action=list_del\&list=<list name>

Example:

.. code-block:: console

  curl -u admin:admin http://localhost:8880/attackers?target_id=SCRBR1\&action=list_del\&list=Test

Purge all attackers
~~~~~~~~~~~~~~~~~~~

This also deletes all lists.

*CLI*

.. code-block:: console

  > target ID attackers purge all

*REST*

.. code-block:: console

  curl -u <user>:<password> http://<host>:<port>/attackers?target_id=<target id>\&action=purge

Example:

.. code-block:: console

  curl -u admin:admin http://localhost:8880/attackers?target_id=SCRBR1\&action=purge

List all attackers list
~~~~~~~~~~~~~~~~~~~~~~~

List all attackers lists for a target (this also returns the number of entries in each list)

*CLI*

.. code-block:: console

  > target ID attackers showlists

*REST*

.. code-block:: console

  curl -u <user>:<password> http://<host>:<port>/attackers?target_id=<target id>\&action=list_ls

Example:

.. code-block:: console

  curl -u admin:admin http://localhost:8880/attackers?target_id=SCRBR1\&action=list_ls

Search attackers
~~~~~~~~~~~~~~~~

Search for an attacker in a specific list (by name), all lists (using '*'), all configured lists 
(using 'static') or all dynamically whitelisted/blacklisted IPs (using 'dynamic').
Returns the list names where the subnet is defined.

*CLI*

.. code-block:: console

  > target ID attackers search LISTNAME|dynamic|* white|gray|black|* SUBNET

*REST*

.. code-block:: console

  curl -u <user>:<password> http://<host>:<port>/attackers?target_id=<target id>\&action=search\&list={<list name>, dynamic, static, *}\&profile={black, white, gray, *}\&subnet=<CIDR>

Example:

.. code-block:: console

  curl -u admin:admin http://localhost:8880/attackers?target_id=SCRBR1\&action=search\&list=*\&profile=*\&subnet=10.10.11.1/32

.. code-block:: console

  curl -u admin:admin http://localhost:8880/attackers?target_id=SCRBR1\&action=search\&list=dynamic\&profile=*\&subnet=10.10.11.1/32

Read dynamic list
~~~~~~~~~~~~~~~~~

This will also include subnets in static lists.

*CLI*

.. code-block:: console

  > target ID attackers show dynamic white|gray|black|*

*REST*

.. code-block:: console

  curl -u <user>:<password> http://<host>:<port>/attackers?target_id=<target id>\&action=list\&profile={black, white, gray}\&list=dynamic

Example:

.. code-block:: console

  curl -u admin:admin http://localhost:8880/attackers?target_id=SCRBR1\&action=list\&profile=white\&list=dynamic

Purge dynamic list
~~~~~~~~~~~~~~~~~~

*CLI*

.. code-block:: console

  > target ID attackers purge dynamic

*REST*

.. code-block:: console

  curl -u <user>:<password> http://<host>:<port>/attackers/dynamic?target_id=<target id>\&action=purge

Example:

.. code-block:: console

  curl -u admin:admin http://localhost:8880/attackers/dynamic?target_id=SCRBR1\&action=purge

Configure dynamic list autopurging
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

(auto remove dynamically whitelisted IPs on idle timeout)

*CLI*

.. code-block:: console

  > target ID attackers dynamic autopurging [enable|disable]

*REST*

.. code-block:: console

  curl -u <user>:<password> http://<host>:<port>/attackers/dynamic/autopurging?target_id=<target id>[\&action={enable, disable}]

Example:

.. code-block:: console

  curl -u admin:admin http://localhost:8880/attackers/dynamic/autopurging?target_id=SCRBR1

.. code-block:: console

  curl -u admin:admin http://localhost:8880/attackers/dynamic/autopurging?target_id=SCRBR1\&action=enable

Configure dynamic list expiration for autopurging
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

*CLI*

This is the idle timeout for IPs which are automatically whitelisted or blacklisted by the engine.

.. code-block:: console

  > target ID attackers dynamic expiration [SEC]

*REST*

.. code-block:: console

  curl -u <user>:<password> http://<host>:<port>/attackers/dynamic/expiration?target_id=<target id>[\&action=update\&value=<sec>]

Example:

.. code-block:: console

  curl -u admin:admin http://localhost:8880/attackers/dynamic/expiration?target_id=SCRBR1

.. code-block:: console

  curl -u admin:admin http://localhost:8880/attackers/dynamic/expiration?target_id=SCRBR1\&action=update\&value=3600

Batch add/delete attackers
~~~~~~~~~~~~~~~~~~~~~~~~~~

Add/del multiple items in a single call from an attacker list (JSON array via POST)
Note: add-fast action updates the datapath faster (less impact on traffic), however it flushed all dynamically-added IPs and does not handle duplicate items across lists

*CLI*

.. code-block:: console

  > target ID attackers load LISTNAME FILEPATH white|gray|black [SEC]

*REST*

Example:

.. code-block:: console

  curl -u admin:admin -X POST http://localhost:8880/attackers?target_id=SCRBR1\&action={add, add-fast}\&list=Test\&profile=black -d '["1.1.1.1/32","2.2.2.2/32"]'

.. code-block:: console

  curl -u admin:admin -X POST http://localhost:8880/attackers?target_id=SCRBR1\&action=del\&list=Test -d '["1.1.1.1/32","2.2.2.2/32"]'

Targets Profiles Configuration
------------------------------

Configure bypass
~~~~~~~~~~~~~~~~

This can be set on the 'default' profile only and overwrites all more specific profiles.

*CLI*

.. code-block:: console

  > target ID profile white|gray|black|default bypass [enable|disable]

*REST*

.. code-block:: console

  curl -u <user>:<password> http://<host>:<port>/profile/bypass?target_id=<target id>\&profile=default[\&action={enable, disable}]

Example:

.. code-block:: console

  curl -u admin:admin http://localhost:8880/profile/bypass?target_id=SCRBR1\&profile=default

.. code-block:: console

  curl -u admin:admin http://localhost:8880/profile/bypass?target_id=SCRBR1\&profile=default\&action=disable

Configure default action
~~~~~~~~~~~~~~~~~~~~~~~~

*CLI*

.. code-block:: console

  > target ID profile white|gray|black|default default [drop|pass]

*REST*

.. code-block:: console

  curl -u <user>:<password> http://<host>:<port>/profile/default?target_id=<target id>\&profile={black, white, gray, default}[\&action=update\&value={pass, drop}]

Example:

.. code-block:: console

  curl -u admin:admin http://localhost:8880/profile/default?target_id=SCRBR1\&profile=default

.. code-block:: console

  curl -u admin:admin http://localhost:8880/profile/default?target_id=SCRBR1\&profile=default\&action=update\&value=drop

Rate limiting per source/dest
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Configure per-source (attacker) per-dest (victim) rate limiting (pkts/s)

*CLI*

.. code-block:: console

  > target ID profile white|gray|black|default rate src [PPS]
  > target ID profile white|gray|black|default rate dst [PPS]

*REST*

.. code-block:: console

  curl -u <user>:<password> http://<host>:<port>/profile/rate/{src, dst}?target_id=<target id>\&profile={black, white, gray, default}[\&action=update\&value=<pkts/s>]

Example:

.. code-block:: console

  curl -u admin:admin http://localhost:8880/profile/rate/src?target_id=SCRBR1\&profile=default

.. code-block:: console

  curl -u admin:admin http://localhost:8880/profile/rate/src?target_id=SCRBR1\&profile=default\&action=update\&value=100

Configure all traffic drop/accept
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

*CLI*

.. code-block:: console

  > target ID profile white|gray|black|default all drop [enable|disable]
  > target ID profile white|gray|black|default all accept [enable|disable]

*REST*

.. code-block:: console

  curl -u <user>:<password> http://<host>:<port>/profile/all/{accept, drop}?target_id=<target id>\&profile={black, white, gray, default}[\&action={enable, disable}]

Example:

.. code-block:: console

  curl -u admin:admin http://localhost:8880/profile/all/drop?target_id=SCRBR1\&profile=black

.. code-block:: console

  curl -u admin:admin http://localhost:8880/profile/all/drop?target_id=SCRBR1\&profile=black\&action=enable

Read UDP/TCP/ICMP policies
~~~~~~~~~~~~~~~~~~~~~~~~~~

Read a summary of the configured policies for each protocol.

*CLI*

.. code-block:: console

  > target ID profile white|gray|black|default [ip|udp|tcp|icmp|dns]

*REST*

.. code-block:: console

  curl -u <user>:<password> http://<host>:<port>/profile/{udp, tcp, icmp}?target_id=<target id>\&profile={black, white, gray, default}

Example:

.. code-block:: console

  curl -u admin:admin http://localhost:8880/profile/udp?target_id=SCRBR1\&profile=default

Configure UDP/TCP/ICMP drop/accept
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

This applies to all ports/types.

*CLI*

.. code-block:: console

  > target ID profile white|gray|black|default [udp|tcp|icmp]
  > target ID profile white|gray|black|default [udp|tcp|icmp] drop [enable|disable]
  > target ID profile white|gray|black|default [udp|tcp|icmp] accept [enable|disable]

*REST*

.. code-block:: console

  curl -u <user>:<password> http://<host>:<port>/profile/{udp, tcp, icmp}/{accept, drop}?target_id=<target id>\&profile={black, white, gray, default}[\&action={enable, disable}]

Example:

.. code-block:: console

  curl -u admin:admin http://localhost:8880/profile/udp/drop?target_id=SCRBR1\&profile=default\&action=enable

Configure GRE Signaling drop/accept
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

This works when decapsulation is enabled only.

*CLI*

.. code-block:: console

  > target ID profile white|gray|black|default gre
  > target ID profile white|gray|black|default gre drop [enable|disable]
  > target ID profile white|gray|black|default gre accept [enable|disable]

*REST*

.. code-block:: console

  curl -u <user>:<password> http://<host>:<port>/profile/gre/{accept, drop}?target_id=<target id>\&profile={black, white, gray, default}[\&action={enable, disable}]

Example:

.. code-block:: console

  curl -u admin:admin http://localhost:8880/profile/gre/accept?target_id=SCRBR1\&profile=default\&action=enable

Configure SYN check engage mode
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Enable to force always on, disable to disable tcp check in any condition, threshold to enable tcp check on traffic threshold, auto to enable tcp check when an attack is automatically detected or thredhold is exceeded.

*CLI*

.. code-block:: console

  > target ID profile white|gray|black|default tcp syn check [disable|threshold|auto|enable]

*REST*

.. code-block:: console

  curl -u <user>:<password> http://<host>:<port>/profile/tcp/syn/check?target_id=<target id>\&profile={black, white, gray, default}[\&action=update\&value={disable, threshold, auto, enable}]

Example:

.. code-block:: console

  curl -u admin:admin http://localhost:8880/profile/tcp/syn/check?target_id=SCRBR1\&profile=default

.. code-block:: console

  curl -u admin:admin http://localhost:8880/profile/tcp/syn/check?target_id=SCRBR1\&profile=default\&action=update\&value=auto

Configure TCP traffic threshold
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Set the maximum expected TCP traffic rate to feed the detection algorithm (Mbit/s).

*CLI*

.. code-block:: console

  > target ID profile white|gray|black|default tcp threshold [MBITPS]

*REST*

.. code-block:: console

  curl -u <user>:<password> http://<host>:<port>/profile/tcp/threshold?target_id=<target id>\&profile={black, white, gray, default}[\&action=update\&value=<Mbit/s>]

Example:

.. code-block:: console

  curl -u admin:admin http://localhost:8880/profile/tcp/threshold?target_id=SCRBR1\&profile=default\&action=update\&value=1000

Configure SYN check method
~~~~~~~~~~~~~~~~~~~~~~~~~~

*CLI*

.. code-block:: console

  > target ID profile white|gray|black|default tcp syn check_method [rfc|proxy|disable]

*REST*

.. code-block:: console

  curl -u <user>:<password> http://<host>:<port>/profile/tcp/syn/check_method?target_id=<target id>\&profile={black, white, gray, default}[\&action=update\&value={rfc, proxy, bypass}]

Example:

.. code-block:: console

  curl -u admin:admin http://localhost:8880/profile/tcp/syn/check_method?target_id=SCRBR1\&profile=default

.. code-block:: console

  curl -u admin:admin http://localhost:8880/profile/tcp/syn/check_method?target_id=SCRBR1\&profile=default\&action=update\&value=rfc

Configure SYN RFC check method threshold
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Maximum expected new TCP connections per second) to feed the mitigation algorithm (syn/s).

*CLI*

.. code-block:: console

  > target ID profile white|gray|black|default tcp syn rfc threshold [PPS]

*REST*

.. code-block:: console

  curl -u <user>:<password> http://<host>:<port>/profile/tcp/syn/rfc/threshold?target_id=<target id>\&profile={black, white, gray, default}[\&action=update\&value=<pps>

Example:

.. code-block:: console

  curl -u admin:admin http://localhost:8880/profile/tcp/syn/rfc/threshold?target_id=SCRBR1\&profile=default

.. code-block:: console

  curl -u admin:admin http://localhost:8880/profile/tcp/syn/rfc/threshold?target_id=SCRBR1\&profile=default\&action=update\&value=100

Enable whitelisting of sessions only
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Enable session whitelisting instead of IPs on traffic verified by the TCP check.

*CLI*

.. code-block:: console

  > target ID profile white|gray|black|default tcp syn wl_session_only [enable|disable]

*REST*

.. code-block:: console

  curl -u <user>:<password> http://<host>:<port>/profile/tcp/syn/wl_session_only?target_id=<target id>\&profile={black, white, gray, default}[\&action={enable, disable}]

Example:

.. code-block:: console

  curl -u admin:admin http://localhost:8880/profile/tcp/syn/wl_session_only?target_id=SCRBR1\&profile=default\&action=enable

Auto-engage whitelisting of sessions only
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Configure whitelisting of sessions only instead of IPs on traffic verified by the TCP check to automatically engage on threshold
The maximum number of whitelisted IPs should be specified to trigger it.
The /tcp/syn/wl_session_only option is ignored when using this.

*CLI*

.. code-block:: console

  > target ID profile white|gray|black|default tcp syn wl_threshold [NUM]

*REST*

.. code-block:: console

  curl -u <user>:<password> http://<host>:<port>/profile/tcp/syn/wl_threshold?target_id=<target id>\&profile={black, white, gray, default}[\&action=update\&value=<whitelist size>]

Example:

.. code-block:: console

  curl -u admin:admin http://localhost:8880/profile/tcp/syn/wl_threshold?target_id=SCRBR1\&profile=default\&action=update\&value=10000

Configure SYN rate limiting
~~~~~~~~~~~~~~~~~~~~~~~~~~~

Limit per source or dest (pkts/s)

*CLI*

.. code-block:: console

  > target ID profile white|gray|black|default tcp syn rate src [PPS]
  > target ID profile white|gray|black|default tcp syn rate dst [PPS]

*REST*

.. code-block:: console

  curl -u <user>:<password> http://<host>:<port>/profile/tcp/syn/rate/{src, dst}?target_id=<target id>\&profile={black, white, gray, default}[\&action=update\&value=<pkts/s>]

Example:

.. code-block:: console

  curl -u admin:admin http://localhost:8880/profile/tcp/syn/rate/src?target_id=SCRBR1\&profile=default

.. code-block:: console

  curl -u admin:admin http://localhost:8880/profile/tcp/syn/rate/src?target_id=SCRBR1\&profile=default\&action=update\&value=20

Configure Auto-Blacklist
~~~~~~~~~~~~~~~~~~~~~~~~

Blacklist sources exceeding SYN rate for some time or not passing the TCP-Check.
It is recommended to also enable /attackers/autopurging.

*CLI*

.. code-block:: console

  > target ID profile white|gray|black|default tcp syn auto_blacklist [enable|disable]

*REST*

.. code-block:: console

  curl -u <user>:<password> http://<host>:<port>/profile/tcp/syn/auto_blacklist?target_id=<target id>\&profile={black, white, gray, default}[\&action={enable, disable}]

Example:

.. code-block:: console

  curl -u admin:admin http://localhost:8880/profile/tcp/syn/auto_blacklist?target_id=SCRBR1\&profile=default\&action=enable

Configure SYN-ACK rate limiting
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Limit per source or dest (pkts/s)

*CLI*

.. code-block:: console

  > target ID profile white|gray|black|default tcp synack rate src [PPS]
  > target ID profile white|gray|black|default tcp synack rate dst [PPS]

*REST*

.. code-block:: console

  curl -u <user>:<password> http://<host>:<port>/profile/tcp/synack/rate/{src, dst}?target_id=<target id>\&profile={black, white, gray, default}[\&action=update\&value=<pkts/s>]

Example:

.. code-block:: console

  curl -u admin:admin http://localhost:8880/profile/tcp/synack/rate/src?target_id=SCRBR1\&profile=default

.. code-block:: console

  curl -u admin:admin http://localhost:8880/profile/tcp/synack/rate/src?target_id=SCRBR1\&profile=default\&action=update\&value=20

Configure SYN-ACK session whitelisting
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

*CLI*

.. code-block:: console

  > target ID profile white|gray|black|default tcp synack wl_session [enable|disable]

*REST*

.. code-block:: console

  curl -u <user>:<password> http://<host>:<port>/profile/tcp/synack/wl_session?target_id=<target id>\&profile={black, white, gray, default}[\&action={enable, disable}]

Example:

.. code-block:: console

  curl -u admin:admin http://localhost:8880/profile/tcp/synack/wl_session?target_id=SCRBR1\&profile=default\&action=enable

Configure SYN-ACK TCP-Amplification protection
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

*CLI*

.. code-block:: console

  > target ID profile white|gray|black|default tcp synack tcp_amp_protection [enable|disable]

*REST*

.. code-block:: console

  curl -u <user>:<password> http://<host>:<port>/profile/tcp/synack/tcp_amp_protection?target_id=<target id>\&profile={black, white, gray, default}[\&action={enable, disable}]

Example:

.. code-block:: console

  curl -u admin:admin http://localhost:8880/profile/tcp/synack/tcp_amp_protection?target_id=SCRBR1\&profile=default\&action=enable

Drop TCP SYN with seq num 0
~~~~~~~~~~~~~~~~~~~~~~~~~~~

*CLI*

.. code-block:: console

  > 

*REST*
 
.. code-block:: console

  curl -u <user>:<password> http://<host>:<port>/profile/tcp/syn/noseqnum/drop?target_id=<target id>\&profile={black, white, gray, default}[\&action={enable, disable}]

Example:

.. code-block:: console

  curl -u admin:admin http://localhost:8880/profile/tcp/syn/noseqnum/drop?target_id=SCRBR1\&profile=default

.. code-block:: console

  curl -u admin:admin http://localhost:8880/profile/tcp/syn/noseqnum/drop?target_id=SCRBR1\&profile=default\&action=enable

Drop TCP SYN with no options
~~~~~~~~~~~~~~~~~~~~~~~~~~~~ 

*CLI*

.. code-block:: console

  > target ID profile white|gray|black|default tcp syn nooption drop [enable|disable]

*REST*

.. code-block:: console

  curl -u <user>:<password> http://<host>:<port>/profile/tcp/syn/nooption/drop?target_id=<target id>\&profile={black, white, gray, default}[\&action={enable, disable}]

Example:

.. code-block:: console

  curl -u admin:admin http://localhost:8880/profile/tcp/syn/nooption/drop?target_id=SCRBR1\&profile=default

.. code-block:: console

  curl -u admin:admin http://localhost:8880/profile/tcp/syn/nooption/drop?target_id=SCRBR1\&profile=default\&action=enable

Drop TCP SYN packets with payload
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~ 

*CLI*

.. code-block:: console

  > target ID profile white|gray|black|default tcp syn payload drop [enable|disable]

*REST*

.. code-block:: console

  curl -u <user>:<password> http://<host>:<port>/profile/tcp/syn/payload/drop?target_id=<target id>\&profile={black, white, gray, default}[\&action={enable, disable}]

Example:

.. code-block:: console

  curl -u admin:admin http://localhost:8880/profile/tcp/syn/payload/drop?target_id=SCRBR1\&profile=default

.. code-block:: console

  curl -u admin:admin http://localhost:8880/profile/tcp/syn/payload/drop?target_id=SCRBR1\&profile=default\&action=enable

Set drop/accept policy per UDP/TCP src/dst port 
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

  Note: in order to block UDP-based amplification attacks set source ports for dns, ntp, snmp, nb, ssdp, cg, qotd, bt, kad, qnp, sp.

*CLI*

.. code-block:: console

  > target ID profile white|gray|black|default [udp|tcp] [src|dst] PORT [drop|accept] [enable|disable]

*REST*

.. code-block:: console

  curl -u <user>:<password> http://<host>:<port>/profile/{udp, tcp}/{src, dst}/{port}/{accept, drop}?target_id=<target id>\&profile={black, white, gray, default}[\&action={enable, disable}]

Example:

.. code-block:: console

  curl -u admin:admin http://localhost:8880/profile/udp/src/53/drop?target_id=SCRBR1\&profile=default

.. code-block:: console

  curl -u admin:admin http://localhost:8880/profile/udp/src/53/drop?target_id=SCRBR1\&profile=default\&action=enable

Set min/max UDP payload length
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

*CLI*

.. code-block:: console

  > target ID profile white|gray|black|default udp payload min_len [BYTES]
  > target ID profile white|gray|black|default udp payload max_len [BYTES]

*REST*

.. code-block:: console

  curl -u <user>:<password> http://<host>:<port>/profile/udp/payload/{min, max}_len?target_id=<target id>\&profile={black, white, gray, default}[\&action=update\&value=<len>]

Example:

.. code-block:: console

  curl -u admin:admin http://localhost:8880/profile/udp/payload/min_len?target_id=SCRBR1\&profile=default

.. code-block:: console

  curl -u admin:admin http://localhost:8880/profile/udp/payload/min_len?target_id=SCRBR1\&profile=default\&action=update\&value=2

Drop UDP fragments
~~~~~~~~~~~~~~~~~~

*CLI*

.. code-block:: console

  > target ID profile white|gray|black|default udp fragment drop [enable|disable]

*REST*

.. code-block:: console

  curl -u <user>:<password> http://<host>:<port>/profile/udp/fragment/drop?target_id=<target id>\&profile={black, white, gray, default}[\&action={enable, disable}]

Example:

.. code-block:: console

  curl -u admin:admin http://localhost:8880/profile/udp/fragment/drop?target_id=SCRBR1\&profile=default

.. code-block:: console

  curl -u admin:admin http://localhost:8880/profile/udp/fragment/drop?target_id=SCRBR1\&profile=default\&action=enable

Set min/max UDP fragments payload length
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

*CLI*

.. code-block:: console

  > target ID profile white|gray|black|default udp fragment payload min_len [BYTES]
  > target ID profile white|gray|black|default udp fragment payload max_len [BYTES]

*REST*

.. code-block:: console

  curl -u <user>:<password> http://<host>:<port>/profile/udp/fragment/payload/{min, max}_len?target_id=<target id>\&profile={black, white, gray, default}[\&action=update\&value=<len>]

Example:

.. code-block:: console

  curl -u admin:admin http://localhost:8880/profile/udp/fragment/payload/min_len?target_id=SCRBR1\&profile=default

.. code-block:: console

  curl -u admin:admin http://localhost:8880/profile/udp/fragment/payload/min_len?target_id=SCRBR1\&profile=default\&action=update\&value=64

Drop UDP with checksum0
~~~~~~~~~~~~~~~~~~~~~~~

*CLI*

.. code-block:: console

  > target ID profile white|gray|black|default udp checksum0 drop [enable|disable]

*REST*

.. code-block:: console

  curl -u <user>:<password> http://<host>:<port>/profile/udp/checksum0/drop?target_id=<target id>\&profile={black, white, gray, default}[\&action={enable, disable}]

Example:

.. code-block:: console

  curl -u admin:admin http://localhost:8880/profile/udp/checksum0/drop?target_id=SCRBR1\&profile=default

.. code-block:: console

  curl -u admin:admin http://localhost:8880/profile/udp/checksum0/drop?target_id=SCRBR1\&profile=default\&action=enable

Configure UDP rate limiting
~~~~~~~~~~~~~~~~~~~~~~~~~~~

Limit all UDP traffic (pkts/s) per source or destination

*CLI*

.. code-block:: console

  > target ID profile white|gray|black|default udp rate src [PPS]
  > target ID profile white|gray|black|default udp rate dst [PPS]

*REST*

.. code-block:: console

  curl -u <user>:<password> http://<host>:<port>/profile/udp/rate/{src, dst}?target_id=<target id>\&profile={black, white, gray, default}[\&action=update\&value=<pkts/s>]

Example:

.. code-block:: console

  curl -u admin:admin http://localhost:8880/profile/udp/rate/src?target_id=SCRBR1\&profile=default

.. code-block:: console

  curl -u admin:admin http://localhost:8880/profile/udp/rate/src?target_id=SCRBR1\&profile=default\&action=update\&value=100

Set drop policy per ICMP type
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

*CLI*

.. code-block:: console

  > target ID profile white|gray|black|default icmp type TYPE drop [enable|disable]
  > target ID profile white|gray|black|default icmp type TYPE accept [enable|disable]

*REST*

.. code-block:: console

  curl -u <user>:<password> http://<host>:<port>/profile/icmp/type/<icmp type>/{accept, drop}?target_id=<target id>\&profile={black, white, gray, default}[\&action={enable, disable}]

Example:

.. code-block:: console

  curl -u admin:admin http://localhost:8880/profile/icmp/type/0/drop?target_id=SCRBR1\&profile=default

.. code-block:: console

  curl -u admin:admin http://localhost:8880/profile/icmp/type/0/drop?target_id=SCRBR1\&profile=default\&action=disable

Set drop policy per TTL values
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

*CLI*

.. code-block:: console

  > target ID profile white|gray|black|default ip
  > target ID profile white|gray|black|default ip ttl TTL drop [enable|disable]
  > target ID profile white|gray|black|default ip ttl TTL accept [enable|disable]

*REST*

.. code-block:: console

  curl -u <user>:<password> http://<host>:<port>/profile/ip/ttl/<ttl value>/{accept, drop}?target_id=<target id>\&profile={black, white, gray, default}[\&action={enable, disable}]

Example:

.. code-block:: console

  curl -u admin:admin http://localhost:8880/profile/ip/ttl?target_id=SCRBR1\&profile=default

.. code-block:: console

  curl -u admin:admin http://localhost:8880/profile/ip/ttl/24/drop?target_id=SCRBR1\&profile=default\&action=enable

Configure DNS check method
~~~~~~~~~~~~~~~~~~~~~~~~~~

*CLI*

.. code-block:: console

  > target ID profile white|gray|black|default dns request check_method [forcetcp|disable]

*REST*

.. code-block:: console

  curl -u <user>:<password> http://<host>:<port>/profile/dns/request/check_method?target_id=<target id>\&profile={black, white, gray, default}[\&action=update\&value={forcetcp, default}]

Example:

.. code-block:: console

  curl -u admin:admin http://localhost:8880/profile/dns?target_id=SCRBR1\&profile=default

.. code-block:: console

  curl -u admin:admin http://localhost:8880/profile/dns/request/check_method?target_id=SCRBR1\&profile=default\&action=update\&value=forcetcp

Configure DNS rate limiting
~~~~~~~~~~~~~~~~~~~~~~~~~~~

Limit DNS requests per source or transaction ID (pkts/s)

*CLI*

.. code-block:: console

  > target ID profile white|gray|black|default dns request rate src [PPS]
  > target ID profile white|gray|black|default dns request rate transaction_id [PPS]

*REST*

.. code-block:: console

  curl -u <user>:<password> http://<host>:<port>/profile/dns/request/rate/src?target_id=<target id>\&profile={black, white, gray, default}[\&action=update\&value=<pkts/s>]

Example:

.. code-block:: console

  curl -u admin:admin http://localhost:8880/profile/dns/request/rate/src?target_id=SCRBR1\&profile=default

.. code-block:: console

  curl -u admin:admin http://localhost:8880/profile/dns/request/rate/src?target_id=SCRBR1\&profile=default\&action=update\&value=20

Configure DNS traffic threshold
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

This is the maximum expected number of queries per second. 
This is used to feed the detection algorithm.
(packets/s)

*CLI*

.. code-block:: console

  > target ID profile white|gray|black|default dns request threshold [PPS]

*REST*

.. code-block:: console

  curl -u <user>:<password> http://<host>:<port>/profile/dns/request/threshold?target_id=<target id>\&profile={black, white, gray, default}[\&action=update\&value=<pps>]

Example:

.. code-block:: console

  curl -u admin:admin http://localhost:8880/profile/dns/request/threshold?target_id=SCRBR1\&profile=default\&action=update\&value=1000

Set drop policy per DNS request type
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

*CLI*

.. code-block:: console

  > target ID profile white|gray|black|default dns request type TYPE drop [enable|disable]

*REST*

.. code-block:: console

  curl -u <user>:<password> http://<host>:<port>/profile/dns/request/type/<dns query type>/drop?target_id=<target id>\&profile={black, white, gray, default}[\&action={enable, disable}]

Example:

.. code-block:: console

  curl -u admin:admin http://localhost:8880/profile/dns/request/type/255/drop?target_id=SCRBR1\&profile=default

.. code-block:: console

  curl -u admin:admin http://localhost:8880/profile/dns/request/type/255/drop?target_id=SCRBR1\&profile=default\&action=enable

Set max DNS subdomain length
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

*CLI*

.. code-block:: console

  > target ID profile white|gray|black|default dns request subdomain_max_len [CHARACTERS]

*REST*

.. code-block:: console

  curl -u <user>:<password> http://<host>:<port>/profile/dns/request/subdomain_max_len?target_id=<target id>\&profile={black, white, gray, default}[\&action=update\&value=<len>]

Example:

.. code-block:: console

  curl -u admin:admin http://localhost:8880/profile/dns/request/subdomain_max_len?target_id=SCRBR1\&profile=default

.. code-block:: console

  curl -u admin:admin http://localhost:8880/profile/dns/request/subdomain_max_len?target_id=SCRBR1\&profile=default\&action=update\&value=10

Hex/string pattern match
~~~~~~~~~~~~~~~~~~~~~~~~

Add <hex/string, offset> pattern to match (drop). Set "" as value to delete a pattern.
Note: 

 - 'payload+' represents beginning of L7 payload (end of L4 headers), it applies to tcp/udp packets only. 
 - when 'payload+' is not specified, 'offset' is considered from the beginning of the ethernet frame.
 - 'string' is case sensitive

*CLI*

.. code-block:: console

  > target ID profile white|gray|black|default pattern PATTERN drop [{hex, string},[payload+]{OFFSET, any},VALUE|-]

*REST*

.. code-block:: console

  curl -u <user>:<password> http://<host>:<port>/profile/pattern/<id>/drop?target_id=<target id>\&profile={black, white, gray, default}\&action=update\&value={hex, string},[payload+]{<offset>, any},<hex/string to match>

Example:

.. code-block:: console

  curl -u admin:admin http://localhost:8880/profile/pattern/1/drop?target_id=SCRBR1\&profile=default\&action=update\&value=hex,56,0954A03AC3320F

.. code-block:: console

  curl -u admin:admin http://localhost:8880/profile/pattern/2/drop?target_id=SCRBR1\&profile=default\&action=update\&value=string,payload+8,Hello

Read active patterns
~~~~~~~~~~~~~~~~~~~~

*CLI*

.. code-block:: console

  > target ID profile white|gray|black|default pattern

*REST*

.. code-block:: console

  curl -u <user>:<password> http://<host>:<port>/profile/pattern?target_id=<target id>\&profile={black, white, gray, default}

Example:

.. code-block:: console

  curl -u admin:admin http://localhost:8880/profile/pattern?target_id=SCRBR1\&profile=default

HTTP request field match
~~~~~~~~~~~~~~~~~~~~~~~~

Add HTTP request field to match (drop). Set "" as value to delete a field.
Note: 'label' is case sensitive, instead 'value' is compared ignoring the case.

*CLI*

.. code-block:: console

  > target ID profile white|gray|black|default http request field FIELD drop [LABEL,VALUE|-]

*REST*

.. code-block:: console

  curl -u <user>:<password> http://<host>:<port>/profile/http/request/field/<id>/drop?target_id=<target id>\&profile={black, white, gray, default}\&action=update\&value=<label>,<value>

Example:

.. code-block:: console

  curl -u admin:admin http://localhost:8880/profile/http/request/field/1/drop?target_id=SCRBR1\&profile=default\&action=update\&value=User-Agent,Bot

.. code-block:: console

  curl -u admin:admin http://localhost:8880/profile/http/request/field/1/drop?target_id=SCRBR1\&profile=white\&action=update\&value=User-Agent,Bot

Read active HTTP request fields
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

*CLI*

.. code-block:: console

  > target ID profile white|gray|black|default http request field

*REST*

.. code-block:: console

  curl -u <user>:<password> http://<host>:<port>/profile/http/request/field?target_id=<target id>\&profile={black, white, gray, default}

Example:
  
.. code-block:: console

  curl -u admin:admin http://localhost:8880/profile/http/request/field?target_id=SCRBR1\&profile=default

Add allowed HTTP hosts
~~~~~~~~~~~~~~~~~~~~~~

*CLI*

.. code-block:: console

  > target ID profile white|gray|black|default http request host ID pass [HOSTNAME|-]

*REST*

.. code-block:: console

  curl -u <user>:<password> http://<host>:<port>/profile/http/request/host/<id>/pass?target_id=<target id>\&profile={black, white, gray, default}\&action=update\&value=<hostname>

Example:
  
.. code-block:: console

  curl -u admin:admin http://localhost:8880/profile/http/request/host/1/pass?target_id=SCRBR1\&profile=default\&action=update\&value=example.com

Read active HTTP hosts
~~~~~~~~~~~~~~~~~~~~~~

*CLI*

.. code-block:: console

  > target ID profile white|gray|black|default http request host

*REST*

.. code-block:: console

  curl -u <user>:<password> http://<host>:<port>/profile/http/request/host?target_id=<target id>\&profile={black, white, gray, default}

Example:
  
.. code-block:: console

  curl -u admin:admin http://localhost:8880/profile/http/request/host?target_id=SCRBR1\&profile=default

Users Management
----------------

Read users list
~~~~~~~~~~~~~~~

*CLI*

.. code-block:: console

  > users

*REST*

.. code-block:: console

  curl -u <user>:<password> http://<host>:<port>/users?action=list

Example:

.. code-block:: console

  curl -u admin:admin http://localhost:8880/users?action=list

Add/del/update users
~~~~~~~~~~~~~~~~~~~~

*CLI*

.. code-block:: console

  > useradd NAME GROUP PASSWORD
  > usermod NAME GROUP [PASSWORD]
  > userdel NAME

*REST*

.. code-block:: console

  curl -u <user>:<password> -X POST https://<host>:<port>/users?action={add, del, update}\&username=<username>\&fullname=<full name>\&group={administrator} -d '{ "password" : "<password>" }'

Example:

.. code-block:: console

  curl -u admin:admin -X POST https://localhost:8880/users?action=add\&username=john\&fullname=John\&group=administrator -d '{ "password" : "temporarypassword" }'

.. code-block:: console

  curl -u admin:admin http://localhost:8880/users?action=del\&username=john


