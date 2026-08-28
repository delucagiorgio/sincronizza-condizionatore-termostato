# Blueprint M4 — Sincronizza condizionatore con termostato

File: `sincronizza_condizionatore_con_termostato.yaml`

Sostituisce le quattro automazioni gemelle `gestione_condizionatore_*`
(rilievo **M4** dell'audit) con una sola definizione parametrica.

## Cosa cambia rispetto alle quattro automazioni attuali

| | Prima | Con il blueprint |
|---|---|---|
| Definizioni | 4 automazioni, ~600 righe | 1 blueprint + 4 istanze |
| Guard sul setpoint | presente in 3 su 4 — **manca al soggiorno** | presente per costruzione, in tutte |
| Riferimenti | `device_id` in trigger e condizioni | solo `entity_id` (sana anche **M3** per queste quattro) |
| Ordine dei blocchi | divergente fra studio e le altre | unico |

Il comportamento è identico: gli stessi 6 trigger, gli stessi 5 rami, lo
stesso preset `away` in assenza. L'unica differenza di runtime è che il
soggiorno smette di scrivere il setpoint quando è già corretto.

## Dove vive questo file

Repository: **https://github.com/delucagiorgio/sincronizza-condizionatore-termostato**
(pubblico, un file per cartella). Questa è la sorgente di verità; il file
dentro Home Assistant ne è una copia.

Il repository è pubblico per una ragione precisa e non per distrazione:
**Home Assistant non può importare da un repository privato.** L'import
scarica l'URL senza credenziali — non esiste un punto, né nella UI né
nell'API, in cui passare un token — quindi un raw URL privato risponde 404,
e i raw URL firmati che GitHub genera (`?token=…`) scadono e vengono
invalidati a ogni push. Pubblico significa quindi «il pulsante Re-import
funziona per sempre».

Cosa c'è di esposto: un YAML interamente parametrico. L'unico valore
concreto è il default `input_select.presenza_casa`. Gli entity_id delle
stanze non sono qui — vivono nelle quattro istanze, che stanno dentro HA e
non vengono pubblicate. **Prima di aggiungere qualsiasi file a questo
repository, ricontrollare che non contenga URL Nabu Casa, token o
indirizzi.**

## Installazione e aggiornamento

Importa da questo URL, o incollalo in **Impostazioni → Automazioni e scene
→ Blueprint → Importa blueprint**:

```
https://github.com/delucagiorgio/sincronizza-condizionatore-termostato/blob/main/thermostat_ac_sync_blueprint/sincronizza_condizionatore_con_termostato.yaml
```

È lo stesso valore scritto in `source_url`, quindi il pulsante **«Re-import
blueprint»** della UI ripesca sempre l'ultima versione di `main`: per
distribuire una modifica basta fare push.

Attenzione a una cosa sola: HA ricava il nome della sottocartella
dall'URL di import, quindi cambiando sorgente il `path` del blueprint
cambia. Ogni istanza memorizza quel `path`, perciò un cambio di sorgente
richiede di aggiornare tutte le istanze e cancellare il blueprint vecchio.

## Le quattro istanze

Entità verificate esistenti il 28/08/2026.

| Istanza | `termostato` | `condizionatore` | `sensore_finestra` |
|---|---|---|---|
| Soggiorno | `climate.termostato_soggiorno` | `climate.condizionatore_soggiorno` | `binary_sensor.termostato_soggiorno_windowopened` |
| Cameretta | `climate.termostato_cameretta` | `climate.condizionatore_cameretta` | `binary_sensor.termostato_cameretta_windowopened` |
| Studio | `climate.termostato_studio` | `climate.condizionatore_studio` | `binary_sensor.termostato_studio_windowopened` |
| Camera | `climate.termostato_camera` | `climate.condizionatore_camera` | `binary_sensor.termostato_camera_windowopened` |

I tre input della sezione «Presenza in casa» — `helper_presenza`,
`stato_assenti` e `preset_presenza` — hanno già i valori giusti come default
(`input_select.presenza_casa`, `Assenti`, `custom`): non vanno toccati.

Nota: fino al 28/08/2026 il condizionatore del soggiorno era
`climate.condizionatore_soggiorno_2` — la trappola del rilievo A1. Il suffisso
segnalava che alla creazione la forma senza `_2` era occupata; da chi, non è
verificabile oggi, perché il registry conserva solo lo stato attuale.
Il 28/08/2026 la forma senza suffisso risultava libera, l'entità è stata
rinominata in `climate.condizionatore_soggiorno` e i quattro consumatori
(tre automazioni e una card) sono stati aggiornati: la tabella qui sopra è di
nuovo simmetrica con le altre tre stanze.

## Migrazione — completata il 28/08/2026

| Stanza | Istanza blueprint | Automazione originale |
|---|---|---|
| Studio | `automation.sincronizza_condizionatore_studio` — on | *cancellata* |
| Cameretta | `automation.sincronizza_condizionatore_cameretta` — on | `gestione_condizionatore_cameretta` — **off** |
| Camera | `automation.sincronizza_condizionatore_camera` — on | `gestione_condizionatore_camera` — **off** |
| Soggiorno | `automation.sincronizza_condizionatore_soggiorno` — on | `gestione_condizionatore_soggiorno` — **off** |

Lo studio è stato il pilota. Le tre originali rimaste sono in `off` e non
cancellate, ciascuna con una `description` che dice perché e come tornare
indietro. Vanno eliminate quando la rete di sicurezza non serve più.

**Per tornare indietro su una stanza:** rimetti in `on` la vecchia e
cancella l'istanza del blueprint. Nessun altro passo da annullare.

## Il preset `away` che restava appeso — corretto il 28/08/2026

Difetto reale, riprodotto sui dati del 22/08/2026 e corretto in questa
versione del blueprint.

La sequenza era questa:

| Ora | Cosa succedeva | Preset | Setpoint |
|---|---|---|---|
| 17:46 | `presenza_casa` → `Assenti` | `custom` | 24 |
| 17:56 | l'automazione di presenza mette `away` (i 10 min del suo `for:`) | `away` | 25 |
| 18:00 | lo **Scheduler** spegne il termostato | `away` | 25 |
| 19:57 | `presenza_casa` → `Casa` | `away` | 25 |
| 19:58 | il termostato è ancora spento, e ancora in `away` | `away` | 25 |

La causa è la guardia nel ramo «Casa» dell'automazione di presenza:
`if not (termostato è off) → set custom`. Un termostato **spento** al momento
del rientro viene saltato, e nulla rimette `custom` — quel ramo scatta solo
sulla transizione, che è già avvenuta. Alla riaccensione successiva il
termostato riparte alla temperatura di assenza.

Lo Scheduler rende la cosa quasi garantita invece che rara: spegne cameretta
e studio alle 18:00 e la camera alle 20:00, tutti i giorni, senza condizioni.

**La correzione** è il blocco «A casa, riaccendendo il termostato, esci dal
preset away»: scatta sui trigger `heat`, `cool` e `window_closed`, solo se
la casa è occupata e solo se il preset è ancora `away`. Un `comfort` o uno
`sleep` scelti a mano restano intatti, e un termostato spento non viene mai
toccato.

Sbavatura da conoscere: quando il preset torna a `custom` il termostato
ripristina il proprio setpoint e lo comunica un istante dopo, quindi la
prima esecuzione può scrivere sul condizionatore ancora il valore di `away`.
Il cambio di attributo fa partire subito una seconda esecuzione che corregge.
Converge da solo, al costo di una scrittura in più.

## Limite noto: la sincronizzazione è a senso unico

Il blueprint replica termostato → condizionatore. **Un comando dato
direttamente al condizionatore non torna indietro al termostato.**

Non è una svista del blueprint: è il comportamento delle quattro
automazioni originali, riprodotto fedelmente. Ma il divario si vede: al
28/08/2026 tutti e quattro i condizionatori erano in `dry` con i quattro
termostati in `cool`.

Prima di aggiungere il senso inverso servono quattro decisioni, perché i
due dispositivi non sono equivalenti:

| | Termostato (Meross) | Condizionatore (Gree) |
|---|---|---|
| Modalità | `off`, `heat`, `cool` | `auto`, `cool`, `dry`, `fan_only`, `heat`, `off` (+ `heat_cool` sul soggiorno) |
| Setpoint | 5–35 °C, passo **0,5** | 16–30 °C, passo **1** |
| Preset | custom, comfort, sleep, away, auto | nessuno |
| Aggiornamento | immediato | **polling a 60 s** |

1. **Modalità non mappabili.** `dry`, `fan_only`, `auto` e `heat_cool` non
   esistono sul termostato. Mapparle su `off` è pericoloso: spegnerebbe il
   termostato, che per la regola diretta spegnerebbe il condizionatore
   uno o due secondi dopo aver ricevuto il comando.
2. **Ping-pong sul setpoint.** Passo 0,5 contro passo 1: un 21,5 sul
   termostato diventa 21 o 22 sul condizionatore, e il ritorno
   riscriverebbe 21 sul termostato cambiando in silenzio la tua scelta.
   Serve una tolleranza.
3. **Il preset `away`.** Il ritorno del setpoint sovrascriverebbe il
   preset che la regola di assenza ha appena impostato.
4. **`unavailable`.** I Gree cadono a intermittenza: il ritorno deve
   ignorare `unavailable` e `unknown`, o proverebbe a scriverli sul
   termostato.
