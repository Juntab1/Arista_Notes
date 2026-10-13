# MLAG Notes

## Leaf 1 & 2 setup 

This is the base MLAG configuration for a pair of leaf switches. The key idea is to create a dedicated peer VLAN and peer link so the switches can synchronize state while keeping the traffic out of the normal VLAN trunking path.

- `vlan 4094` is the reserved MLAG peer VLAN.
- `Port-Channel100` is the peer link between the two switches.
- `trunk group MLAGPEER` removes the peer VLAN from normal VLAN allowed lists.

<div style="background-color:#1f1f1f; border:1px solid #3a3a3a; border-radius:8px; padding:12px; color:#e5e7eb; font-family:'Consolas','Monaco',monospace; line-height:1.6;">
<pre><code>enable
configure terminal

vlan 4094 <span style="color:#6ee7b7;"># 4094 is the usual VLAN used for MLAG</span>
   name MLAG_PEER
!
interface Ethernet1
   description MLAG_PEER_LINK
   channel-group 100 mode active
!
interface Port-Channel100
   description MLAG_PEER_LINK
   switchport mode trunk
   switchport trunk group MLAGPEER <span style="color:#6ee7b7;"># like an allow list for VLANs, but it is a global admin rule</span>
!
vlan 4094
   trunk group MLAGPEER <span style="color:#6ee7b7;"># strips it out of the normal allowed VLAN list, invisible to the rest of the switch</span>
!
interface Vlan4094
   ip address 10.255.255.1/30
!
mlag configuration
   domain-id MLAG-LAB
   local-interface Vlan4094
   peer-address 10.255.255.2
   peer-link Port-Channel100
!
end
</code></pre>
</div>

> Note: `vlan 4094` is the usual VLAN used for MLAG, and it should be kept separate from the regular user-facing VLANs.

## Spine 1

This step configures the spine as the Layer 3 uplink point for both MLAG leaf switches. Each spine interface is turned into a routed interface and given a /30 point-to-point IP address so the leaves can communicate before we add dynamic routing.

### 1) Connect to the spine
From the Ubuntu host:

```bash
docker exec -it clab-mlag-lab-spine1 Cli
```

### 2) Configure the spine interfaces
Inside EOS:

```eos
enable
configure terminal

interface Ethernet1
   description TO_LEAF1
   no switchport
   ip address 10.0.1.1/30
   no shutdown

interface Ethernet2
   description TO_LEAF2
   no switchport
   ip address 10.0.2.1/30
   no shutdown
end
```

This changes both interfaces from Layer 2 switchports into routed Layer 3 interfaces.

### 3) Configure each leaf toward spine1
Connect to each leaf and apply the matching address on the interface that faces the spine.

Leaf 1:

```bash
docker exec -it clab-mlag-lab-leaf1 Cli
```

```eos
enable
configure terminal

interface Ethernet2
   description TO_SPINE1
   no switchport
   ip address 10.0.1.2/30
   no shutdown
end
```

Leaf 2:

```bash
docker exec -it clab-mlag-lab-leaf2 Cli
```

```eos
enable
configure terminal

interface Ethernet2
   description TO_SPINE1
   no switchport
   ip address 10.0.2.2/30
   no shutdown
end
```

### 4) Verify connectivity
From each switch, confirm the interfaces are up:

```eos
show ip interface brief
show interfaces status
```

From the spine, test reachability to both leaves:

```eos
ping 10.0.1.2
ping 10.0.2.2
```

From leaf 1:

```eos
ping 10.0.1.1
```

From leaf 2:

```eos
ping 10.0.2.1
```

If the interfaces are up and the pings succeed, the spine-to-leaf Layer 3 links are working correctly.

> Important: This step creates the routed links, but it does not yet enable dynamic routing. After verifying connectivity, the next step is to configure OSPF across the spine and both leaves so they can exchange routes.

## Adding a server with bonding to the MLAG pair

The best way to make the server behave like a real end host in this lab is to connect it to both leaves with a Linux bond and to present a single logical uplink to the network. This gives you active/active redundancy, LACP negotiation, and an easy way to verify that MLAG is working as expected.

### 1) Update the topology
Add the server as a Linux node and connect both of its NICs to different leaves.

```yaml
topology:
  nodes:
    server1:
      kind: linux
      image: alpine:latest

    leaf1:
      kind: ceos

    leaf2:
      kind: ceos

  links:
    - endpoints: ["server1:eth1", "leaf1:eth5"]
    - endpoints: ["server1:eth2", "leaf2:eth5"]
```

This creates a dual-homed server: one link goes to leaf1 and the other goes to leaf2.

### 2) Configure the server-facing port-channels on both leaves
Each leaf should treat its server-facing port as part of the same port-channel so the server sees one logical uplink.

On leaf 1:

