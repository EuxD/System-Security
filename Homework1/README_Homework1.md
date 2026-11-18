# Homework 1 – OpenSSL

Primo homework del corso di **System Security**: crittografia simmetrica e asimmetrica, certificati X.509 v3 e conservazione della chiave privata, tutto con OpenSSL.

## Indice

- [Obiettivi](#obiettivi)
- [Struttura della cartella](#struttura-della-cartella)
- [Requisiti](#requisiti)
- [1. Crittografia simmetrica](#1-crittografia-simmetrica)
- [2. Crittografia asimmetrica](#2-crittografia-asimmetrica)
- [3. Certificati X.509 v3 e catena di certificati](#3-certificati-x509-v3-e-catena-di-certificati)
- [4. Conservazione della chiave privata](#4-conservazione-della-chiave-privata)
- [5. Simulazione con Docker](#5-simulazione-con-docker)

---

## Obiettivi

L'homework chiede di usare OpenSSL per:

1. **Generare chiavi simmetriche** e usarle con diversi algoritmi simmetrici.
2. **Generare chiavi asimmetriche** e usarle per cifrare, firmare file e firmare certificati, generando una catena di certificati.
3. **Generare certificati** secondo lo standard **X.509 v3**.
4. **Conservare la chiave privata** in un file PKCS#12 (`.p12`) e in un keystore gestito con `keytool`.

La presentazione completa del lavoro è in [`Presentazione_HW1.pptx`](Presentazione_HW1.pptx).

## Struttura della cartella

```
Homework1/
├── README.md
├── Presentazione_HW1.pptx
├── Simmetrica/
│   ├── AES-CBC/                    # AES-256 in modalità CBC
│   └── 3DES-CBC/                   # 3DES, confronto ECB/CBC
│       └── 3DES_chiave_corta/      # 3DES con K2 = K3 (equivale a DES)
├── Asimmetrica/
│   ├── Crittazione_Semplice+Firma/ # RSA: cifratura e firma
│   ├── Certificati/                # Root CA, CA intermedia, server
│   │   ├── root/
│   │   ├── intermediate/
│   │   └── server/
│   └── Keystore/                   # file PKCS#12 / keystore
└── progetto_docker_ibrida/         # simulazione Alice, Bob e CA con Docker
    ├── Dockerfile
    ├── docker-compose.yml
    ├── AVVIO_DOCKER.md             # come avviare i container
    ├── guida_comandi_utilizzo.md   # comandi da eseguire nei container
    └── data/                       # file generati da CA, Alice e Bob
```

I comandi usati in ogni esercizio sono nei file `Procedimento.txt` delle cartelle `Simmetrica/` e nella guida dentro `progetto_docker_ibrida/`.

## Requisiti

- **OpenSSL 3.x**
- **JDK** (per `keytool`), solo per la parte sul keystore
- **Docker Desktop** (o Docker Engine con il plugin Compose), solo per la simulazione Alice/Bob/CA

## 1. Crittografia simmetrica

Cartella: [`Simmetrica/`](Simmetrica/)

- **AES-256-CBC**: chiave da 32 byte e IV da 16 byte. Con blocchi da 16 byte, un file di 46 byte diventa di 48 byte dopo il padding.
- **3DES-CBC**: chiave da 24 byte e IV da 8 byte.
- **ECB contro CBC**: cifrando un file con blocchi ripetuti, 3DES-ECB produce blocchi cifrati identici e rivela la struttura del messaggio. In CBC ogni blocco viene messo in XOR con il precedente, quindi blocchi uguali producono output diversi.
- **3DES con chiave corta**: con K2 = K3, 3DES degenera in DES semplice. Il file si può quindi decifrare anche con DES, usando il provider `legacy` di OpenSSL.

## 2. Crittografia asimmetrica

Cartella: [`Asimmetrica/Crittazione_Semplice+Firma/`](Asimmetrica/Crittazione_Semplice+Firma/)

Generazione di una coppia di chiavi RSA da 2048 bit e uso per:

- **riservatezza**: il messaggio viene cifrato con la chiave pubblica e decifrato con quella privata;
- **firma digitale**: il messaggio viene firmato con la chiave privata (SHA-256) e la firma verificata con la chiave pubblica.

Nella cartella è presente anche una chiave privata protetta da password (`rsa_priv_enc.pem`).

## 3. Certificati X.509 v3 e catena di certificati

Cartella: [`Asimmetrica/Certificati/`](Asimmetrica/Certificati/)

La catena è **Root CA → CA intermedia → certificato server (`localhost`)**:

| Livello | Tipo | Note |
|---|---|---|
| Root CA | autofirmata, `CA:TRUE` | 10 anni |
| CA intermedia | firmata dalla Root, `CA:TRUE, pathlen:0` | 5 anni |
| Server | firmato dall'intermedia, `CA:FALSE`, `serverAuth` | 1 anno |

L'intermedia non si autofirma: genera una richiesta di certificato (CSR) che la Root firma. Lo stesso vale per il server, con la CA intermedia come firmataria. Il file `server_chain.crt` contiene il certificato del server seguito da quello dell'intermedia.

Verificando il certificato del server con la sola Root si ottiene l'errore `unable to get local issuer certificate`: la verifica riesce solo fornendo anche il certificato dell'intermedia.

## 4. Conservazione della chiave privata

Cartella: [`Asimmetrica/Keystore/`](Asimmetrica/Keystore/)

La chiave privata del server viene esportata insieme alla sua catena in un file **PKCS#12** (`.p12`) e poi importata in un keystore con `keytool`. I passaggi sono nella presentazione.

## 5. Simulazione con Docker

Cartella: [`progetto_docker_ibrida/`](progetto_docker_ibrida/)

Tre container Alpine con OpenSSL simulano uno scenario completo:

- **ca**: Root CA e CA intermedia, firma i certificati di Alice e Bob.
- **alice** e **bob**: generano le chiavi e le richieste di certificato, verificano le catene e si scambiano messaggi con **cifratura ibrida**: il messaggio è cifrato con AES-256-CBC, la chiave di sessione è cifrata con RSA e il messaggio è firmato con SHA-256 + RSA.

I container comunicano attraverso un volume condiviso (`/canale`), che rappresenta il canale di comunicazione.

- **Come avviare i container:** [`progetto_docker_ibrida/AVVIO_DOCKER.md`](progetto_docker_ibrida/AVVIO_DOCKER.md)
- **Comandi da eseguire passo per passo:** [`progetto_docker_ibrida/guida_comandi_utilizzo.md`](progetto_docker_ibrida/guida_comandi_utilizzo.md)
