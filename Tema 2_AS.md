# Reconocimiento activo
### Networking

 #ping
```
ping -c <COUNT> <MACHINE_IP>
```
- [[ICMP]] echo request / ICMP echo reply

 #traceroute
```
traceroute <MACHINE_IP>
```
- Mide TTL incremental e ICMP de vuelta
- UNIX: UDP / Windows: ICMP
- También opcional TCP SYN traceroute

### Escaneo

**ARP e ICMP**
```
arp-scan -I <INTERFACE> --localnet
```

#nmap
- **TCP:**
``` 
sudo nmap -sS -p- [-n -Pn --min-rate <N>] <TARGET>
``` 
- **UDP:** 
```
sudo nmap -sU --top-ports <N> [-n -Pn] <TARGET>
```

