# MLAG Notes

## Leaf 1 setup

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

## Quick lab control

For now, I am stopping the lab with:

```bash
sudo containerlab stop --topo topology.clab.yml
```

When I come back, I can restart it with:

```bash
sudo containerlab start --topo topology.clab.yml
```