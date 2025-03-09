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