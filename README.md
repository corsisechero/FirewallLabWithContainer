# FirewallLabWithContainer

# Docker Network Setup e Configurazione Firewall

## Versione del Docker Compose
```yaml
version: '3.9'
services:
  firewall:
    image: ubuntu:latest
    container_name: firewall
    privileged: true
    command: bash -c "sysctl -w net.ipv4.ip_forward=1 && tail -f /dev/null"
    networks:
      internal_net:
        ipv4_address: 192.168.20.254
      external_net:
        ipv4_address: 192.168.21.254
      dmz_net:
        ipv4_address: 192.168.22.254

  client1:
    image: alpine:latest
    container_name: client1
    privileged: true
    command: tail -f /dev/null
    networks:
      internal_net:
        ipv4_address: 192.168.20.2

  client2:
    image: alpine:latest
    container_name: client2
    privileged: true
    command: tail -f /dev/null
    networks:
      external_net:
        ipv4_address: 192.168.21.2

  webserver:
    image: nginx:latest
    container_name: webserver
    privileged: true
    networks:
      dmz_net:
        ipv4_address: 192.168.22.2

networks:
  internal_net:
    driver: bridge
    ipam:
      config:
        - subnet: 192.168.20.0/24
  external_net:
    driver: bridge
    ipam:
      config:
        - subnet: 192.168.21.0/24
  dmz_net:
    driver: bridge
    ipam:
      config:
        - subnet: 192.168.22.0/24
```

### Configurazione delle Route per Client e Webserver

#### Per `client1`:
```bash
docker exec -it client1 bin/sh
ip route del default
ip route add default via 192.168.20.254
```

#### Per `client2`:
```bash
docker exec -it client2 bin/sh
ip route del default
ip route add default via 192.168.21.254
```

#### Per `webserver`:
Il comando `ip` potrebbe non essere disponibile, quindi si consiglia di installarlo. Per il momento, testa la configurazione solo su `client1` e `client2`.

---

## Gestire il Firewall con iptables

### 1. Accedi al Container `firewall`
Se non sei già dentro, accedi al container firewall:
```bash
docker exec -it firewall bash
```

### 2. Installa iptables
Una volta dentro, esegui i seguenti comandi per installare iptables:
```bash
apt update
apt install iptables
```

### 3. Verifica le Regole
Per verificare le regole attive, esegui:
```bash
iptables -L -v -n
```

---

## Bloccare e Sbloccare il Ping

### 1. Aggiungere una Regola per Bloccare il Ping da `client1` a `client2`

Accedi al container `firewall` e aggiungi una regola per bloccare il ping (ICMP) da `client1` (IP: 192.168.20.2) a `client2` (IP: 192.168.21.2):
```bash
iptables -A FORWARD -s 192.168.20.2 -d 192.168.21.2 -p icmp -j DROP
```

### 2. Verifica che il Ping sia Bloccato
Accedi al container `client1` e prova a fare un ping a `client2`:
```bash
docker exec -it client1 bin/sh
ping 192.168.21.2
```
Dovresti vedere che il ping non riceve risposta, poiché è stato bloccato dalla regola.

### 3. Rimuovere la Regola per Consentire il Ping
Per rimuovere la regola e permettere nuovamente il ping, esegui il seguente comando:
```bash
iptables -D FORWARD -s 192.168.20.2 -d 192.168.21.2 -p icmp -j DROP
```

### 4. Verifica che il Ping Funzioni
Dopo aver rimosso la regola, prova di nuovo a fare un ping da `client1` a `client2`:
```bash
ping 192.168.21.2
```
Questa volta, il ping dovrebbe funzionare e riceverai una risposta.

### 5. Verifica le Regole `iptables`
Per controllare che la regola sia stata effettivamente aggiunta o rimossa, esegui:
```bash
iptables -L -v -n
```

---

## Riepilogo

- **Per bloccare il ping**: Usa il comando:
  ```bash
  iptables -A FORWARD -s 192.168.20.2 -d 192.168.21.2 -p icmp -j DROP
  ```

- **Per rimuovere il blocco e consentire il ping**: Usa il comando:
  ```bash
  iptables -D FORWARD -s 192.168.20.2 -d 192.168.21.2 -p icmp -j DROP
  ```

---
```
