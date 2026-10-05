# Join tra Frutta e Magazzino — Apache NiFi

Pipeline ETL costruita interamente in **Apache NiFi**, senza database esterni, che simula un'integrazione dati tra due fonti indipendenti: un catalogo prodotti ("Frutta", con i prezzi) e un sistema di giacenze ("Magazzino", con le quantità).

Il flusso risponde a tre domande tipiche di data integration:
- Quali prodotti sono presenti **in entrambe** le fonti? → *inner join*
- Quali prodotti sono **a catalogo ma senza giacenza**? → *left outer join*
- Quali prodotti sono **in magazzino ma non a catalogo**? → *right outer join*
- Un **report di sintesi** con i conteggi delle tre casistiche

## Perché questo progetto

NiFi non offre un processore "JOIN" nativo. Il flusso dimostra come emularlo in modo nativo (senza script custom né database) sfruttando:
- uno **schema Avro unificato**, per unire due fonti con colonne diverse senza perdita di dati
- un **self-join SQL** (via `QueryRecord` / Apache Calcite), auto-referenziando la stessa tabella virtuale con alias diversi
- un **attributo di correlazione** per garantire che il report finale aggreghi sempre dati coerenti dello stesso ciclo di esecuzione

## Struttura del flusso

```
GenerateFlowFile (Frutta)  ─┐
                             ├─► UpdateRecord ─► MergeRecord ─► UpdateAttribute (batch_id)
GenerateFlowFile (Magazzino)─┘                                          │
                              ┌───────────────┬───────────────┬─────────┤
                              ▼               ▼               ▼         ▼
                       QueryRecord       QueryRecord     QueryRecord  QueryRecord
                       (Inner Join)      (Left Join)     (Right Join) (Report: 3 query)
                              │               │               │         │
                       UpdateAttribute  UpdateAttribute UpdateAttribute │
                       (filename)       (filename)      (filename)     ▼
                              │               │               │   MergeContent
                              │               │               │  (correlato su batch_id)
                              │               │               │         │
                              │               │               │   UpdateAttribute
                              │               │               │   (filename)
                              └───────┬───────┴───────┬───────┘
                                      ▼                ▼
                                           PutFile
```

Output finale: 4 file CSV — `inner_join.csv`, `left_join.csv`, `right_join.csv`, `report_finale.csv`.

## Screenshot

*(sostituisci con le tue immagini nella cartella `screenshots/`)*

![Vista d'insieme del flusso](screenshots/flusso-completo.png)
![Esempio di query self-join](screenshots/query-inner-join.png)
![Risultato finale](screenshots/output-finale.png)

## Come eseguirlo

**Prerequisiti**: Apache NiFi 2.x installato in locale.

1. **Importa il flusso**: nella canvas di NiFi, trascina l'icona "Process Group" e scegli "Import from file" (o trascina direttamente `flow-definition.json` dalla barra in alto) per caricare il template.
2. **Crea un Parameter Context**: menu ☰ in alto a destra → Parameter Contexts → crea un nuovo context con un parametro `output_dir`, valore = il percorso dove vuoi che vengano scritti i file di output sul tuo PC (es. `./output` oppure un percorso assoluto a tua scelta).
3. **Assegna il Parameter Context**: clic destro sullo sfondo della canvas → Configure → seleziona il Parameter Context creato al passo precedente.
4. **Abilita i Controller Service**: menu ☰ → Controller Services → abilita (icona fulmine ⚡) `AvroSchemaRegistry`, `CSVReader` e `CSVRecordSetWriter`.
5. **Avvia il flusso**: seleziona tutti i processori (Ctrl+A) → Start. I due `GenerateFlowFile` sono impostati sullo stesso intervallo di schedulazione, così da restare sempre sincronizzati tra loro (condizione necessaria perché il merge unisca sempre un batch frutta con un batch magazzino).
6. **Controlla l'output**: nella cartella indicata nel parametro `output_dir` troverai i 4 file CSV generati.

## Struttura del repository

```
.
├── flow-definition.json   # Template del flusso NiFi, importabile direttamente
├── README.md
└── screenshots/           # Immagini del flusso e dei risultati
```

## Note tecniche

Il dettaglio di ogni processore (perché è stato scelto, come è configurato, le query SQL usate) è disponibile in [`documentazione_flusso_nifi.md`](documentazione_flusso_nifi.md).

## Licenza

MIT — libero riutilizzo per scopi di studio e dimostrativi.
