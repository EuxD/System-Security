# Guida ai comandi: CA, Alice e Bob

Questa guida riporta, in ordine, i comandi OpenSSL da eseguire nei tre container dell'ambiente Docker. Per avviare i container e entrare al loro interno, vedi [`AVVIO_DOCKER.md`](AVVIO_DOCKER.md).

Ogni sezione indica **in quale container** eseguire i comandi:

| Container | Come entrare | Cartella di lavoro |
|---|---|---|
| `ca` | `docker compose exec ca bash` | `/work` (`data/ca`) |
| `alice` | `docker compose exec alice bash` | `/work` (`data/alice`) |
| `bob` | `docker compose exec bob bash` | `/work` (`data/bob`) |

`/canale` è la cartella condivisa tra i tre container e rappresenta il canale di comunicazione: ciò che viene copiato lì è visibile a tutti.

## Indice

1. [Root CA](#1-root-ca-container-ca)
2. [CA intermedia](#2-ca-intermedia-container-ca)
3. [Chiavi e richieste di certificato di Alice e Bob](#3-chiavi-e-richieste-di-certificato-container-alice-e-bob)
4. [Firma dei certificati](#4-firma-dei-certificati-container-ca)
5. [Pubblicazione dei certificati delle CA](#5-pubblicazione-dei-certificati-delle-ca-container-ca)
6. [Recupero e verifica dei certificati](#6-recupero-e-verifica-dei-certificati)
7. [Comunicazione con cifratura ibrida](#7-comunicazione-con-cifratura-ibrida)
8. [Risposta di Bob](#8-risposta-di-bob)
9. [Secondo messaggio di Alice](#9-secondo-messaggio-di-alice)
10. [Note](#10-note)

---

## 1. Root CA (container `ca`)

```bash
docker compose exec ca bash
```

Una volta dentro il container:

```bash
mkdir -p root inter

# chiave privata della Root CA
openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:4096 -out root/root.key

# certificato autofirmato della Root CA (10 anni)
openssl req -x509 -new -key root/root.key -sha256 -days 3650 \
  -subj "/C=IT/O=Homework/CN=Root CA" \
  -addext "basicConstraints=critical,CA:TRUE" \
  -addext "keyUsage=critical,keyCertSign,cRLSign" \
  -out root/root.crt

# controllo del risultato
openssl x509 -in root/root.crt -noout -subject -issuer
```

Il certificato è autofirmato, quindi `subject` e `issuer` devono coincidere.

## 2. CA intermedia (container `ca`)

```bash
# chiave privata della CA intermedia
openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:2048 -out inter/inter.key

# l'intermedia chiede alla Root di essere certificata (prepara la richiesta)
openssl req -new -key inter/inter.key -sha256 \
  -subj "/C=IT/O=Homework/CN=Intermediate CA" \
  -out inter/inter.csr
```

**La Root firma l'intermedia** (5 anni, `pathlen:0`: l'intermedia può firmare certificati finali ma non altre CA):

```bash
openssl req -x509 -in inter/inter.csr \
  -CA root/root.crt -CAkey root/root.key \
  -days 1825 -sha256 \
  -addext "basicConstraints=critical,CA:TRUE,pathlen:0" \
  -addext "keyUsage=critical,keyCertSign,cRLSign" \
  -out inter/inter.crt
```

**Verifica:**

```bash
openssl verify -CAfile root/root.crt inter/inter.crt
openssl x509 -in inter/inter.crt -noout -subject -issuer -ext basicConstraints,keyUsage
```

## 3. Chiavi e richieste di certificato (container `alice` e `bob`)

**Alice** (container `alice`):

```bash
openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:2048 -out alice.key

openssl req -new -key alice.key -sha256 \
  -subj "/C=IT/O=Homework/CN=Alice" \
  -out /canale/alice.csr
```

**Bob** (container `bob`):

```bash
openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:2048 -out bob.key

openssl req -new -key bob.key -sha256 \
  -subj "/C=IT/O=Homework/CN=Bob" \
  -out /canale/bob.csr
```

Le chiavi private restano nel container di ciascuno. Sul canale passa solo la richiesta di certificato (CSR).

## 4. Firma dei certificati (container `ca`)

**Alice**, certificato firmato dalla **Root** (1 anno):

```bash
openssl req -x509 -in /canale/alice.csr \
  -CA root/root.crt -CAkey root/root.key \
  -days 365 -sha256 \
  -addext "basicConstraints=critical,CA:FALSE" \
  -addext "keyUsage=critical,digitalSignature,keyEncipherment" \
  -out /canale/alice.crt
```

**Bob**, certificato firmato dalla **CA intermedia** (1 anno):

```bash
openssl req -x509 -in /canale/bob.csr \
  -CA inter/inter.crt -CAkey inter/inter.key \
  -days 365 -sha256 \
  -addext "basicConstraints=critical,CA:FALSE" \
  -addext "keyUsage=critical,digitalSignature,keyEncipherment" \
  -out /canale/bob.crt
```

I due certificati hanno percorsi di fiducia diversi: quello di Alice è firmato direttamente dalla Root, quello di Bob passa dall'intermedia.

## 5. Pubblicazione dei certificati delle CA (container `ca`)

```bash
cp root/root.crt inter/inter.crt /canale/
```

## 6. Recupero e verifica dei certificati

**Alice** (container `alice`): verifica che il suo certificato sia valido rispetto alla Root.

```bash
cp /canale/alice.crt /canale/root.crt .
openssl verify -CAfile root.crt alice.crt
```

**Bob** (container `bob`): il certificato è firmato dall'intermedia, quindi serve anche `inter.crt` come certificato intermedio.

```bash
cp /canale/bob.crt /canale/inter.crt /canale/root.crt .
openssl verify -CAfile root.crt -untrusted inter.crt bob.crt
```

## 7. Comunicazione con cifratura ibrida

### Passo 1: Alice manda il suo certificato a Bob (container `alice`)

```bash
cp alice.crt /canale/
```

### Passo 2: Bob verifica il certificato di Alice ed estrae la chiave pubblica (container `bob`)

```bash
cp /canale/alice.crt .
openssl verify -CAfile root.crt alice.crt
openssl x509 -in alice.crt -noout -subject -issuer -dates
openssl x509 -in alice.crt -pubkey -noout > alice.pub
```

### Passo 3: Bob risponde con la sua catena completa (container `bob`)

```bash
cat bob.crt inter.crt > bob-chain.pem
cp bob-chain.pem /canale/
```

### Passo 4: Alice verifica la catena di Bob ed estrae la chiave pubblica (container `alice`)

```bash
cp /canale/bob-chain.pem .
openssl verify -CAfile root.crt -untrusted bob-chain.pem bob-chain.pem
openssl x509 -in bob-chain.pem -noout -subject -issuer
openssl x509 -in bob-chain.pem -pubkey -noout > bob.pub
```

### Passo 5: Alice firma e cifra il messaggio (container `alice`)

```bash
# il messaggio
echo "Ciao Bob, questo è un messaggio riservato da Alice." > msg.txt

# firma: hash SHA-256 del messaggio, cifrato con la chiave privata di Alice
openssl dgst -sha256 -sign alice.key -out msg.sig msg.txt

# chiave simmetrica casuale (32 byte in esadecimale) e IV (16 byte)
openssl rand -hex 32 > k.key
openssl rand -hex 16 > iv.txt

# cifratura del messaggio con la chiave di sessione (AES-256-CBC)
openssl enc -aes-256-cbc -K $(cat k.key) -iv $(cat iv.txt) -in msg.txt -out msg.enc

# cifratura della chiave di sessione con la chiave pubblica di Bob
openssl pkeyutl -encrypt -pubin -inkey bob.pub -in k.key -out k.enc

# invio sul canale: solo ciò che deve viaggiare
cp msg.enc k.enc iv.txt msg.sig /canale/
```

### Passo 6: Bob decifra e verifica (container `bob`)

```bash
# recupero i dati dal canale
cp /canale/msg.enc /canale/k.enc /canale/iv.txt /canale/msg.sig .

# recupero la chiave di sessione decifrando con la chiave privata di Bob
openssl pkeyutl -decrypt -inkey bob.key -in k.enc -out ks.key

# decifro il messaggio con la chiave di sessione
openssl enc -d -aes-256-cbc -K $(cat ks.key) -iv $(cat iv.txt) -in msg.enc -out msg.txt

# visualizzo il messaggio
cat msg.txt

# verifico la firma: controlla che il messaggio non sia stato modificato
openssl dgst -sha256 -verify alice.pub -signature msg.sig msg.txt
```

Se tutto è corretto, l'ultimo comando stampa `Verified OK`.

> A questo punto Alice e Bob condividono la chiave di sessione e possono comunicare tra loro. Per lo scambio reale si dovrebbero aggiungere dei **NONCE** per evitare i **replay attack**; qui si considera il canale sicuro.

## 8. Risposta di Bob

Container `bob`:

```bash
# Bob crea la risposta per Alice
echo "Ciao Alice, ho ricevuto il tuo messaggio" > risp.txt

# Bob firma la risposta con la sua chiave privata
openssl dgst -sha256 -sign bob.key -out risp.sig risp.txt

# Bob cifra la risposta con la chiave di sessione
openssl enc -aes-256-cbc -K $(cat ks.key) -iv $(cat iv.txt) -in risp.txt -out risp.enc

# Bob mette messaggio cifrato e firma sul canale
cp risp.enc risp.sig /canale/
```

Container `alice`:

```bash
# Alice recupera firma e messaggio
cp /canale/risp.enc /canale/risp.sig .

# Alice decifra il messaggio con la chiave di sessione
openssl enc -d -aes-256-cbc -K $(cat k.key) -iv $(cat iv.txt) -in risp.enc -out risp.txt

# Alice legge il messaggio
cat risp.txt

# Alice verifica la firma di Bob
openssl dgst -sha256 -verify bob.pub -signature risp.sig risp.txt
```

## 9. Secondo messaggio di Alice

Con la chiave di sessione già condivisa, Alice può inviare altri messaggi senza ripetere lo scambio della chiave con RSA.

Container `alice`:

```bash
echo "Ciao Bob, come stai" > msg2.txt

openssl dgst -sha256 -sign alice.key -out msg2.sig msg2.txt
openssl enc -aes-256-cbc -K $(cat k.key) -iv $(cat iv.txt) -in msg2.txt -out msg2.enc

cp msg2.enc msg2.sig /canale/
```

Container `bob`:

```bash
cp /canale/msg2.enc /canale/msg2.sig .

openssl enc -d -aes-256-cbc -K $(cat ks.key) -iv $(cat iv.txt) -in msg2.enc -out msg2.txt
cat msg2.txt

openssl dgst -sha256 -verify alice.pub -signature msg2.sig msg2.txt
```

### Prova di manomissione (facoltativa)

Per vedere che la firma rileva le modifiche, Bob può alterare il messaggio decifrato e rifare la verifica:

```bash
echo "riga aggiunta" >> msg2.txt
openssl dgst -sha256 -verify alice.pub -signature msg2.sig msg2.txt
```

Questa volta il comando stampa `Verification failure`, perché il contenuto non corrisponde più alla firma di Alice.

## 10. Note

- **Cosa viaggia sul canale:** solo CSR, certificati, messaggi cifrati, chiave di sessione cifrata, IV e firme. Le chiavi private (`root.key`, `inter.key`, `alice.key`, `bob.key`) e la chiave di sessione in chiaro (`k.key` e `ks.key`) non vengono mai copiate in `/canale`.
- **Perché due catene diverse:** Alice è certificata direttamente dalla Root e Bob dalla CA intermedia. Per questo Bob deve pubblicare la catena completa (`bob-chain.pem`) e Alice verifica con `-untrusted`.
- **Avviso `No -copy_extensions given`:** compare quando la CA firma una richiesta e si può ignorare. Significa che le estensioni vengono prese dai parametri `-addext` del comando e non dalla CSR.
- **IV riutilizzato:** per semplicità tutti i messaggi usano lo stesso IV (`iv.txt`) con la stessa chiave di sessione. In un sistema reale, con CBC, l'IV deve essere diverso a ogni messaggio.
- **Replay attack:** senza un nonce, un attaccante potrebbe rimandare un messaggio valido già intercettato. Qui il problema non è trattato.
