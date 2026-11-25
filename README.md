# Your MikroTik could be attacking others: a DNS reflection case study

[Versión en español](README.es.md)

## Summary

While analyzing NetFlow data at an internet service provider, I found that hundreds of customer routers were being used as **DNS reflectors** in amplification attacks against third parties. I found the problem and coordinated the response.

| Metric | Value |
|---|---|
| Customer routers used as reflectors | 232 |
| Share that were MikroTik | **82%** |
| Victims hit in 24 hours | **1,400+** (mostly game servers) |
| Amplification | ~20x |
| Peak attack traffic | ~1 Gbps |

## What happened

- Customer routers had **DNS "allow remote requests" enabled** and **UDP/TCP port 53 open on the WAN**, turning them into open resolvers.
- Attackers sent small DNS queries with the **victim's spoofed IP** as source. Each router answered the victim with a response ~20 times larger.
- About 20 Mbps of queries turned into hundreds of Mbps of attack traffic, with peaks near 1 Gbps.
- The traffic was visible in NetFlow from the first day of collection, two weeks before it was formally detected.
- **No customer had reported a problem.** Their internet "worked fine", while their upload link was being used to attack others.

## How it was found

1. NetFlow showed constant outbound UDP/53 traffic from customer addresses.
2. Correlating flows with the inventory (Grafana + NetBox) linked each source to a customer service, and each destination to a victim and a time window.
3. A script generated an incident report per event.
4. Grouping by device vendor showed that 82% of the reflectors were MikroTik routers with the same configuration mistake.

## Why blocking the victim is not enough

Blackholing the destination (RTBH) stops a single attack, but the attacker simply moves to another victim through the same reflectors. The real fix is at the source: closing the open resolvers.

**Lesson for automation:** before automating RTBH, define a whitelist and an automatic expiry for every block. Otherwise you end up with a growing list of blocked prefixes nobody reviews.

## How to fix it (MikroTik)

Disable remote DNS requests if the router does not need to serve DNS to other hosts:

```
/ip dns set allow-remote-requests=no
```

If it must serve the LAN, block DNS from the WAN:

```
/ip firewall filter
add chain=input protocol=udp dst-port=53 in-interface-list=WAN action=drop comment="Block DNS from WAN"
add chain=input protocol=tcp dst-port=53 in-interface-list=WAN action=drop comment="Block DNS from WAN"
```

Also keep RouterOS updated and do not expose Winbox, the API or the web interface to the internet. In 2021 the **Meris** botnet took control of around 250,000 exposed MikroTik routers through an old Winbox vulnerability.

## Open challenge

The technical fix is simple. The hard part is **communication with customers**: most of them do not know their router is involved, and acting on their service requires a clear policy.

## Notes

All graphs and data are anonymized. No IP addresses, configurations of the provider's network, topology or customer data are included.
