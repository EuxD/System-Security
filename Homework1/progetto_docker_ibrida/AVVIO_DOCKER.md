# Avvio dei container Docker (Alice, Bob e CA)

Questa guida spiega come avviare l'ambiente Docker dell'homework e come entrare nei container per eseguire i comandi. I comandi OpenSSL veri e propri, da eseguire nell'ordine giusto, sono in [`guida_comandi_utilizzo.txt`](guida_comandi_utilizzo.txt).

## Cosa fa l'ambiente

Il progetto avvia **tre container** basati su Alpine Linux, con OpenSSL e bash:

| Container | Ruolo | Cartella locale | Cartella nel container |
|---|---|---|---|
| `ca` | Root CA e CA intermedia | `data/ca` | `/work` |
| `alice` | Utente Alice | `data/alice` | `/work` |
| `bob` | Utente Bob | `data/bob` | `/work` |

Tutti e tre condividono un **volume** chiamato `canale`, montato in `/canale`. Rappresenta il canale di comunicazione: ogni file che un container copia lì è visibile agli altri due. Il resto resta privato nel container, quindi per esempio le chiavi private non passano mai dal canale.

Le cartelle `data/...` sono montate sul tuo computer, quindi tutto ciò che crei in `/work` compare anche nella cartella locale e resta salvato dopo lo spegnimento dei container.

## Requisiti

- **Docker Desktop** installato e **aperto** (su Windows deve essere attivo il backend WSL2).
- Verifica che Docker risponda:

```powershell
docker --version
docker compose version
```

## Avvio

**1. Apri un terminale nella cartella del progetto**, quella che contiene `docker-compose.yml` e `Dockerfile` (`progetto_docker_ibrida`).

Su Windows il modo più semplice è aprire la cartella in Esplora file, fare clic destro in uno spazio vuoto e scegliere **Apri nel terminale**. Oppure:

```powershell
cd "percorso\della\repo\Homework1\progetto_docker_ibrida"
```

Controlla di essere nella cartella giusta:

```powershell
dir
```

Devi vedere `docker-compose.yml`, `Dockerfile`, `data` e `guida_comandi_utilizzo.txt`.

**2. Costruisci l'immagine e avvia i container** in background:

```powershell
docker compose up -d --build
```

**3. Controlla che siano attivi:**

```powershell
docker compose ps
```

Devi vedere `ca`, `alice` e `bob` con stato `running`. I container non stampano nulla perché restano in attesa (`sleep infinity`): ci lavori entrando dentro.

## Entrare nei container

Apri **un terminale per ogni container**, tutti nella stessa cartella del progetto:

```powershell
docker compose exec ca bash
docker compose exec alice bash
docker compose exec bob bash
```

Il prompt cambia in qualcosa come `ca:/work#`. Da lì puoi eseguire i comandi di `guida_comandi_utilizzo.txt`. Per uscire da un container basta scrivere `exit`: i container restano accesi.

Un ordine tipico di lavoro:

1. **`ca`**: crea la Root CA e la CA intermedia.
2. **`alice` e `bob`**: ognuno genera la propria chiave e la richiesta di certificato (`.csr`), copiandola in `/canale`.
3. **`ca`**: firma i certificati di Alice (con la Root) e di Bob (con l'intermedia) e pubblica i certificati delle CA in `/canale`.
4. **`alice` e `bob`**: recuperano i certificati dal canale e li verificano.
5. **`alice` e `bob`**: si scambiano messaggi con cifratura ibrida e firma.

I punti 4 e 5 alternano i container, quindi tieni aperti i tre terminali.

## Spegnimento

Dalla cartella del progetto:

```powershell
docker compose stop        # ferma i container, che restano esistenti
docker compose down        # ferma e rimuove i container
docker compose down -v     # rimuove anche il volume "canale"
```

I file nelle cartelle `data/...` **non vengono cancellati** da nessuno di questi comandi. Il contenuto del canale (`/canale`) si perde solo con `down -v`.

