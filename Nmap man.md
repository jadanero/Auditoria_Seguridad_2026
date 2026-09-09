**Manual Command**:
``` 

```

```
nmap [Scan Type(s)] [Options] {target specification}
```

**Default TCP SYN scan:**
```
nmap -sS <target>
```

**Default UDP SYN scan:**
```
nmap -sU <target>
```

**Scan common ports**:
```
nmap -F <target>
```

**OS and service version detection:** 
```
nmap -A <target>
```

**Specific port scan:** 
```
nmap -p 80,443 <target>
``` 
	or
```
nmap -p 1-1000 <target>
```

