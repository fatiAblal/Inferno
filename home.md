# INFERNO SVELATO: UN VIAGGIO ATTRAVERSO LE FASI DELL’ATTACCO

## INTRODUZIONE
Il report fornisce un'analisi dettagliata di una demo in cui è stato simulato un attacco alla macchina virtuale "Inferno" scaricata da Vulnhub Labs. L'obiettivo della dimostrazione era di illustrare praticamente le tecniche di penetration testing e le vulnerabilità comuni nella cybersecurity. "Inferno" è una macchina virtuale progettata per l'apprendimento e la pratica del penetration testing, offrendo un ambiente simulato per sviluppare competenze nel rilevamento e nell'exploit di vulnerabilità informatiche. Dal punto di vista pratico si richiedeva di trovare le due hash keys, una associata all'utente 'user' (contenuta nel file local.txt) e l'altra all'utente 'root' (contenuta nel file proof.txt), presenti nella macchina al fine di acquisire i privilegi di amministratore ('root'). Per eseguire la demo, sono state utilizzate due macchine virtuali: "Kali Linux" e "Inferno", configurate per creare un ambiente di test idoneo.

## RECONNAISSANCE
Ho iniziato individuando su quale indirizzo IP si trovasse la macchina “Inferno”. Dopo di che, cercando tale indirizzo sul browser è apparsa la seguente pagina web statica:

![Pagina statica ottenuta](images/paginaStatica.png)

Ho dunque proseguito raccogliendo i dati preliminari sulla macchina per identificare potenziali punti di ingresso. Le attività principali includevano:

- **Nmap scan**: in particolare ho usato queste due versioni del comando:
  - `nmap 10.0.2.11`: per rilevare gli host, le porte aperte e i servizi in esecuzione sulla macchina target. Risultato: una lunga lista di open ports. Tuttavia, molte di queste sono rabbit holes, ovvero dei falsi positivi. Quindi per ovviare a questo problema è stata eseguita una variante di nmaps:
  - `nmaps -sCV 10.0.2.11 -vv`: per rilevare informazioni aggiuntive come dettagli sull’OS e potenziali vulnerabilità permettendo così l’identificazione dei servizi d’interesse per questa simulazione. Risultato: ci sono due porte effettivamente aperte e sono evidenziate nella figura sottostante:

![Porte realmente aperte](images/porteAperte.png)

- **Gobuster**: uno strumento di brute force utilizzato per scoprire file e directory nascosti su un server web. Ho utilizzato Gobuster con il comando:
  - `gobuster dir -u 10.0.2.11 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt`: specifica il percorso della wordlist da utilizzare per la scansione. La wordlist contiene un elenco di nomi di directory e file comuni che Gobuster proverà a trovare sul server web target.
  Risultato: l'esistenza di una directory denominata ‘/inferno’ potrebbe indicare un sito web alternativo non è immediatamente visibile o linkato dalle pagine principali. Potrebbe quindi contenere file sensibili, moduli di login, o altre risorse che possono essere sfruttate durante un penetration test. È dunque un punto di ingresso potenziale che merita ulteriori indagini.

## INITIAL ACCESS
Tentando di accedere all'indirizzo 10.0.2.11/inferno nel browser, è comparsa una finestra di login che richiedeva l'inserimento di username e password di tipologia Basic Auth identificando così un possibile punto di ingresso nel server web.

![Finestra di login ottenuta](images/finestraLogin.png)

# CREDENTIAL ACCESS
Per ottenere le credenziali necessarie per accedere alla finestra di login sulla macchina, ho utilizzato:

- **Hydra**: è uno strumento che esegue attacchi di forza bruta sulle credenziali di accesso, testando combinazioni di nomi utente e password per ottenere accesso non autorizzato a sistemi protetti da autenticazione. Per condurre questo tipo di attacco, ho utilizzato la wordlist 'rockyou.txt', nota per contenere milioni di credenziali comuni e spesso utilizzate. Il comando utilizzato è stato il seguente:
  - `hydra -l admin -P /usr/share/wordlists/rockyou.txt 10.0.2.11 http-get /inferno`
  Risultato: username=admin    password=dante1

# FOOTHOLD
In questa fase, ho constatato che, utilizzando le credenziali ottenute, sono riuscita ad accedere alla finestra di login precedentemente visualizzata. Tuttavia, dopo aver inserito le credenziali e confermato l'accesso, è emerso un'altra finestra di login che richiedeva nuovamente l'autenticazione. Questa situazione rappresenta una vulnerabilità poiché le credenziali vengono riutilizzate, aumentando il rischio di accessi non autorizzati.

Successivamente, ho esplorato la possibilità di modificare file .php per ottenere un maggiore controllo sul sistema. Tuttavia, a causa delle restrizioni sui permessi, non ho potuto apportare modifiche ai file identificati. In alternativa, ho analizzato la struttura di uno dei file .php per comprendere meglio l'ambiente di sviluppo utilizzato.

Dopo aver scoperto che il sistema utilizza Codiad, un ambiente di sviluppo integrato (IDE) basato sul web, ho proseguito con una ricerca online per individuare exploit già esistenti per Codiad. Ho trovato un exploit su GitHub all'indirizzo [https://github.com/WangYihang/Codiad-Remote-Code-Execute-Exploit](https://github.com/WangYihang/Codiad-Remote-Code-Execute-Exploit). In seguito, ho scaricato il file exploit.py e l'ho memorizzato sul desktop della mia macchina Kali.



