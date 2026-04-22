# nodered-aula-messages

**Dansk** | [English](#english)

---

## Dansk

### Hvad gør dette projekt?

Et Node-RED flow der henter alle beskedtråde fra [AULA](https://www.aula.dk) – den danske kommunikationsplatform for folkeskoler – og publicerer dem via MQTT.

Flowet bruger [Scaarups Home Assistant AULA-integration](https://github.com/scaarup/aula)'s `aula.api_call` service som proxy. Det betyder at MitID-autentificeringen håndteres ét sted (i HA), og Node-RED blot kalder HA's REST API med et Long-Lived Token. Ingen AULA-session eller OAuth-tokens håndteres i Node-RED.

```
[Node-RED] → HA REST API → aula.api_call service → AULA API (via HA's session)
                ↓
           [MQTT Broker] → Home Assistant / andre modtagere
```

**Publicerede MQTT-emner (standard præfiks: `aula/messages`):**

| Emne | Indhold | Retained |
|------|---------|----------|
| `aula/messages` | Opsummering af alle tråde | ✓ |
| `aula/messages/status` | Heartbeat med antal tråde/ulæste | ✓ |
| `aula/messages/{threadId}` | Fuld tråd med alle beskeders tekst | ✓ |
| `aula/messages/unread` | Kun ulæste tråde (trigger til automationer) | ✗ |
| `aula/messages/bridge/state` | `online`/`offline` LWT-besked | ✓ |

---

### Forudsætninger

1. **Node-RED** ≥ 3.0 med **Node.js ≥ 18** (kræves for den native `fetch` API)
2. **[Scaarups HA AULA-integration](https://github.com/scaarup/aula)** installeret og succesfuldt autentificeret via MitID i Home Assistant
3. **Home Assistant** tilgængeligt på netværket fra Node-RED (f.eks. `http://192.168.1.240:8123`)
4. En **MQTT-broker** (fx Mosquitto) tilgængelig fra Node-RED

---

### Opsætning

#### 1. Verificér at AULA-integrationen virker i HA

Gå til **HA → Udviklerværktøjer → Services** og kald:
```yaml
service: aula.api_call
data:
  uri: "?method=messaging.getThreads&sortOn=date&orderDirection=desc&page=0"
return_response: true
```
Hvis du ser beskedtråde i svaret, er integrationen klar.

#### 2. Opret et Long-Lived Access Token i HA

1. HA → **Profil** (klik på dit navn nederst til venstre)
2. Scroll ned til **Sikkerhed** → **Long-Lived Access Tokens**
3. Klik **Opret token**, giv det et navn (f.eks. "Node-RED AULA")
4. Kopiér tokenet — det vises kun én gang

#### 3. Sæt miljøvariabler i Node-RED

Tilføj til Node-REDs miljø (fx i `settings.js`, din container, eller HA's Node-RED addon-konfiguration):

```bash
HA_URL=http://192.168.1.240:8123   # Erstat med din HA-adresse
HA_TOKEN=eyJhbGci...               # Dit Long-Lived Access Token
```

Valgfri:
```bash
MQTT_TOPIC_PREFIX=aula/messages    # Standard: aula/messages
```

> **HA addon:** I Node-RED addons konfiguration under *Environment Variables*.

#### 4. Importér flowet i Node-RED

1. Åbn Node-RED → **Hamburgermenu → Import**
2. Vælg `flows.json` fra dette repository (eller indsæt indholdet)
3. Klik **Import**

#### 5. Konfigurér MQTT-broker

1. Dobbeltklik på noden **AULA MQTT Broker** (i config-node-listen)
2. Ret **host** og **port** til din MQTT-broker
3. Tilføj evt. brugernavn/adgangskode

#### 6. Deploy og verificér

1. Klik **Deploy**
2. Tjek status-indikatorer efter 5 sekunder (første poll):
   - **HA API Fetcher** → "N tråde, M ulæste" (grøn)
3. Abonnér på MQTT for at verificere:
   ```bash
   mosquitto_sub -h localhost -t "aula/messages/#" -v
   ```

---

### Eksempel på MQTT-output

**`aula/messages` (opsummering):**
```json
{
  "timestamp": "2026-04-22T10:30:00.000Z",
  "threadCount": 5,
  "unreadCount": 2,
  "threads": [
    {
      "id": 123456,
      "subject": "Forældremøde d. 5. maj",
      "read": false,
      "messageCount": 3
    }
  ]
}
```

**`aula/messages/123456` (enkelt tråd):**
```json
{
  "id": 123456,
  "subject": "Forældremøde d. 5. maj",
  "read": false,
  "messageCount": 3,
  "messages": [
    {
      "id": 789012,
      "sendDateTime": "2026-04-21T14:00:00.000Z",
      "sender": "Lars Hansen",
      "text": "Kære forældre, vi holder forældremøde tirsdag d. 5. maj kl. 19.",
      "messageType": "Message"
    }
  ]
}
```

**`aula/messages/status` (heartbeat):**
```json
{
  "timestamp": "2026-04-22T10:30:00.000Z",
  "threadCount": 5,
  "unreadCount": 2
}
```

---

### Konfigurationsmuligheder

| Miljøvariabel | Standard | Beskrivelse |
|---------------|----------|-------------|
| `HA_URL` | `http://homeassistant.local:8123` | Home Assistant base URL |
| `HA_TOKEN` | *(påkrævet)* | HA Long-Lived Access Token |
| `MQTT_TOPIC_PREFIX` | `aula/messages` | Præfiks for alle MQTT-emner |

Polling-interval sættes i **Inject-noden** (standard: 300 sekunder / 5 minutter).

---

### Fejlfinding

| Problem | Løsning |
|---------|---------|
| **HA API Fetcher** viser "No HA_TOKEN" | Sæt `HA_TOKEN` miljøvariablen |
| **HA API Fetcher** viser "HTTP 401" | Tokenet er ugyldigt — opret et nyt i HA |
| **HA API Fetcher** viser "HTTP 404" | AULA-integrationen er ikke installeret i HA, eller `aula.api_call` service eksisterer ikke |
| **HA API Fetcher** viser "HTTP 500" | AULA-integrationens session er udløbet — genautentificer i HA |
| **HA API Fetcher** viser "Unexpected getThreads response" | AULA returnerede en uventet struktur — tjek HA-loggen |
| MQTT-forbindelsen fejler | Tjek host/port i MQTT-broker config-noden |

---

### Sikkerhed

- **HA_TOKEN** og **MQTT-credentials** gemmes aldrig i `flows.json` — kun i Node-REDs miljøvariabler
- HA Long-Lived Tokens giver bred adgang til HA — overvej at bruge en dedikeret HA-bruger med minimale rettigheder
- MQTT-topics er retained — beskeder forbliver i brokeren indtil næste opdatering

---

### Bidrag

Issues og pull requests modtages gerne på [GitHub](https://github.com/Dokport/nodered-aula-messages/issues).

---

## English

### What does this project do?

A Node-RED flow that fetches all message threads from [AULA](https://www.aula.dk) – the Danish school communication platform – and publishes them via MQTT.

The flow uses [Scaarup's Home Assistant AULA integration](https://github.com/scaarup/aula)'s `aula.api_call` service as a proxy. MitID authentication is handled by Home Assistant; Node-RED simply calls the HA REST API using a Long-Lived Token. No AULA session or OAuth tokens are managed in Node-RED.

```
[Node-RED] → HA REST API → aula.api_call service → AULA API (via HA's session)
                ↓
           [MQTT Broker] → Home Assistant / other consumers
```

**Published MQTT topics (default prefix: `aula/messages`):**

| Topic | Content | Retained |
|-------|---------|----------|
| `aula/messages` | Summary of all threads | ✓ |
| `aula/messages/status` | Heartbeat with thread/unread counts | ✓ |
| `aula/messages/{threadId}` | Full thread with all message text | ✓ |
| `aula/messages/unread` | Unread threads only (for automations) | ✗ |
| `aula/messages/bridge/state` | `online`/`offline` LWT message | ✓ |

---

### Prerequisites

1. **Node-RED** ≥ 3.0 with **Node.js ≥ 18** (required for the native `fetch` API)
2. **[Scaarup's HA AULA integration](https://github.com/scaarup/aula)** installed and successfully authenticated via MitID in Home Assistant
3. **Home Assistant** reachable on the network from Node-RED (e.g. `http://192.168.1.240:8123`)
4. An **MQTT broker** (e.g. Mosquitto) reachable from Node-RED

---

### Setup

#### 1. Verify the AULA integration works in HA

Go to **HA → Developer Tools → Services** and call:
```yaml
service: aula.api_call
data:
  uri: "?method=messaging.getThreads&sortOn=date&orderDirection=desc&page=0"
return_response: true
```
If you see message threads in the response, the integration is ready.

#### 2. Create a Long-Lived Access Token in HA

1. HA → **Profile** (click your name in the bottom left)
2. Scroll to **Security** → **Long-Lived Access Tokens**
3. Click **Create Token**, give it a name (e.g. "Node-RED AULA")
4. Copy the token — it is shown only once

#### 3. Set environment variables in Node-RED

Add to Node-RED's environment (e.g. in `settings.js`, your container, or the HA Node-RED add-on config):

```bash
HA_URL=http://192.168.1.240:8123   # Replace with your HA address
HA_TOKEN=eyJhbGci...               # Your Long-Lived Access Token
```

Optional:
```bash
MQTT_TOPIC_PREFIX=aula/messages    # Default: aula/messages
```

#### 4. Import the flow into Node-RED

1. Open Node-RED → **Hamburger menu → Import**
2. Select `flows.json` from this repository (or paste the contents)
3. Click **Import**

#### 5. Configure the MQTT broker

1. Double-click the **AULA MQTT Broker** config node
2. Set the **host** and **port** for your MQTT broker
3. Add credentials if required

#### 6. Deploy and verify

1. Click **Deploy**
2. Check the status indicators after 5 seconds (first poll):
   - **HA API Fetcher** → "N threads, M unread" (green)
3. Subscribe to verify output:
   ```bash
   mosquitto_sub -h localhost -t "aula/messages/#" -v
   ```

---

### MQTT Output Example

**`aula/messages` (summary):**
```json
{
  "timestamp": "2026-04-22T10:30:00.000Z",
  "threadCount": 5,
  "unreadCount": 2,
  "threads": [
    {
      "id": 123456,
      "subject": "Parent meeting May 5th",
      "read": false,
      "messageCount": 3
    }
  ]
}
```

**`aula/messages/123456` (individual thread):**
```json
{
  "id": 123456,
  "subject": "Parent meeting May 5th",
  "read": false,
  "messageCount": 3,
  "messages": [
    {
      "id": 789012,
      "sendDateTime": "2026-04-21T14:00:00.000Z",
      "sender": "Lars Hansen",
      "text": "Dear parents, we are holding a parent meeting on Tuesday May 5th at 7pm.",
      "messageType": "Message"
    }
  ]
}
```

**`aula/messages/status` (heartbeat):**
```json
{
  "timestamp": "2026-04-22T10:30:00.000Z",
  "threadCount": 5,
  "unreadCount": 2
}
```

---

### Configuration

| Environment Variable | Default | Description |
|---------------------|---------|-------------|
| `HA_URL` | `http://homeassistant.local:8123` | Home Assistant base URL |
| `HA_TOKEN` | *(required)* | HA Long-Lived Access Token |
| `MQTT_TOPIC_PREFIX` | `aula/messages` | Prefix for all MQTT topics |

The polling interval is set in the **Inject node** (default: 300 seconds / 5 minutes).

---

### Troubleshooting

| Problem | Solution |
|---------|---------|
| **HA API Fetcher** shows "No HA_TOKEN" | Set the `HA_TOKEN` env var |
| **HA API Fetcher** shows "HTTP 401" | Token is invalid — create a new one in HA |
| **HA API Fetcher** shows "HTTP 404" | AULA integration not installed, or `aula.api_call` service not found |
| **HA API Fetcher** shows "HTTP 500" | AULA integration session expired — re-authenticate in HA |
| **HA API Fetcher** shows "Unexpected getThreads response" | AULA returned unexpected structure — check HA logs |
| MQTT connection fails | Check host/port in the MQTT broker config node |

---

### How it works

```
[Inject (every 5 min)]
       ↓
[HA API Fetcher]     — POSTs to HA REST API: /api/services/aula/api_call?return_response
       ↓                Calls messaging.getThreads (paginated) then
       ↓                messaging.getMessagesForThread per thread (paginated)
       ↓                HA uses its active MitID-authenticated AULA session
[MQTT Formatter]     — splits data into per-topic MQTT messages
       ↓
[MQTT Out]           — publishes to broker
```

The `aula.api_call` service in Scaarup's integration is registered with `supports_response=SupportsResponse.ONLY`, meaning it always returns the AULA API response directly. HA's REST API wraps it as `{"service_response": <aula_response>}`.

---

### Security

- **HA_TOKEN** and MQTT credentials are never stored in `flows.json` — only in Node-RED environment variables
- HA Long-Lived Tokens grant broad HA access — consider using a dedicated HA user with minimal permissions
- MQTT topics are retained — messages persist in the broker until the next poll update

---

### Contributing

Issues and pull requests welcome at [GitHub](https://github.com/Dokport/nodered-aula-messages/issues).

---

### License

MIT — see [LICENSE](LICENSE)

### Acknowledgements

- [scaarup/aula](https://github.com/scaarup/aula) — Home Assistant integration providing the `aula.api_call` service that this flow relies on
- [zinen/node-red-contrib-aula-education](https://github.com/zinen/node-red-contrib-aula-education) — original Node-RED AULA node (UNI-login era), API call patterns referenced
