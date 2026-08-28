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

Questo repository **privato** è la sorgente di verità del blueprint. Il file
installato dentro Home Assistant ne è una copia.

Attenzione a una conseguenza che non è ovvia: **Home Assistant non può
importare da un repository privato.** L'import scarica l'URL senza
credenziali — non esiste un punto, né nella UI né nell'API, in cui passare
un token — quindi un raw URL privato risponde 404, e i raw URL firmati che
GitHub genera (`?token=…`) scadono e vengono invalidati a ogni push.

Il deploy verso HA resta quindi manuale. Le due vie sono descritte sotto.

## Installazione — fatta il 28/08/2026

Il blueprint **è installato** nell'istanza, importato via URL. HA lo ha
salvato in:

```
blueprints/automation/192.168.68.130/sincronizza_condizionatore_con_termostato.yaml
```

La sottocartella prende il nome dall'host dell'URL di import: è brutta ma
innocua. Rinominarla richiede di spostare il file a mano sulla macchina HA
e va fatto **prima** di creare altre istanze, perché il `path` è
memorizzato dentro ogni automazione che usa il blueprint.

Il server HTTP che serviva il file era temporaneo, quindi **il pulsante
«Re-import blueprint» della UI oggi fallisce**. Per riapplicare una
modifica al file: rialza il server con il comando qui sotto e reimporta
con `overwrite=true`, oppure sposta il `source_url` su un Gist pubblico.

```
! nohup python3 -m http.server 8765 --directory /home/giorgio/claude_home_assistant/blueprints --bind 0.0.0.0 > /tmp/bp-server.log 2>&1 &
```

Alternativa senza porte aperte: copia il file in
`config/blueprints/automation/audit/` sulla macchina HA (app File editor,
Studio Code Server o Samba), poi **Impostazioni → Automazioni e scene →
Blueprint → Ricarica blueprint**. La sottocartella serve: HA non carica
blueprint messi direttamente in `blueprints/automation/`.

## Le quattro istanze

Entità verificate esistenti il 28/08/2026.

| Istanza | `termostato` | `condizionatore` | `sensore_finestra` |
|---|---|---|---|
| Soggiorno | `climate.termostato_soggiorno` | `climate.condizionatore_soggiorno` | `binary_sensor.termostato_soggiorno_windowopened` |
| Cameretta | `climate.termostato_cameretta` | `climate.condizionatore_cameretta` | `binary_sensor.termostato_cameretta_windowopened` |
| Studio | `climate.termostato_studio` | `climate.condizionatore_studio` | `binary_sensor.termostato_studio_windowopened` |
| Camera | `climate.termostato_camera` | `climate.condizionatore_camera` | `binary_sensor.termostato_camera_windowopened` |

`helper_presenza` e `stato_assenti` hanno già i valori giusti come default
(`input_select.presenza_casa` / `Assenti`): non vanno toccati.

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
