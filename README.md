1. Aim:
To design and simulate a basic Local Area Network in Cisco Packet Tracer using 3 PCs, a switch, and to study the working of the Address Resolution Protocol (ARP) by observing, in real time, how a host discovers the MAC address of another host on the same network before any data can actually be delivered.

2. Problem Statement:
A typical industrial floor has PLCs, sensors, and operator stations all wired into the same segment, and all of them think purely in terms of IP addresses when deciding who to send data to. The Ethernet hardware underneath doesn't care about IP at all, though — frames get delivered using MAC addresses, full stop. So a controller can know exactly which IP it needs to reach for a particular sensor and still have zero way of knowing which physical port or NIC that sensor actually sits behind. One fix people reach for is just hard-coding every device's MAC address into every other device, but that falls apart fast on a real plant floor where hardware gets swapped, replaced under warranty, or re-IP'd after a firmware update, and now half your static table is wrong until someone notices. ARP gets around the manual-table problem by letting a device ask the question out loud — broadcast it to everyone on the wire — and whoever owns that IP just answers back, so the mapping gets built automatically instead of typed in by hand. This matters a lot more in an industrial context than it sounds like it should, because if that lookup stalls or comes back wrong, a controller can't reach the actuator it's supposed to be driving, and that's not a minor inconvenience on a production line. The three-PC build in this repo is meant to make that resolution step something you can actually watch happen — broadcast going out, one host answering, the cache filling in — instead of something you just take on faith from a textbook diagram. It's also worth keeping in mind while watching it that ARP has no authentication built in at all, so the same broadcast-and-trust mechanism that makes it convenient is also why ARP spoofing exists as an attack in the first place.

3. Scope of the Solution:
This stays deliberately small: three PCs and one switch, all sitting on 192.168.1.0/24, nothing routed anywhere. The point is to isolate just the ARP behaviour — broadcast out, switch floods it, one host answers, cache fills in — without dragging in routing, inter-VLAN ARP, proxy ARP, or gratuitous ARP, each of which is really its own separate lab. Stay inside that boundary and what you're watching is the same resolution step that any industrial LAN depends on whenever a PC, sensor, or PLC needs to reach another device on the same wire.

4. Architecture:
Nothing fancy here — three end devices, each with one static IP, all home-run to a single switch over copper straight-through cable. A plain star, which is really all you need for what this lab is trying to show.
No default gateway is configured on any PC, because every device lives on the same subnet — there's nothing to route, so the resolution problem stays purely a Layer 2/Layer 3 boundary issue, which is same  what ARP  is try to handle.

5. Required Components:
 PC0, PC1, PC2 (PC-PT)
•	Generic end-device model in Packet Tracer.
•	These are the actual hosts issuing and receiving ARP requests/replies — every observation in this lab happens from one of their command prompts.
 Switch0 (Cisco 2960 series)
•	Fixed-configuration Layer 2 access switch.
•	Forwards frames by MAC address, but critically, floods any frame whose destination MAC it hasn't learned yet — which is exactly what happens to a broadcast ARP request, and that flooding behaviour is what lets every other host on the segment see and evaluate the request.
 Copper Straight-Through Cable
•	Physical/Layer 1 link between PC NIC and switch port.
•	Connects two different device types (PC and switch use different pin assignments).
 Static IP Addressing (same /24)
•	Manually assigned IPv4 addresses on each PC.
•	ARP is only relevant within a single broadcast domain — every PC needed to be on the same subnet for the resolution process to even apply, instead of being handed off to a gateway.
 Command Prompt (Desktop Tab)
•	Built-in CLI on each PT PC.
•	This is where ping, arp -a, and arp -d are actually executed and where the timing/loss statistics that hint at ARP delay become visible.
 Simulation Mode + Event List
•	Packet Tracer's packet-by-packet replay tool.
•	Lets the broadcast-flood-unicast-reply sequence be watched frame by frame instead of taken on faith — this is the part that actually proves what ARP is doing under the hood.
 ARP Cache (arp -a / arp -d)
•	Dynamic IP-to-MAC mapping table on each host.
•	This is the artifact ARP exists to build; clearing it before each test (arp -d) is what forces a fresh resolution cycle to actually happen so it can be observed.

6. Software / Device Library Specification:
•	Everything here was pulled from Cisco Packet Tracer's built-in device library, no external software involved:
•	End Devices category → generic PC-PT: the standard workstation model in Packet Tracer. Each one exposes a Desktop tab with IP Configuration, a Command Prompt, and a few other simulated applications — all of it accessible without any extra setup.
•	Network Devices category → Switches → Cisco Catalyst 2960-24TT: a fixed-configuration Layer 2 switch, simulated running its own internal IOS image. Note: the original brief calls for an 8-port switch — if your instructor wants port-count to match exactly, swap this for the 2960-8TC model from the same Switches list; the ARP behaviour demonstrated is identical either way since it has nothing to do with port count.
•	Connections category → Copper Straight-Through: used for every PC-to-switch link. Packet Tracer auto-validates the cable type per link — a green triangle on both ends means the connection and interface speeds/duplex negotiated correctly.
•	No additional libraries, packages, or external tools are needed; the entire simulation runs natively inside Packet Tracer.

