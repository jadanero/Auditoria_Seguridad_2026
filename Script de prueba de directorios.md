```
#!/bin/bash

echo "[*] Lanzando búsqueda masiva en paralelo (50 hilos)..."

# seq genera los números y xargs -P 50 los procesa en paralelo
seq -w 0 99999 | xargs -P 50 -I {} bash -c '
    id="{}"
    url="https://content-prod-live.cert.starbucks.com/binary/v2/asset/143-${id}.pdf"
    
    if wget --spider -q "$url"; then
        echo "[+] ¡ENCONTRADO! Descargando: $url"
        wget -q "$url"
    fi
```

