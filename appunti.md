# IMPLEMENTAZIONE

## Punto 1 — Schema evento unificato
L'idea è definire un dataclass (o dizionario tipizzato) che diventa l'unità atomica del sistema. Ogni "frame" del buffer contiene:
```
EventRecord:
  timestamp_sim   ← world.get_snapshot().timestamp.elapsed_seconds
  timestamp_wall  ← time.time()
  camera_frame    ← np.ndarray BGR (da image_to_bgr)
  vehicle_state:
    location      ← vehicle.get_location()
    velocity      ← vehicle.get_velocity()
    acceleration  ← vehicle.get_acceleration()
    control       ← vehicle.get_control()  (throttle, steer, brake)
  warnings        ← lista stringhe (es. "COLLISION_IMMINENT")
  mqtt_messages   ← lista payload ricevuti nell'ultimo tick
  reasoning_text  ← str | None  (da explanation agent)
```

Il timestamp di simulazione è la chiave di allineamento: tutto viene sincronizzato su di esso perché in modalità sync (`world.tick()` restituisce un `WorldSnapshot` con timestamp univoco).

## Punto 2 — Ring buffer da 30 secondi + flush su evento
Struttura dati: `collections.deque(maxlen=N)` dove `N = 30s / fixed_dt`.

Con `fixed_dt = 0.05s` → `maxlen = 600` record.
```
Flusso per tick:
  1. world.tick() → snapshot
  2. leggi camera_frame dalla queue del listener
  3. leggi vehicle_state via get_velocity() / get_control()
  4. leggi mqtt_messages accumulati dal subscriber MQTT
  5. costruisci EventRecord con timestamp_sim
  6. deque.append(record)   ← il più vecchio cade automaticamente
```

Trigger di flush — un thread separato monitora:
  - Collisione: `sensor.other.collision` listener → imposta un flag `collision_event`
  - Near miss: distanza LiDAR < soglia → imposta `near_miss_event`
  - Manuale: keypress / messaggio MQTT specifico
```
Quando il flag è alzato:
snapshot_list = list(deque)          # copia atomica
serialize_to_disk(snapshot_list)     # vedi punto 3
```
La serializzazione deve avvenire in un thread separato per non bloccare il loop di simulazione.


## Punto 3 — File per sessione + viewer
Organizzazione su disco:
```
session_2026-04-28T14:32:00/
  metadata.json          ← mappa, meteo, durata, trigger type
  events.jsonl           ← un JSON per record (vehicle_state, warnings, mqtt, reasoning)
  frames/
    000001.jpg           ← camera_frame compressa
    000002.jpg
    ...
  index.json             ← mapping frame_id → timestamp_sim
```
I frame vengono salvati separatamente (JPEG ~10-20 KB) per tenere events.jsonl leggero e navigabile.

**Viewer con timeline**:

Opzione desktop (più semplice con i tool già usati):
```
OpenCV + pygame:
  - barra timeline orizzontale (click → seek a timestamp)
  - pannello sinistro: warnings + reasoning_text (eventuale) (una spiegazione in linguaggio naturale o semi‑strutturata generata dall’“explanation agent” che motiva una decisione, un warning o uno stato osservato, es. “freno perché ostacolo a 8 m”) dell'istante corrente
  - pannello destro: grafici speed/brake/steer via matplotlib in-memory
  - in alto a dx: icona per cambiare camera
```

**Schema complessivo dei thread**
```
Thread MAIN      → world.tick() + costruzione EventRecord + append deque
Thread MQTT      → subscriber paho-mqtt, accumula messaggi in una queue
Thread AGENT     → inference del reasoning, input/output queue
Thread FLUSHER   → in attesa di flag, serializza su disco in background
Thread VIEWER    → (opzionale live) legge deque e mostra OpenCV
```
Tutti condividono la `deque` protetta da un `threading.Lock` durante la copia per il flush.