7. Procedure:
•	Open Cisco Packet Tracer and start a new, blank project file.
•	From the End Devices category in the bottom-left device shelf, drag three generic PCs onto the canvas. Leave their default names (PC0, PC1, PC2).
•	From Network Devices → Switches, drag one switch onto the canvas (Switch0).
•	Select Connections → Copper Straight-Through and connect each PC's FastEthernet0 interface to a separate Fast Ethernet port on Switch0. Wait for the link indicators to turn solid green on both ends of every cable before moving on.
•	Click PC0 → Desktop tab → IP Configuration. Select Static and enter:
•	IPv4 Address: 192.168.1.10
•	Subnet Mask: 255.255.255.0
•	Default Gateway: left blank (0.0.0.0) — not needed on a single flat subnet
•	Repeat step 5 for PC1 (192.168.1.20) and PC2 (192.168.1.30), same subnet mask, no gateway.
•	Close the IP Configuration windows. At this point the topology is fully addressed and cabled.
•	Switch the workspace from Realtime to Simulation mode using the toggle in the bottom-right corner.
•	Open the Simulation Panel → Edit Filters, and restrict visible events to ARP only (click Show All/None then re-enable ARP) — this keeps the Event List from filling up with unrelated traffic.
•	Open PC0 → Desktop → Command Prompt.
•	Run arp -d to wipe out any existing ARP cache entry, so the next ping is guaranteed to trigger a fresh resolution.
•	Run ping 192.168.1.20 to ping PC1.
•	Note that the first echo request comes back as Request timed out — PC0 doesn't have PC1's MAC address yet, so that first attempt is consumed entirely by the ARP exchange instead of an actual ICMP reply.
•	Switch over to the Simulation tab and step through the Event List frame by frame: PC0 sends a broadcast ARP request, Switch0 floods it out every other port, PC2 receives and silently discards it (the requested IP doesn't match its own), and PC1 replies directly back to PC0 with a unicast ARP reply.
•	Watch the remaining ping replies succeed normally now that PC0 has cached PC1's MAC address. Final ping statistics typically read Sent = 4, Received = 3, Lost = 1 (25% loss) — the one loss being the ARP-delayed first packet, not a real connectivity failure.
•	Run arp -a on PC0 and confirm a new dynamic entry now exists, mapping 192.168.1.20 to PC1's MAC address.
•	Run arp -d again to clear the cache, then ping 192.168.1.30 to repeat the entire process against PC2.
•	Step through the Event List again and confirm the same broadcast/flood/unicast-reply pattern occurs, just now with PC1 as the host that silently drops the frame.
•	Run arp -a again and confirm a second dynamic entry now exists for 192.168.1.30, mapped to PC2's MAC address.

8. Configuration Details:
•	IP Address Assignment
Device	IP Address	Subnet Mask
PC0	  192.168.1.10	255.255.255.0
PC1	  192.168.1.20	255.255.255.0
PC2	  192.168.1.30	255.255.255.0

9. Observations and Results:
•	First ping to a fresh IP always drops exactly one packet — Sent = 4, Received = 3, Lost = 1 (25% loss) both times — and that loss lines up exactly with the ARP exchange, not an actual connectivity issue.
•	Switch0 floods the broadcast to every port except the one it came in on, no matter which specific host the request was meant for. PC2 dropping the first request and PC1 dropping the second confirms the switch isn't doing any filtering on broadcasts — it can't, since broadcasts have no learned port mapping to forward against.
•	After the first cycle, arp -a on PC0 shows 192.168.1.20 → 0040.0B15.47DB. After the second cycle, it shows 192.168.1.30 → 00D0.58D1.B5E6. One new entry per cycle, matching exactly whichever host was just pinged — nothing pre-populated, nothing left over from before the arp -d.
•	Once that entry exists, every later ping to the same address skips the broadcast step completely — round-trip times drop and stay consistent, which is the clearest visible sign the cache is actually being used rather than just sitting there unused.

10. Conclusion:
Most networking material covers ARP in a sentence or two and moves on. Building this out in Packet Tracer turns that sentence into something with an actual timeline — a packet that gets lost, a switch flooding ports it didn't need to, one PC ignoring a request that wasn't addressed to it, and a cache entry that didn't exist a second earlier. Both ping cycles in this repo ended the same way: one timed-out packet, one new dynamic entry, clean replies after that. Seeing that pattern repeat exactly on the second host (PC2) rather than being a fluke of the first one is really the point of running it twice — it confirms this is the actual mechanism at work, not a one-off simulation quirk.


