# Progetto di WebRTC: Creazione di una istanza completa di Matrix

# Lo scopo di questo progetto

L’obiettivo di questo progetto è di realizzare un sistema di comunicazione tramite la piattaforma [Matrix](https://matrix.org/), in una maniera semplicemente replicabile da qualsiasi utente.

## Perchè usare Matrix?

**Matrix** è un progetto open source di instant messaging, capace di fornire diverse funzionalità: supporta numerosi protocolli sia per la messaggistica testuale che lo streaming audiovisivo (tra cui **VoIP/WebRTC**), è supportato da numerosi client (che siano web o nativi) e server e supporta la **crittografia End-to-End**.

Per tal motivo, in questo progetto sarà configurato un server Matrix tale che:

- **Sia facilmente selfhostabile da qualsiasi utente**: Matrix è un progetto decentralizzato, possiamo quindi creare un’istanza privata oppure federarla, ossia intercomunicante con le altre istanze presenti online.
- **Sia facilmente accessibile da remoto**: usando un reverse proxy è possibile associare un nome di dominio all’istanza attraverso il quale poter accedere da un client qualsiasi.
- **Possa comunicare con servizi di messagistica terzi**: Matrix supporta un gran numero di [**bridges**](https://matrix.org/ecosystem/bridges/) capaci di farci interagire con utenti di piattaforme proprietarie come **Whatsapp** e **Telegram** (risolvendo uno dei più grandi scogli che spesso una ha nel passaggio verso una nuova piattaforma).

Per garantire la portabilità del progetto, dati i suoi numerosi componenti interdipendenti, il tutto sarà realizzato puramente tramite l’uso di **docker** (e definito attraverso un singolo file compose) e la modifica apposita di specifici file di configurazione a cui i container saranno mappati.

---

# Creazione dell’Istanza

## Requisiti

- Un sistema con installato Linux.
- [Docker Community Edition.](https://docs.docker.com/engine/install/)
- [Il plugin Docker Compose.](https://docs.docker.com/compose/install/linux/)
- Un dominio: in questo report viene usato un dominio Cloudflare ma è facilmente sostituire le dipendenze per traefik con quelle per altri domini (anche gratuiti, come [DuckDNS](https://www.duckdns.org/)).

L’infrastruttura è stata testata su un PC con Debian 12 Bookworm ma l’infrastruttura è compatibile con WIndows tramite WSL2. 

## Preparativi - Configurazione per l’accesso esterno

### Port forwarding per l’accesso remoto

Per fare ciò, dobbiamo in primis aprire delle porte sul router:

- **80** per HTTP.
- **443** per HTTPS.
- **3478** per il server relay TURN.
- **5349** per abilitare TLS su TURN.
- Un range (es. **49160-49200**) per instanziare videochiamate simultanee.

### Proxy tramite Traefik

**Traefik** è un application proxy che, nel nostro caso d’uso, ci permette di definire direttamente tutte le regole di rete basilari, definendo i **certificati TLS** (che saranno aggiornati automaticamente tramite certbot e letsencrypt).

Definiamo quindi una rete interna per docker e il container per Traefik:

```yaml
networks:  
 matrix:

services:
  traefik:    
  image: traefik:latest    
  container_name: traefik    
  networks:
    - matrix
  command:
    - --providers.docker=true      
    - --providers.docker.exposedByDefault=false      
    - --accessLog=true      
    - --api.insecure=true      
    - --entryPoints.web.address=:80      
    - --entryPoints.websecure.address=:443      
    - --entryPoints.web.forwardedHeaders.trustedIPs=173.245.48.0/20,103.21.244.0/22,103.22.200.0/22,103.31.4.0/22,141.101.64.0/18,108.162.192.0/18,190.93.240.0/20,188.114.96.0/20,197.234.240.0/22,198.41.128.0/17,162.158.0.0/15,104.16.0.0/13,104.24.0.0/14,172.64.0.0/13,131.0.72.0/22      
    - --entryPoints.websecure.forwardedHeaders.trustedIPs=173.245.48.0/20,103.21.244.0/22,103.22.200.0/22,103.31.4.0/22,141.101.64.0/18,108.162.192.0/18,190.93.240.0/20,188.114.96.0/20,197.234.240.0/22,198.41.128.0/17,162.158.0.0/15,104.16.0.0/13,104.24.0.0/14,172.64.0.0/13,131.0.72.0/22      
    - --entryPoints.web.http.redirections.entryPoint.to=websecure      
    - --entryPoints.web.http.redirections.entryPoint.scheme=https      
    - --certificatesResolvers.letsencrypt.acme.dnsChallenge=true      
    - --certificatesResolvers.letsencrypt.acme.dnsChallenge.provider=cloudflare      
    - --certificatesResolvers.letsencrypt.acme.email=${INSERIRE_EMAIL}      
    - --certificatesResolvers.letsencrypt.acme.storage=/etc/traefik/acme/acme.json      
    - --entryPoints.websecure.http.tls.certResolver=letsencrypt      
    - --entryPoints.websecure.http.tls.domains[0].main=${NOME_DOMINIO}      
    - --entryPoints.websecure.http.tls.domains[0].sans=*.${NOME_DOMINIO}      
    - --serversTransport.insecureSkipVerify=true      
    - --global.sendAnonymousUsage=false    
    volumes:
		- '/var/run/docker.sock:/var/run/docker.sock:ro'      
      	- './data/traefik/acme:/etc/traefik/acme'
    restart: always    
    ports:      
    - "80:80"      
    - "443:443"      
    environment:      
    - CF_API_EMAIL=${INSERIRE_EMAIL}      
    - CF_API_KEY=${INSERIRE_CHIAVE_API}
```

## Fase 1: Server Matrix con Synapse

Ora che abbiamo effettuato tutti gli step preliminari, passiamo alla configurazione del server Matrix.

Data la natura open source di Matrix, esistono differenti server: in questo caso useremo **Synapse**. 

### Creazione di database Postgres

Inanzitutto, è necessario un database: creiamo quindi un’istanza di **Postgres**:

```yaml
postgres:    
  container_name: 'postgres'    
  image: "postgres:11.4"        
  restart: "unless-stopped"    
  networks:      
      - matrix    
  environment:      
  POSTGRES_PASSWORD: "admin"    
  volumes:      
      - "./matrix/postgres/data:/var/lib/postgresql/data"
```

Dobbiamo però anche definire il database interno, in modo che sia accessibile dal server Synapse.

Per tal motivo serve accedere al DB per configurarlo: avviamo il container tramite `docker compose create postgres` e accedendoci tramite `docker exec -it /bin/sh` per eseguire `psql -U postgres` e infine creare ruoli e databases.

```sql
-- Creazione del ruolo nel DB --
CREATE ROLE synapse;
ALTER ROLE synapse WITH PASSWORD 'INSERIRE_PASSWORD';
ALTER ROLE synapse WITH LOGIN;

-- Database dell'istanza Matrix --
CREATE DATABASE synapse ENCODING 'UTF8' LC_COLLATE='C' LC_CTYPE='C' template=template0 OWNER synapse;
GRANT ALL PRIVILEGES ON DATABASE synapse TO synapse;

-- Database apposito per il bridge whatsapp --
CREATE DATABASE whatsapp ENCODING 'UTF8' LC_COLLATE='C' LC_CTYPE='C' template=template0 OWNER synapse;
GRANT ALL PRIVILEGES ON DATABASE whatsapp TO synapse;

-- Database apposito per il bridge telegram --
CREATE DATABASE telegram ENCODING 'UTF8' LC_COLLATE='C' LC_CTYPE='C' template=template0 OWNER synapse;
GRANT ALL PRIVILEGES ON DATABASE telegram TO synapse;
```

### Configurazione di Synapse

Passiamo adesso all’effettivo server Synapse, partendo dal container:

```yaml
  synapse:    
  container_name: 'synapse'    
  image: "matrixdotorg/synapse:latest"    
  restart: "unless-stopped"    
  networks:      
      - matrix          
  environment:      
      SYNAPSE_CONFIG_DIR: "/data"      
      SYNAPSE_CONFIG_PATH: "/data/homeserver.yaml"      
  volumes:      
      - "./matrix/synapse:/data"      
      - "./matrix/bridges:/bridges:rw"    
  labels:      
      - "traefik.enable=true"      
      - "traefik.http.services.synapse.loadbalancer.server.port=8008"      
      - "traefik.http.routers.synapse.rule=Host(`synapse.${NOME_DOMINIO}`)"      
      - "traefik.http.routers.synapse.entrypoints=websecure"      
      - "traefik.http.routers.synapse.tls.certresolver=letsencrypt"
```

Si definisce un mapping a una cartella di configurazione: al suo interno serve definire la configurazione specifica in `./matrix/synapse/homeserver.yaml` in tal modo:

```yaml
web_client_location: https://$CLIENT_NAME.$MY_DOMAIN/ 
public_baseurl: https://synapse.$MY_DOMAIN/
enable_registration: true
enable_registration_without_verification: true

listeners:  
- port: 8008    
	tls: false    
	type: http    
	x_forwarded: true    
	resources:      
	- names: [client, federation]        
      compress: false

retention:  
	enabled: true

federation_ip_range_blacklist:
  - '127.0.0.0/8'
  - '10.0.0.0/8'  
  - '172.16.0.0/12'  
  - '192.168.0.0/16'  
  - '100.64.0.0/10'  
  - '169.254.0.0/16'  
  - '::1/128'  
  - 'fe80::/64'  
  - 'fc00::/7'
  
database:  
	name: psycopg2  
	args:    
      user: synapse    
      password: INSERIRE_PASSWORD    
      database: synapse    
      host: matrix-postgres       
      cp_min: 5    
      cp_max: 10
      
# app_service_config_files:  
	# /bridges/whatsapp/registration.yaml
	# /bridges/telegram/registration.yaml
	
# turn_uris: [ "turn:turn.NOME_DOMINIO?transport=udp", "turn:turn.NOME_DOMINIO?transport=tcp", "turns:turn.NOME_DOMINIO?transport=udp", "turns:turn.NOME_DOMINIO?transport=tcp" ]
# turn_shared_secret: "INSERIRE_SECRET"
# turn_user_lifetime: 86400000
# turn_allow_guests: True
```

Nel caso si volesse avere un’istanza defederata basterà aggiungere nella sezione`federation_ip_range_blacklist` 0.0.0.0/0.

Fatto ciò, avviamo il container con `docker compose create synapse`: possiamo testare se è tutto operativo.

![immagine.png](report/immagine.png)

### Abilitazione alla federazione con Nginx

Adesso impostiamo il Reverse Proxy **Nginx**, anche qui si definisce il container…

```yaml
  nginx:    
  container_name: 'nginx'    
  image: "nginx:latest"    
  restart: "unless-stopped"    
  networks:      
  - matrix     
  volumes:      
  - "./matrix/nginx/matrix.conf:/etc/nginx/conf.d/matrix.conf"      
  - "./matrix/nginx/www:/var/www/"    
  labels:      
  - "traefik.enable=true"      
  - "traefik.http.services.matrix.loadbalancer.server.port=80"      
  - "traefik.http.routers.matrix.rule=Host(`matrix.${NOME_DOMINIO}`)"      
  - "traefik.http.routers.matrix.entrypoints=websecure"      
  - "traefik.http.routers.matrix.tls.certresolver=letsencrypt"
```

…e configuriamo internalmente in `./matrix/nginx/matrix.conf` :

```yaml

server {  
	listen         80 default_server;  
	server_name    matrix.NOME_DOMINIO;
	
 location /_matrix {
   proxy_pass http://matrix-synapse:8008;
   proxy_set_header X-Forwarded-For $remote_addr;
   client_max_body_size 128m;  
 }

  location /.well-known/matrix/ {    
  root /var/www/;    
  default_type application/json;    
  add_header Access-Control-Allow-Origin  *;  
  }
}
```

Infine, possiamo avviare con `docker compose create nginx` e testare la federazione andando su https://federationtester.matrix.org/#matrix.NOME_DOMINIO: se ogni campo esce come “Success” allora la federazione è attiva.

![immagine.png](report/immagine%201.png)

## Fase 4: Abilitazione delle videochiamate

Matrix supporta diverse opzioni per chiamate e videochiamate: in questo caso proviamo a usare una soluzione stand-alone, **TURN**, configurabile sia sulla stessa macchina dove vi è il server Synapse che su una separata.

### TURN

**Trasversal Using Relays around NAT** (TURN) permette agli endpoint sotto NAT (che necessitano di conoscere i rispettivi indirizzi server-reflexive per poter iniziare una comunicazione reciproca) della videochiamata di comunicare tramite un relay (TURN server), in grado di far associare ad ognuno un **relay address**, un indirizzo di trasporto pubblico utilizzato per inoltrare tra NAT e il relay.  

### Configurazione di CoTURN

Creiamo quindi un’istanza

```yaml
  coturn:    
  image: "instrumentisto/coturn:latest"    
  container_name: 'coturn'    
  restart: "unless-stopped"    
  network_mode: "host"    
  volumes:      
      - "./matrix/coturn:/etc/coturn"
```

Servirà anche definire il file di configurazione in `./data/coturn/turnserver.conf`, dopo aver creato un secret per l’autenticazione:

```yaml
server-name=turn.NOME_DOMINIO
realm=turn.NOME_DOMINIO
external-ip=turn.NOME_DOMINIO/IP_LOCALE
listening-ip=0.0.0.0
relay-ip=0.0.0.0

listening-port=3478
tls-listening-port=5349
min-port=49160
max-port=49200

verbose

use-auth-secret
static-auth-secret=INSERIRE_SECRET
```

Infine servirà aggiungere il supporto a TURN, con annesso lo stesso secret definito in turnserver.conf: decommentiamo quindi la sezione apposita in `./matrix/synapse/homeserver.yaml`.

```yaml
turn_uris: [ "turn:turn.NOME_DOMINIO?transport=udp", "turn:turn.NOME_DOMINIO?transport=tcp", "turns:turn.NOME_DOMINIO?transport=udp", "turns:turn.NOME_DOMINIO?transport=tcp" ]
turn_shared_secret: "INSERIRE_SECRET"
turn_user_lifetime: 86400000
turn_allow_guests: True
```

Possiamo riavviare Synapse con `docker compose restart synapse` e avviare il server TURN con `docker compose create coturn`.

## Fase 5: Bridging

Il server Matrix è operativo: ora possiamo aggiungere il supporto ai bridges: per fare ciò usiamo delle immagini fornite dal progetto **mautrix-bridges**.

### Whatsapp

Definiamo un container con il bridge mautrix apposito

```yaml
whatsapp:    
	container_name: whatsapp    
	image: dock.mau.dev/mautrix/whatsapp    
	restart: unless-stopped    
	networks:      
      - matrix    
	volumes:    
      - ./matrix/bridges/whatsapp:/data
```

e lo eseguiamo una sola volta, con `docker compose create whatsapp`, in modo da generare automaticamente il file `./matrix/bridges/whatsapp/config.yaml`.

Prendiamo tale file e sostituiamo questi parametri:

```yaml
homeserver:
	address: https://matrix.NOME_DOMINIO
	domain: matrix.NOME_DOMINIO

appservice:
	address: http://whatsapp:29318
	hostname: 0.0.0.0

database:
	uri: postgres://synapse:INSERIRE_PASSWORD@postgres/whatsapp?sslmode=disable
```

Avviamo una seconda volta il container, si creerà adesso un file `./matrix/bridges/whatsapp/registration.yaml`.

### Telegram

Come prima cosa, è necessario creare un’API Telegram tramite https://my.telegram.org/apps e ottenere le chiavi API (`api_id` e `api_hash`).

Ripetiamo il processo fatto con Whatsapp: 

```yaml
  telegram:    
  container_name: telegram    
  image: dock.mau.dev/mautrix/telegram    
  restart: unless-stopped    
  networks:      
      - matrix    
  volumes:    
      - ./matrix/bridges/telegram:/data
```

anche qui si avvia una volta con `docker compose create telegram` e modifichiamo il file `./matrix/bridges/telegram/config.yaml` allo stesso modo, includendo stavolta anche i codici API di Telegram:

```yaml
homeserver:
	address: https://matrix.NOME_DOMINIO
	domain: matrix.NOME_DOMINIO

appservice:
	address: http://telegram:29317
	hostname: 0.0.0.0

database:
	uri: postgres://synapse:INSERIRE_PASSWORD@postgres/whatsapp?sslmode=disable
	
telegram:    
	api_id: API_ID    
	api_hash: API_HASH
```

E con il secondo avvio del container avremo `./matrix/bridges/telegram/registration.yaml`.

### Abilitazione dei Bridges

Aggiungiamo adesso i due file registration alla configurazione Synapse editando `./matrix/synapse/homeserver.yaml` e decommentando la sezione apposita.

```yaml
app_service_config_files:
	-  /bridges/whatsapp/registration.yaml
	-  /bridges/telegram/registration.yaml
```

Infine si riavvia Synapse con `docker compose restart synapse` e avremo finito di configurare l’infrastruttura Matrix.

---

# Accesso tramite Client

Come già detto, Matrix è accessibile da differenti Client, attraverso i quali possiamo:

- Contattare utenti del server, con id `@user:matrix.NOME_DOMINIO`.
- Entrare in chat di gruppo, con id `#room:matrix.NOME_DOMINIO`.
- Quanto detto prima ma, grazie alla federazione, anche su server diversi.
- Collegare gli account per i bridge, aventi id `@whatsappbot:matrix.NOME_DOMINIO` e `@telegrambot:matrix.NOME_DOMINIO`.

## Client Nativi

Un esempio su dispositivi mobile sono **Element** e **Element X**, attraverso i quali è possibile accedere/registrarsi ad una qualsiasi istanza Matrix (per il nostro server dovremo usare come fornitore https://matrix.NOME_DOMINIO).

![immagine.png](report/immagine%202.png)

![immagine.png](report/immagine%203.png)

![immagine.png](report/immagine%204.png)

![immagine.png](report/immagine%205.png)

## WebClient

Possiamo anche definire un semplice **WebClient Element** accessibile da browser con il seguente container:

```yaml
  element:    
  container_name: 'element'    
  image: "vectorim/element-web:latest"    
  restart: "unless-stopped"    
  networks:      
  - matrix    
  volumes:      
  - "./matrix/element/config.json:/app/config.json:ro"    
  labels:      
  - "traefik.enable=true"      
  - "traefik.http.services.element.loadbalancer.server.port=80"      
  - "traefik.http.routers.element.rule=Host(`chat.${NOME_DOMINIO}`)"      
  - "traefik.http.routers.element.entrypoints=websecure"      
  - "traefik.http.routers.element.tls.certresolver=letsencrypt"
```

avviamo una volta `docker compose create element`, sostituiamo la seguente opzione nel file `./matrix/element/config.json` e riavviamo con `docker compose restart element`:

```json
{    
	"default_server_config": {        
      "m.homeserver": {            
      	"base_url": "https://matrix.NOME_DOMINIO"     
      }
}
```

Adesso avremo anche una webui in cui gli utenti possono registrarsi e accedere al server.

![immagine.png](report/immagine%206.png)

![immagine.png](report/immagine%207.png)
