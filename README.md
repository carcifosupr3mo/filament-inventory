Filament Inventory
> Inventory and print-job tracking for a Bambu Lab P1S with AMS. Stock, consumption and print history are read directly from the printer's data streams instead of being logged by hand.
![Status](https://img.shields.io/badge/status-in%20production-2ea043?style=flat-square)
![Python](https://img.shields.io/badge/Python-3.12-3776AB?style=flat-square&logo=python&logoColor=white)
![FastAPI](https://img.shields.io/badge/FastAPI-009688?style=flat-square&logo=fastapi&logoColor=white)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL_16-4169E1?style=flat-square&logo=postgresql&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-2496ED?style=flat-square&logo=docker&logoColor=white)
![License](https://img.shields.io/badge/license-MIT-6e7681?style=flat-square)
Obiettivo
Tenere l'inventario dei filamenti a mano non funziona: dopo qualche stampa non si sa più quanti grammi restano su una bobina, quale bobina è stata usata per quale pezzo, né quando conviene riordinare.
Questo sistema legge i dati direttamente dalla stampante — stato via MQTT, file di progetto via FTPS — e ricostruisce da solo consumo, storico delle stampe e giacenze, senza che l'operatore debba registrare nulla.
Funzionalità
Inventario bobine con marca, materiale, colore, peso residuo, fornitore e posizione di stoccaggio
Tracciamento automatico dei lavori di stampa con macchina a stati `IDLE → PREPARE → RUNNING → FINISH/FAILED`
Calcolo del consumo a cascata su quattro livelli: grammi esatti estratti dal 3MF → stima proporzionale → caricamento manuale come fallback
Risoluzione slot → bobina basata su materiale e colore, non sull'indice dello slot AMS
Statistiche su consumo per materiale, per colore e nel tempo, con ogni KPI navigabile verso la vista filtrata corrispondente
Import/export CSV dell'inventario
Notifiche Telegram a fine stampa e su soglia di scorta
Rotazione automatica dei backup del database
Tecnologie
Ambito	Scelta
Linguaggio	Python 3.12
Framework	FastAPI + uvicorn
ORM	SQLAlchemy 2.0 (async)
Database	PostgreSQL 16
Reverse proxy	Caddy
Runtime	Docker
Frontend	Vanilla JS + Tabulator.js
Protocolli stampante	MQTT (8883), FTPS (990)
Architettura
```
                    ┌──────────────────────────┐
                    │  Bambu Lab P1S + AMS     │
                    └────┬─────────────────┬───┘
                MQTT 8883│                 │FTPS 990
                         v                 v
          ┌──────────────────────┐  ┌──────────────────┐
          │  MQTT supervisor     │  │  FTPS client     │
          │  (delta merging)     │  │  (3MF parsing)   │
          └──────────┬───────────┘  └────────┬─────────┘
                     └────────┬──────────────┘
                              v
                    ┌───────────────────┐
       browser ───> │ Caddy ──> FastAPI │ ──> PostgreSQL 16
                    └───────────────────┘
```
Tre container Docker su un LXC Proxmox: PostgreSQL, applicazione, reverse proxy.
Il supervisore MQTT mantiene una connessione persistente con la stampante e fonde gli aggiornamenti parziali (delta) sullo stato completo, perché la stampante invia lo stato integrale solo su richiesta esplicita.
Requisiti
Docker e Docker Compose
Bambu Lab P1S raggiungibile sulla rete locale
Access code e numero di serie della stampante
Installazione
```bash
git clone https://github.com/carcifosupr3mo/filament-inventory.git
cd filament-inventory

cp .env.example .env
# compila .env, poi:

docker compose up -d
```
L'interfaccia risponde sulla porta configurata in `.env`.
Configurazione
Variabile	Descrizione	Esempio
`PRINTER_HOST`	Indirizzo della stampante sulla LAN	`10.0.0.50`
`PRINTER_SERIAL`	Numero di serie	`YOUR_PRINTER_SERIAL`
`PRINTER_ACCESS_CODE`	Access code mostrato sul display	`YOUR_ACCESS_CODE`
`POSTGRES_USER`	Utente del database	`YOUR_DB_USER`
`POSTGRES_PASSWORD`	Password del database	`YOUR_DB_PASSWORD`
`POSTGRES_DB`	Nome del database	`filament`
`TELEGRAM_BOT_TOKEN`	Token del bot per le notifiche (opzionale)	`YOUR_BOT_TOKEN`
`TELEGRAM_CHAT_ID`	Chat di destinazione (opzionale)	`YOUR_CHAT_ID`
Il file `.env` non va mai committato: è già escluso dal `.gitignore`.
Utilizzo
Apri l'interfaccia nel browser. All'accensione della stampante il supervisore MQTT si collega da solo e la dashboard inizia a popolarsi; non serve avviare nulla a mano.
Al primo avvio il database viene precaricato con marche, fornitori (inclusi quelli svizzeri) e posizioni di stoccaggio, modificabili dall'interfaccia. I campi combinati permettono di creare al volo un record che non esiste ancora, con deduplicazione che ignora maiuscole e minuscole.
Struttura del progetto
```
filament-inventory/
├── app/
│   ├── models/          modelli SQLAlchemy
│   ├── routers/         endpoint FastAPI
│   ├── services/        MQTT, FTPS, statistiche, backup
│   └── static/          frontend
├── tests/
├── docker-compose.yml
├── .env.example
└── README.md
```
Servizi e porte
Servizio	Porta	Esposto
Caddy	80	solo LAN
FastAPI	interna	no
PostgreSQL	interna	no
MQTT verso stampante	8883 (uscita)	—
FTPS verso stampante	990 (uscita)	—
Sicurezza
Nessuna autenticazione, per scelta progettuale. Il sistema è accessibile solo dalla LAN, con un unico operatore e nessun port forwarding verso Internet. Aggiungere un layer di login avrebbe introdotto complessità senza aumentare la sicurezza reale in questo scenario. Non esporre questo servizio su Internet senza prima aggiungere autenticazione.
Credenziali della stampante e del database solo in variabili d'ambiente, mai nel codice.
Nessuna porta applicativa raggiungibile dall'esterno della rete locale.
Backup del database con rotazione automatica.
Test
```bash
pytest
```
Sei suite, 178 asserzioni.
Note tecniche
Alcune cose scoperte durante lo sviluppo, non documentate altrove:
Nei file 3MF di Bambu Lab i dati del filamento sono tag `<metadata key=... value=...>`, non attributi XML. I campi `prediction` e `weight` all'inizio erano interpretati male proprio per questo.
L'indice del filamento nel 3MF non corrisponde all'indice dello slot AMS. L'abbinamento va fatto su tipo di materiale e colore. È il risultato di uno script diagnostico eseguito contro la stampante reale.
MQTT funziona anche con la stampante in cloud mode; `pushall` risponde con 63 campi.
`connected: false` nello stato della stampante indica che è spenta, non un errore di configurazione.
La serializzazione dei job richiede eager loading delle relazioni prima del commit, altrimenti il lazy loading fallisce fuori dalla sessione.
I confronti tra datetime richiedono coerenza tra naive e aware.
JSONB ha bisogno di una variante compatibile con SQLite per la portabilità.
Problemi conosciuti
Il calcolo del consumo è esatto solo quando il 3MF è recuperabile via FTPS; negli altri casi si ricade sulla stima proporzionale.
Testato solo su P1S con AMS Classic.
Sviluppi futuri
[ ] Supporto a più stampanti
[ ] Previsione di riordino basata sul consumo storico
[ ] Esportazione dello storico in PDF
Autore
Nathan Pollini — @carcifosupr3mo
Licenza
MIT — vedi LICENSE.
