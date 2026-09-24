# ppal

Client da riga di comando per l'API **PrinterPlan**, pensato per essere usato anche da un LLM (Claude, kiro-cli, ecc.).

- **Un solo file eseguibile**, senza dipendenze: non servono Python, Node o altri runtime.
- **Tool dinamici**: l'elenco delle operazioni viene letto ogni volta dal backend, quindi quando il backend aggiunge o modifica endpoint `ppal` li vede subito, senza aggiornamenti.
- **Due modi d'uso**: come comando da terminale, oppure come server [MCP](https://modelcontextprotocol.io) da collegare al tuo assistente.
- **Login con AccessKey** e gestione automatica del token (rinnovo incluso).

## Installazione

**Mac e Linux**

```sh
curl -fsSL https://github.com/Stefac/ppal/releases/latest/download/install.sh | sh
```

**Windows** (PowerShell)

```powershell
irm https://github.com/Stefac/ppal/releases/latest/download/install.ps1 | iex
```

Lo script scarica il file giusto per il tuo sistema, ne verifica lo SHA-256 e lo installa in `~/.local/bin` (Mac/Linux) o `%LOCALAPPDATA%\ppal` (Windows). Se la cartella non è nel `PATH` ti dice come aggiungerla.

Opzioni: `PPAL_VERSION=0.1.0` per una versione precisa, `PPAL_INSTALL_DIR=/percorso` per cambiare cartella.

Se preferisci fare a mano, scarica il file per la tua piattaforma dalla pagina [Releases](https://github.com/Stefac/ppal/releases), rendilo eseguibile (`chmod +x`) e mettilo nel `PATH`.

Piattaforme: macOS (Apple Silicon e Intel), Linux (amd64 e arm64), Windows (amd64 e arm64).

## Primi passi

```sh
# 1. Indica dove gira il backend (local è già http://localhost:5001)
ppal env set remote https://<indirizzo-del-backend>
ppal env use remote

# 2. Accedi con la tua AccessKey (viene chiesta a terminale)
ppal login

# 3. Guarda cosa puoi fare e provalo
ppal tools
ppal tools SearchPrinters
ppal call SearchPrinters status=offline limit=10
```

## Comandi

| Comando | Cosa fa |
|---|---|
| `ppal env` | Mostra gli ambienti configurati e quello corrente |
| `ppal env use <nome>` | Sceglie l'ambiente di default (`local`, `remote`, …) |
| `ppal env set <nome> <url>` | Crea o aggiorna un ambiente |
| `ppal login [--key KEY]` | Accede con l'AccessKey (da `--key`, da `PPAL_ACCESS_KEY` o chiesta a terminale) |
| `ppal logout` | Dimentica token e AccessKey dell'ambiente |
| `ppal status` | Mostra ambiente, URL e stato del login |
| `ppal tools` | Elenca le operazioni esposte dal backend |
| `ppal tools <nome>` | Spiega come chiamare una operazione (parametri, tipi, default) |
| `ppal call <nome> [k=v …]` | Esegue una operazione e stampa il risultato come JSON compatto |
| `ppal guide` | Istruzioni pensate per un LLM |
| `ppal mcp` | Avvia il server MCP (vedi sotto) |

Per usare un ambiente diverso solo per una chiamata: `ppal --env local call SearchPrinters limit=5`.

### Passare i parametri a `call`

```sh
ppal call SearchPrinters model=WF-C878 limit=5        # k=v: i valori sono JSON se validi, altrimenti testo
ppal call GetPrinter customerPrinterId=589
ppal call SomeOperation '{"filter":"x","body":{"a":1}}' # oppure un unico oggetto JSON
echo '{"body":{"a":1}}' | ppal call SomeOperation -    # oppure JSON da stdin
```

Il nome dell'operazione si può abbreviare se non è ambiguo. Se la risposta è un errore HTTP, il messaggio va su stdout, `HTTP <codice>` su stderr e il codice di uscita è 1.

## Usarlo con un assistente LLM

### Come server MCP (consigliato)

Accedi una volta (`ppal login`), poi registra il server. Con **Claude Code**:

```sh
claude mcp add printerplan -- ppal mcp
```

Con **Claude Desktop** (o altri client MCP), aggiungi al file di configurazione:

```json
{
  "mcpServers": {
    "printerplan": {
      "command": "ppal",
      "args": ["mcp"]
    }
  }
}
```

Se il client non trova `ppal` nel `PATH`, usa il percorso completo. Per un ambiente specifico aggiungi `"--env", "remote"` agli `args`.

Il server espone solo tre strumenti leggeri, così l'assistente consuma pochi token:

1. `list_tools`: elenca le operazioni disponibili (con `filter` opzionale);
2. `describe_tool`: spiega come chiamarne una;
3. `call_tool`: la esegue.

Con `ppal mcp --all` ogni operazione viene invece esposta come strumento a sé.

### Da riga di comando

Se l'assistente può eseguire comandi (Bash), aggiungi le istruzioni al file `CLAUDE.md` o `AGENTS.md` del tuo progetto:

```sh
ppal guide --snippet >> CLAUDE.md
```

L'assistente leggerà `ppal guide` e userà `ppal tools` e `ppal call`.

## Accesso e sicurezza

- Serve un'**AccessKey** personale. `ppal` la usa per ottenere un token e lo rinnova da solo quando scade.
- Sessione e AccessKey sono salvate in un file leggibile solo da te (permessi `0600`), nella cartella di configurazione dell'utente:
  macOS `~/Library/Application Support/ppal`, Linux `~/.config/ppal`, Windows `%AppData%\ppal`.
- Non incollare l'AccessKey nei prompt di un assistente e non scriverla in file del progetto. `ppal logout` cancella tutto.
- Per l'uso non interattivo puoi impostare `PPAL_ACCESS_KEY` nell'ambiente del processo.
- Ogni utente vede solo i dati che la sua AccessKey permette (un amministratore vede tutta l'azienda, gli altri solo i clienti assegnati).

## Variabili d'ambiente

| Variabile | Effetto |
|---|---|
| `PPAL_ENV` | Ambiente da usare (come `--env`) |
| `PPAL_URL` | Sostituisce l'URL dell'ambiente |
| `PPAL_ACCESS_KEY` | AccessKey per il login automatico |
| `PPAL_SPEC` | URL o file da cui leggere lo swagger, se il backend non lo espone |

## Aggiornamento e rimozione

- **Aggiornare**: riesegui il comando di installazione.
- **Rimuovere**: cancella il file `ppal` (o `ppal.exe`) e la cartella di configurazione indicata sopra.

## Problemi comuni

- **`ppal: command not found`**: la cartella di installazione non è nel `PATH`. Aggiungila (l'installer stampa il comando).
- **macOS blocca il file** («sviluppatore non verificato»): succede se lo hai scaricato con il browser. Sblocca con `xattr -d com.apple.quarantine /percorso/ppal`. Con lo script `curl` non capita.
- **«non sei loggato»**: esegui `ppal login` (o imposta `PPAL_ACCESS_KEY`).
- **«impossibile leggere lo swagger»**: controlla l'URL dell'ambiente con `ppal status` e che il backend sia raggiungibile.
- **L'elenco delle operazioni non cambia dopo un aggiornamento del backend**: `ppal tools --refresh` ignora la cache di 5 minuti.