```eos
enable
configure terminal

interface Ethernet5
   description TO_SERVER1
   channel-group 11 mode active
!
interface Port-Channel11
   description SERVER1_BOND
   switchport mode trunk
   switchport trunk allowed vlan 10
   mlag 11
   no shutdown
end
```

On leaf 2:

```eos
enable
configure terminal

interface Ethernet5
   description TO_SERVER1
   channel-group 11 mode active
!
interface Port-Channel11
   description SERVER1_BOND
   switchport mode trunk
   switchport trunk allowed vlan 10
   mlag 11
   no shutdown
end
```

The important part here is that both leaves configure the same port-channel ID and the same VLAN, while MLAG keeps the channel synchronized across the pair.

### 3) Create the Linux bond on the server
Inside the server container:

```bash
docker exec -it clab-mlag-lab-server1 bash
```

Then create the bond:

```bash
modprobe bonding
ip link add bond0 type bond mode 802.3ad
ip link set eth1 master bond0
ip link set eth2 master bond0
ip link set eth1 up
ip link set eth2 up
ip link set bond0 up
ip addr add 10.0.10.10/24 dev bond0
ip route replace default via 10.0.10.1
```

This gives the server a single interface, `bond0`, while both physical NICs participate in the link aggregation. The bond uses 802.3ad (LACP), which matches the switch port-channel configuration.

> In a real lab, you would often also enable LACP timers, add `bond_lacp_rate fast`, and verify the bond state with `cat /proc/net/bonding/bond0`.

### 4) Verify the bond and the MLAG behavior
On either leaf, check that the port-channel is up and that LACP is learning the remote:

```eos
show port-channel summary
show lacp neighbor
show interfaces status
```

On the server, confirm the bond is active:

```bash
cat /proc/net/bonding/bond0
```

You should see both `eth1` and `eth2` as active members of the bond.

### 5) Test reachability
From the server:

```bash
ping -c 5 10.0.10.1
ping -c 5 10.0.1.1
ping -c 5 10.0.2.1
```

From the leaf or spine:

```eos
show ip arp
show mac address-table
ping 10.0.10.10
```

If the bond is healthy and the MLAG pair is behaving correctly, the server will reach the network through both links, and traffic will stay up even if one member of the bond fails.

> This is the cleanest lab pattern for a server behind MLAG: a single Linux bond on the host, paired with a port-channel on both leaves, and a trunked VLAN path for the server traffic.

> This is the point where the lab starts behaving like a real network: the switches carry the topology, and the server becomes the end host that validates the design.

Great! That confirms Alpine can see both network interfaces. Next, we’ll configure the switch-side MLAG port-channel and the Linux LACP bond so the host can use both links.
Step 4 — Configure the leaves
Run these commands in the Leaf1 EOS CLI:
enable
configure terminal

vlan 10
   name SERVERS
!
interface Ethernet3
   channel-group 10 mode active
!
interface Port-Channel10
   switchport mode access
   switchport access vlan 10
   mlag 10
!
end


Now run the same configuration on Leaf2:
enable
configure terminal

vlan 10
   name SERVERS
!
interface Ethernet3
   channel-group 10 mode active
!
interface Port-Channel10
   switchport mode access
   switchport access vlan 10
   mlag 10
!
end


This assumes Et3 is the interface connected to server1 on each leaf, as in the topology we added.
What these commands do:
- channel-group 10 mode active enables LACP on each physical switch port.
- Port-Channel10 represents the combined logical interface.
- mlag 10 pairs the downstream port-channel across both leaves.
- VLAN 10 provides the Layer 2 network for the server.
Step 5 — Configure Alpine's LACP bond
Inside the Alpine container (server1), first install the networking tools if you haven't already:
apk add --no-cache iproute2


Then configure the bond:
ip link add bond0 type bond mode 802.3ad
ip link set eth1 down
ip link set eth2 down
ip link set eth1 master bond0
ip link set eth2 master bond0
ip link set bond0 up
ip link set eth1 up
ip link set eth2 up


Give the host an IP address in VLAN 10:
ip addr add 192.168.10.10/24 dev bond0


Check the result:
ip -br link
ip -br addr
cat /proc/net/bonding/bond0


The last command should show the bond's status and LACP information. If bond creation fails, stop and share the error; the container may need additional kernel bonding support.
Step 6 — Verify MLAG
Run these commands on both leaves:
show mlag
show mlag interfaces
show port-channel summary


You should see Port-Channel10 associated with MLAG ID 10, and LACP should establish the links. The bond's two links may not both forward traffic at the same time in every failure or hashing scenario—that's normal for LACP.
One important note: the host IP is only useful for communication with other devices in VLAN 10; you won't necessarily be able to ping it from the spine until you configure VLAN routing.

## Quick lab control

For now, I am stopping the lab with:

```bash
sudo containerlab stop --topo topology.clab.yml
```

When I come back, I can restart it with:

```bash
sudo containerlab start --topo topology.clab.yml
```