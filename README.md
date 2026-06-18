# Repository di Supporto - Esame Cybersecurity 2026

## INDICE DEGLI ESERCIZI
1. [Esercizio 1: Configura sudo affinche un utente possa eseguire solo un comando specifico (es nmap)](#1-restrizioni-comandi)
2. [Esercizio 2: Configura sudo affinche un utente possa eseguire solo un comando specifico ma senza un parametro (es si puo eseguire nmap ma non nmap -p)](#2-restrizioni-comandi-2)
3. [Esercizio 3: Ricerca File con SGID attivo](#3-ricerca-file-con-sgid-attivo)
4. [Esercizio 4: Unit File Systemd (Backdoor Netcat)](#4-unit-file-systemd-backdoor-netcat)
5. [Esercizio 5: Configurazione Modulo PAM (Complessità Password)](#5-configurazione-modulo-pam-complessita-password)
6. [Esercizio 6: Hardening Bootloader GRUB (Password e Modifica Parametri)](#6-hardening-bootloader-grub)
7. [Esercizio 7: Hardening Permessi /var/log (Solo Root)](#7-hardening-permessi-varlog)
8. [Esercizio 8: Sudoers per Lettura Log (Privilegi Minimi CAT)](#8-sudoers-per-lettura-log-cat)
9. [Esercizio 9: Monitoraggio File Descriptor Attivi con LSOF](#9-monitoraggio-file-descriptor-attivi-con-lsof)
10. [Esercizio 10: Privilege Escalation tramite Docker (Gruppo Docker)](#10-privilege-escalation-tramite-docker)
11. [Esercizio 11: Ricerca File Scrivibili da Chiunque (World-Writable) in /home](#11-ricerca-file-scrivibili-da-chiunque-world-writable-in-home)

---
## 1. Restrizioni Comandi
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
## 2. Restrizioni Comandi 2
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
