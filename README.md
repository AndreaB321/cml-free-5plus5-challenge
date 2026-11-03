# CML Free 5 + 5 challenge: DHCP relay, DNS, TACACS+ and port security

A Cisco Modeling Labs (CML) 2.11 lab for CCNA preparation. It tests DHCP relay, DNS, a TACACS+ login and one Layer 2 security feature (port security), all checked from a host, and it fits within the CML Free limits of 5 network devices and 5 auxiliary nodes.

The lab is in [`cml-free-5plus5-challenge.yaml`](cml-free-5plus5-challenge.yaml). Import it from the CML dashboard with **Import Lab**. The step-by-step checks are also stored in the lab notes, so you can follow them inside CML.

## Topology

```
PC1 ─┐                                       ┌─ DNS-DHCP (dnsmasq) 10.20.20.10
     SW1 (IOL-L2) ── R1 (IOL-XE) ── SRV-SW ──┤
PC2 ─┘  VLAN 10       DHCP relay             └─ TACACS (tac_plus)   10.20.20.20
     10.10.10.0/24                     10.20.20.0/24
```

| Bucket | Nodes | Count |
|---|---|---|
| Network | R1 (IOL-XE), SW1 (IOL-L2) | 2 of 5 |
| Auxiliary | PC1, PC2 (Alpine), DNS-DHCP (dnsmasq), TACACS (tac_plus) | 4 of 5 |
| Not counted | SRV-SW (Unmanaged Switch) | - |

On CML 2.10 Free, with its single limit of 5 nodes, this lab has 6 counted nodes and does not fit. It fits only with the 5 + 5 rule introduced in CML 2.11.

## Addressing

| Segment | Subnet | Gateway |
|---|---|---|
| USERS, VLAN 10 | 10.10.10.0/24 | R1 Et0/0, 10.10.10.1 (DHCP relay to 10.20.20.10) |
| SERVERS | 10.20.20.0/24 | R1 Et0/1, 10.20.20.1 |

- SW1 SVI VLAN 10: 10.10.10.2
- DNS-DHCP (dnsmasq): 10.20.20.10, DHCP scope 10.10.10.100-150, domain `lab.local`
- TACACS (tac_plus): 10.20.20.20

## Lab credentials

These are lab-only values, built into the topology file. Do not reuse them anywhere else.

| Where | Username / password |
|---|---|
| PC1, PC2 | cisco / cisco |
| R1 over SSH, from TACACS+ | tacadmin / admin123 (privilege 15), tacoper / oper123 (privilege 1) |
| R1 local fallback | localadmin / LocalOnly1 (accepted only when TACACS+ does not answer) |
| R1 console | no login, enable secret `cisco` |
| TACACS+ shared key | Cisco123! |

Start all nodes. Wait about a minute after R1 boots: an EEM applet generates the RSA keys for SSH.

## The checks, all from a host

### 1. DHCP relay (from PC1)

```sh
sudo udhcpc -i eth0 -x hostname:pc1
ip addr show eth0
ip route
cat /etc/resolv.conf
```

PC1 should get an address between 10.10.10.100 and .150, a default route via 10.10.10.1 and nameserver 10.20.20.10. The DHCP server is on another subnet, so a lease proves that the relay works. Repeat on PC2 with `-x hostname:pc2`.

Optional, on R1: `debug ip dhcp server packet` shows the relayed packets.

### 2. DNS (from PC1)

```sh
nslookup r1.lab.local        # A record
nslookup 10.10.10.1          # PTR record
nslookup www.lab.local       # CNAME to r1.lab.local
nslookup -type=mx lab.local  # MX to mail.lab.local
nslookup pc2.lab.local       # registered by dnsmasq when PC2 took its lease
```

### 3. TACACS+ login (from PC1)

```sh
ssh tacadmin@r1.lab.local     # prompt R1#  (privilege 15 from TACACS+)
ssh tacoper@r1.lab.local      # prompt R1>  (privilege 1 from TACACS+)
ssh localadmin@r1.lab.local   # rejected: the server answers and does not know this user
```

The negative test shows that authentication really goes through the server. Stop the TACACS node in CML and try again:

```sh
ssh localadmin@r1.lab.local   # accepted after a short timeout: fallback to the local database
ssh tacadmin@r1.lab.local     # rejected: this user exists only on the server
```

Start the TACACS node again. On R1, `show tacacs` shows the counters. On the TACACS console, `cat /var/log/tacplus-acct.log` shows the accounting records.

### 4. Layer 2 security: port security (from PC1 and PC2)

Both host ports on SW1 allow one MAC address (sticky), with violation mode shutdown and automatic recovery after 30 seconds.

On PC1, write down the current MAC address, then change it:

```sh
ip link show eth0
sudo ip link set dev eth0 down
sudo ip link set dev eth0 address 02:00:00:00:66:66
sudo ip link set dev eth0 up
ping -c 5 10.10.10.1          # fails
```

On PC2, `ping -c 5 10.10.10.1` still works, so only the PC1 port was shut down. Optional, on SW1: `show port-security interface Ethernet0/1` and `show interfaces status err-disabled`.

Restore the original MAC address on PC1 and wait about 30 seconds for errdisable recovery:

```sh
sudo ip link set dev eth0 down
sudo ip link set dev eth0 address <original MAC>
sudo ip link set dev eth0 up
ping -c 5 10.10.10.1          # works again
```

## Why port security and not DHCP snooping

The CML Free reference platform has IOL-L2 as its only switch image. Cisco lists port security among the features tested on IOL-L2. The Cisco Community has reports, one of them on CML 2.10, of DHCP snooping on IOL-L2 that shows as operational but leaves the binding table empty and does not block a rogue server. A check that can give a false positive is not good enough for a challenge that must be verified from a host.

## What has been tested

- **DNS:** the dnsmasq configuration was run locally with the same dnsmasq version as the CML container (2.91). The syntax check passes and every record (A, PTR, CNAME, MX) answers as expected.
- **DHCP:** the dnsmasq DHCP syntax is valid. A relayed lease was not tested outside CML.
- **Lab file:** imported into CML. All 7 nodes, 6 links and the node configurations were stored as expected.
- **Not yet run end to end:** the lab has not been started, so the four checks above have not yet been proven on a running lab.

## Open points for CML Free users

- Since CML 2.10, the `dnsmasq` and `tacplus` containers ship on a separate reference platform ISO. It is not confirmed that this ISO is available to CML Free users.
- It is not confirmed that these two containers count as auxiliary nodes under the 2.11 Free limits.

## Spare budget

Three network slots and one auxiliary slot are still free. Two ideas: add R2 and move the servers one routed hop away (OSPF plus a relay across the core), or move SW1 management to TACACS+ as well.

Share your own 5 + 5 design, or an improvement to this one, in the comments of the LinkedIn post or as an issue in this repository.

## License

MIT, see [LICENSE](LICENSE).
