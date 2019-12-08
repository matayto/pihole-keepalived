# pihole-keepalived

Simple failover configurations for a multi-pihole infrastructure. Tested on Raspbian Buster Lite, 2019-09

The keepalived setup assumes that a non-responsive TCP query against port 53 on the peer indicates the peer is down. At present, this failover configuration does not address issues at the DNS application layer. 

TODO: Improve via DNS healthcheck script using UDP MISC_CHECK. Limited testing indicates this has CPU/perf implications, and further testing is required.

## Requirements

1. Two pihole instances installed and running.
2. Static IP assignment on both running pihole instances.
3. Only one pihole is configured to provide DHCP leases, or another device on the network is providing DHCP, such as a router or firewall.

## Static IP Assignment

Plan for three static IP addresses in your network.

1. The main DNS VIP, maintained by keepalived
2. Primary pihole IP
3. Secondary pihole IP

On your primary pihole, using your text editor of choice, ensure the following in /etc/dhcpcd.conf, adjusting the IP address:

```
interface eth0
static ip_address=192.168.0.101/24
static routers=192.168.0.1
static domain_name_servers=192.168.0.100 192.168.0.101 192.168.0.102
```

Save the file and either restart or run `systemctl restart dhcpcd`. 

Repeat the above steps on your secondary pihole, adjusting IP addresses as necessary.


## Custom DHCP config

If one of the pihole is providing DHCP leases, to ensure that DHCP clients can properly utilize the keepalived DNS VIP, set the following in `/etc/dnsmasq.d/99-custom.conf`, creating the file if necessary:

```
dhcp-option=6,192.168.1.100,192.168.1.101,192.168.1.102
```

Restart dnsmasq via `sudo pihole restartdns`.


## Keepalived setup, primary pihole

1. Install keepalived.

```
sudo apt-get update && sudo apt-get install keepalived
```

2. Create the keepalived configuration at /etc/keepalived/keepalived.conf below.

```
vrrp_instance pihole {
        interface eth0
        state BACKUP
        virtual_router_id 100
        priority 100
        authentication {
                auth_type PASS
                auth_pass s0me_random_p@ssw0rd
        }
        virtual_ipaddress {
                192.168.0.100/24 dev eth0
        }
}

# Check for TCP
virtual_server 192.168.0.100 53 {
  delay_loop 6
  lb_algo wlc
  protocol TCP

  real_server 192.168.0.101 53 {
    weight 100
    TCP_CHECK {
      connect_timeout 6
    }
  }

  real_server 192.168.0.102 53 {
    weight 100
    TCP_CHECK {
      connect_timeout 6
    }
  }
}
```

3. Start and enable keepalived

```
systemctl --now enable keepalived
```

4. Validate primary pihole

Ensure no errors in keepalived journal messages 

```
journalctl -u keepalived
```

Validate the new VIP 192.168.0.100 VIP exists on eth0

```
ip a s eth0
```

Attempt to resolve DNS against the VIP

```
dig @192.168.0.100 duckduckgo.com
dig +notcp @192.168.0.100 duckduckgo.com
```

## Keepalived Setup, secondary pihole

On 192.168.0.102, perform steps 1 (install keepalived) and 2 (configure keepalived), using the following configuration. Take note of the different: `priority` values. The router ID references the last octet of the VIP and needs to be internally consistent between all peers. The priority on the secondary must be _lower_ than the primary. 

```
vrrp_instance pihole {
        interface eth0
        state BACKUP
        virtual_router_id 100
        priority 99
        authentication {
                auth_type PASS
                auth_pass s0me_random_p@ssw0rd
        }
        virtual_ipaddress {
                192.168.0.100/24 dev eth0
        }
}

# Check for TCP
virtual_server 192.168.0.100 53 {
  delay_loop 6
  lb_algo wlc
  protocol TCP

  real_server 192.168.0.101 53 {
    weight 100
    TCP_CHECK {
      connect_timeout 6
    }
  }

  real_server 192.168.0.102 53 {
    weight 100
    TCP_CHECK {
      connect_timeout 6
    }
  }
}
```

Repeat steps 3 and 4 from the primary keepalived config above, keeping in mind that the VIP should _not_ appear on the secondary pihole. 

Test failover by shutting down the primary pihole and validating that the keepalived VIP appears on the secondary.
