Integrazione personalizzata per Home Assistant per i prodotti di ricarica Daze (Dazebox) e accesso con Google Auth
=======================================================================================

[![hacs_badge](https://img.shields.io/badge/HACS-Custom-orange.svg)](https://github.com/hacs/integration)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

Integrazione personalizzata per Home Assistant che espone i dati dei sensori di una wallbox [Daze](https://www.dazeservice.com/),
letti dallo stesso backend usato dall'app web ufficiale [webportal.dazeservice.com](https://webportal.dazeservice.com/).

> **Fork di [rdndnl/ha-daze](https://github.com/rdndnl/ha-daze)** che aggiunge l'accesso per gli **account che entrano
> con Google** (e quindi non hanno una password). Gli account con email e password continuano a funzionare come prima.
> Basato sulla v0.3.1 dell'originale.

> **Progetto non ufficiale.** Questa integrazione non è affiliata a Daze, né approvata o supportata da Daze.
> È stata realizzata analizzando il traffico di rete del portale web pubblico, poiché Daze non pubblica
> un'API. Può smettere di funzionare senza preavviso se Daze cambia il proprio backend.

## Cosa aggiunge questo fork

L'integrazione originale accede con email e password. Se ti sei registrato con il pulsante
"Accedi con Google" non esiste nessuna password, quindi quel metodo non può funzionare. Questo fork aggiunge un secondo
modo per configurare l'integrazione: incolli il **refresh token di Cognito** che il portale web salva nel browser
dopo un accesso con Google, e l'integrazione lo scambia da sola con nuovi token di accesso
(`REFRESH_TOKEN_AUTH`, lo stesso meccanismo che già usava per rinnovare le sessioni).

- Nuovo menu di configurazione: *Email e password* oppure *Account Google (refresh token)*.
- L'email dell'account viene letta dall'ID token, quindi non devi scriverla.
- Se il token salvato scade o viene revocato, Home Assistant avvia la normale procedura di riautenticazione e
  te ne chiede uno nuovo.
- Tutto il resto (sensori, pulsanti, polling) è invariato rispetto all'originale.

> **Stato: sperimentale.** Il percorso di accesso con Google è nuovo e ha avuto molti meno test nel mondo reale
> rispetto a quello con email e password. Se qualcosa non funziona, apri una issue.

## Funzionalità

La versione attuale contiene **solo sensori in sola lettura**, nessun controllo della ricarica (avvio/arresto, modalità eco): potrebbe arrivare in una versione futura.

Per ogni presa di ricarica:

- Potenza (W) ed energia della sessione in corso (kWh)
- Corrente di ricarica e tensione AC per fase (L1/L2/L3, se l'impianto è trifase)
- Temperatura della scheda e dell'involucro
- Stato della ventola
- Stato, stato EVSE ed errore di sistema, decodificati in valori leggibili

Per ogni wallbox (diagnostica):

- SSID Wi-Fi, versione software/firmware
- Corrente di rete per fase

Per ogni sito ("network" nella terminologia di Daze):

- Tariffa energetica e valuta

## Requisiti

- Home Assistant 2024.1.0 o successivo
- Un account Daze, a scelta tra:
  - l'email e la password che usi con l'app mobile o su [webportal.dazeservice.com](https://webportal.dazeservice.com/), oppure
  - un account che accede con Google (vedi [Account Google](#account-google))

## Installazione

### Tramite HACS (repository personalizzato)

Questa integrazione non è nello store predefinito di HACS, quindi va aggiunta come repository personalizzato.

Il modo più rapido è questo pulsante, che apre il repository direttamente nell'HACS della tua istanza di Home Assistant:

<a href="https://my.home-assistant.io/redirect/hacs_repository/?owner=depgiuseppe&amp;repository=ha-daze&amp;category=Integration" target="_blank" rel="noreferrer noopener"><img src="https://my.home-assistant.io/badges/hacs_repository.svg" alt="Apri la tua istanza di Home Assistant e apri un repository nell'Home Assistant Community Store."></a>

Poi premi **Scarica** e riavvia Home Assistant.

In alternativa, a mano:

1. HACS → menu `⋮` (in alto a destra) → **Repository personalizzati**
2. Repository: `https://github.com/depgiuseppe/ha-daze`, Categoria: **Integrazione**
3. Trova "Daze" in HACS e installalo
4. Riavvia Home Assistant

> Questo fork usa lo stesso dominio di integrazione (`daze`) dell'originale. Se avevi installato in precedenza
> `rdndnl/ha-daze`, rimuovilo da HACS (e toglilo dall'elenco dei repository personalizzati) prima di installare
> questo, poi riavvia Home Assistant.

### Manuale

Copia `custom_components/daze` nella cartella `config/custom_components/` di Home Assistant,
poi riavvia Home Assistant.

## Configurazione

Impostazioni → Dispositivi e servizi → **Aggiungi integrazione** → cerca **Daze**, poi scegli come accedi.

### Email e password

Scegli **Email e password** e inserisci email e password del tuo account Daze.

### Account Google

Scegli **Account Google (refresh token)** e incolla il tuo refresh token. Per ottenerlo:

1. Apri una **finestra privata/in incognito** e vai su [webportal.dazeservice.com](https://webportal.dazeservice.com/).
2. Accedi con Google.
3. Apri gli strumenti per sviluppatori del browser (F12) → **Application** (o **Applicazione**, se sono in italiano) →
   **Local storage** (**Archiviazione locale**) → `https://webportal.dazeservice.com`.
4. Copia il valore della chiave che termina con `.refreshToken`
   (ha una forma simile a `CognitoIdentityServiceProvider.<client id>.Google_<numero>.refreshToken`).
5. **Chiudi la finestra senza premere "Esci".** Il logout revoca il token e l'integrazione
   smetterebbe di funzionare.

Note:

- Tratta il refresh token come una password. Non incollarlo in issue, chat o screenshot.
- Per quanto tempo un refresh token resti valido lo decide Daze. Quando scade, Home Assistant ti chiederà di
  riautenticarti: ripeti i passaggi qui sopra e incolla un token nuovo.
- Usare una finestra privata dà all'integrazione una sessione tutta sua, separata da quella del tuo browser di tutti i giorni.

### Opzioni

L'intervallo di aggiornamento (predefinito 60 s) si può modificare in seguito dalle opzioni **Configura** dell'integrazione.

## Limitazioni

- Non è ancora possibile avviare/fermare la ricarica né gestire la modalità eco (previsto in una versione futura).
- Le etichette di `status`/`evse_state`/`system_error` sono state confrontate con il sorgente dell'app web ufficiale
  e coprono tutti i valori che vi sono definiti, ma ne sono stati osservati dal vivo solo pochi
  (soprattutto `standby`/`none`). Se un sensore passa a `unknown`, apri una issue
  indicando il valore grezzo.
- Gli account con email e password si autenticano sullo stesso client applicativo AWS Cognito del portale web, con
  `USER_PASSWORD_AUTH` diretto (senza browser/webview). Se Daze disattivasse questo flusso, l'integrazione dovrebbe essere rivista.
- Gli account Google non possono accedere in modo automatico: Cognito accetta solo gli indirizzi di reindirizzamento registrati
  per il portale web, quindi il refresh token va copiato a mano dal browser e rinnovato quando scade.

## Progetti collegati

Progetti indipendenti costruiti su questa integrazione, mantenuti da altre persone:

- **[DAZE Dashboard](https://github.com/fabiovit/daze-dashboard)** di [@fabiovit](https://github.com/fabiovit) -
  un pannello laterale per Home Assistant dedicato alle wallbox Daze (vista della ricarica in tempo reale, diagnostica, statistiche delle sessioni).

> Questi progetti non sono mantenuti da questo progetto, né affiliati o supportati da esso: segnala i problemi
nei loro repository.

## Sviluppo

### Preparazione

Richiede Python 3.14+ e [`uv`](https://docs.astral.sh/uv/).

```bash
uv sync
```

Questo installa gli strumenti di sviluppo e test dichiarati nel gruppo di dipendenze `dev` di `pyproject.toml`
(Home Assistant core, pytest e le librerie usate da `scripts/auth_spike.py`). L'integrazione
in sé non ha dipendenze a runtime oltre a quelle già fornite da Home Assistant: vedi
`custom_components/daze/manifest.json`.

### Eseguire i test

```bash
uv run python -m pytest tests/ -q
```

Usa `python -m pytest`, non il semplice `uv run pytest`: quest'ultimo non aggiunge la radice del
repository a `sys.path`, quindi il pacchetto `custom_components` non risulterebbe importabile.

I test usano payload di esempio catturati da vere sessioni HAR
(`tests/fixtures/*.json`) più il banco di prova di Home Assistant
(`pytest-homeassistant-custom-component`): non servono né un account Daze reale né accesso alla rete.

### Provare su un'istanza reale di Home Assistant

Crea un link simbolico dell'integrazione nella configurazione di HA e riavvia HA:

```bash
ln -s "$(pwd)/custom_components/daze" /path/to/homeassistant/config/custom_components/daze
```

Poi aggiungila da Impostazioni → Dispositivi e servizi come di consueto.

### `scripts/auth_spike.py`

Uno script autonomo (non incluso nell'integrazione distribuita) usato per verificare quale flusso di
autenticazione Cognito accetta il backend, senza passare da Home Assistant. Utile se Daze
cambiasse la propria configurazione di accesso e fosse necessario rivedere `custom_components/daze/auth.py`:

```bash
DAZE_EMAIL="you@example.com" DAZE_PASSWORD="..." uv run python scripts/auth_spike.py
```

## Contributi

Issue e PR per questo fork sono benvenute su [github.com/depgiuseppe/ha-daze](https://github.com/depgiuseppe/ha-daze/issues).
Per tutto ciò che non riguarda l'accesso con Google, valuta di segnalarlo anche al progetto originale su
[github.com/rdndnl/ha-daze](https://github.com/rdndnl/ha-daze/issues).


## Ringraziamenti

Questo progetto è un fork di [rdndnl/ha-daze](https://github.com/rdndnl/ha-daze): tutto il lavoro di reverse engineering
e l'integrazione stessa sono opera del suo autore e dei suoi collaboratori.

Grazie a [MasterJKapp](https://github.com/jkuran) per il contributo sul
controllo della ricarica (avvio/arresto/ripresa) e sui sensori di sessione, [PR #15](https://github.com/rdndnl/ha-daze/pull/15).

## Licenza

[MIT](LICENSE), come il progetto originale.
