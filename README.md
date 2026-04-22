# nodered-aula-messages

**Dansk** | [English](#english)

---

## Dansk

### Hvad gør dette projekt?

Et Node-RED flow der poller [AULA](https://www.aula.dk) – den danske kommunikationsplatform for folkeskoler – og publicerer alle beskedtråde og deres indhold via MQTT.

Flowet er baseret på AULA's interne REST API (v22) og bruger den OAuth-token som allerede er etableret af [Scaarups Home Assistant AULA-integration](https://github.com/scaarup/aula). På den måde genimplementeres MitID-login-flowet ikke – i stedet genbruges den eksisterende session.

**Publicerede MQTT-emner (standard præfiks: `aula/messages`):**

| Emne | Indhold | Retained |
|------|---------|----------|
| `aula/messages` | Opsummering af alle tråde | ✓ |
| `aula/messages/status` | Heartbeat med antal tråde/ulæste | ✓ |
| `aula/messages/{threadId}` | Fuld tråd med alle beskeder | ✓ |
| `aula/messages/unread` | Kun ulæste tråde (trigger til automationer) | ✗ |
| `aula/messages/bridge/state` | `online`/`offline` LWT-besked | ✓ |

---

### Forudsætninger

1. **Node-RED** ≥ 3.0 med **Node.js ≥ 18** (kræves for den native `fetch` API)
2. **[Scaarups HA AULA-integration](https://github.com/scaarup/aula)** installeret og autentificeret via MitID i Home Assistant
3. En **MQTT-broker** (fx Mosquitto) tilgængelig fra Node-RED

---

### Opsætning

#### 1. Hent refresh-token fra Home Assistant

Scaarups integration gemmer OIDC-tokens i HA's konfigurationslagring. Der er to metoder:

**Metode A – HA-filsystem (enkel):**
```bash
# SSH til din HA-instans og find AULA's config entry:
cat /config/.storage/core.config_entries | python3 -m json.tool \
  | grep -A 30 '"domain": "aula"'
# Find feltet "refresh_token" under "data"
```

**Metode B – HA Template Sensor (ingen SSH nødvendig):**

Tilføj til din `configuration.yaml`:
```yaml
template:
  - sensor:
      - name: "AULA Refresh Token"
        state: >
          {{ state_attr('sensor.aula_<dit_barn>', 'refresh_token') }}
```
> Bemærk: Eksponér aldrig token via et offentligt HA-dashboard.

#### 2. Sæt miljøvariabel i Node-RED

Tilføj til Node-REDs miljø (fx i `settings.js` eller via din container/addon):
```bash
AULA_REFRESH_TOKEN=<din_refresh_token>
```

Valgfrie variabler:
```bash
AULA_API_VERSION=22          # AULA API-version (standard: 22)
MQTT_TOPIC_PREFIX=aula/messages  # MQTT-emnepræfiks
```

#### 3. Importér flowet i Node-RED

1. Åbn Node-RED → **Hamburgermenu → Import**
2. Vælg `flows.json` fra dette repository
3. Klik **Import**

#### 4. Konfigurér MQTT-broker

1. Dobbeltklik på noden **AULA MQTT Broker** (grå config-node i listen)
2. Ret host og port til din MQTT-broker
3. Tilføj evt. brugernavn/adgangskode

#### 5. Deploy og verificér

1. Klik **Deploy**
2. Tjek status-indikatorerne på noderne:
   - `Token Manager` → "Token refreshed" (grøn)
   - `AULA Poller` → "N threads, M unread" (grøn)
3. Abonnér på MQTT-emnet for at verificere output:
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
      "text": "Kære forældre, vi holder forældremøde...",
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
| `AULA_REFRESH_TOKEN` | *(påkrævet)* | OIDC refresh-token fra HA AULA-integration |
| `AULA_API_VERSION` | `22` | AULA REST API-version |
| `MQTT_TOPIC_PREFIX` | `aula/messages` | Præfiks for alle MQTT-emner |

Polling-interval sættes i **Inject-noden** (standard: 300 sekunder / 5 minutter).

---

### Fejlfinding

| Problem | Løsning |
|---------|---------|
| `Token Manager` viser "No refresh token" | Sæt `AULA_REFRESH_TOKEN` env-variablen korrekt |
| `Token Manager` viser "Refresh error" | Token er udløbet/ugyldig – hent nyt token fra HA |
| `AULA Poller` viser "HTTP 401" | Access-token er ugyldig – refresh fejlede stille |
| `AULA Poller` viser "HTTP 410" | API-versionen er forældet – sæt `AULA_API_VERSION` til en højere version |
| MQTT-forbindelsen fejler | Tjek host/port i MQTT-broker config-noden |

---

### Bidrag

Issues og pull requests modtages gerne på [GitHub](https://github.com/Dokport/nodered-aula-messages/issues).

---

## English

### What does this project do?

A Node-RED flow that polls [AULA](https://www.aula.dk) – the Danish school communication platform – and publishes all message threads and their content via MQTT.

The flow uses AULA's internal REST API (v22) and borrows the OAuth token already established by [Scaarup's Home Assistant AULA integration](https://github.com/scaarup/aula), avoiding any need to re-implement the MitID login flow.

**Published MQTT topics (default prefix: `aula/messages`):**

| Topic | Content | Retained |
|-------|---------|----------|
| `aula/messages` | Summary of all threads | ✓ |
| `aula/messages/status` | Heartbeat with thread/unread counts | ✓ |
| `aula/messages/{threadId}` | Full thread with all messages | ✓ |
| `aula/messages/unread` | Unread threads only (for automations) | ✗ |
| `aula/messages/bridge/state` | `online`/`offline` LWT message | ✓ |

---

### Prerequisites

1. **Node-RED** ≥ 3.0 with **Node.js ≥ 18** (required for the native `fetch` API)
2. **[Scaarup's HA AULA integration](https://github.com/scaarup/aula)** installed and authenticated via MitID in Home Assistant
3. An **MQTT broker** (e.g. Mosquitto) reachable from Node-RED

---

### Setup

#### 1. Obtain the refresh token from Home Assistant

Scaarup's integration stores OIDC tokens in HA's configuration storage. Two methods:

**Method A – HA filesystem (simple):**
```bash
# SSH into your HA instance and find AULA's config entry:
cat /config/.storage/core.config_entries | python3 -m json.tool \
  | grep -A 30 '"domain": "aula"'
# Find the "refresh_token" field under "data"
```

**Method B – HA Template Sensor (no SSH required):**

Add to your `configuration.yaml`:
```yaml
template:
  - sensor:
      - name: "AULA Refresh Token"
        state: >
          {{ state_attr('sensor.aula_<your_child>', 'refresh_token') }}
```
> Warning: Never expose this token via a public HA dashboard.

#### 2. Set the environment variable in Node-RED

Add to Node-RED's environment (e.g. in `settings.js` or your container/add-on):
```bash
AULA_REFRESH_TOKEN=<your_refresh_token>
```

Optional variables:
```bash
AULA_API_VERSION=22               # AULA API version (default: 22)
MQTT_TOPIC_PREFIX=aula/messages   # MQTT topic prefix
```

#### 3. Import the flow into Node-RED

1. Open Node-RED → **Hamburger menu → Import**
2. Select `flows.json` from this repository
3. Click **Import**

#### 4. Configure the MQTT broker

1. Double-click the **AULA MQTT Broker** config node
2. Set the host and port for your MQTT broker
3. Add credentials if required

#### 5. Deploy and verify

1. Click **Deploy**
2. Check the status indicators on the nodes:
   - `Token Manager` → "Token refreshed" (green)
   - `AULA Poller` → "N threads, M unread" (green)
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
      "text": "Dear parents, we are holding a parent meeting...",
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

### Configuration Options

| Environment Variable | Default | Description |
|---------------------|---------|-------------|
| `AULA_REFRESH_TOKEN` | *(required)* | OIDC refresh token from HA AULA integration |
| `AULA_API_VERSION` | `22` | AULA REST API version |
| `MQTT_TOPIC_PREFIX` | `aula/messages` | Prefix for all MQTT topics |

The polling interval is set in the **Inject node** (default: 300 seconds / 5 minutes).

---

### Troubleshooting

| Problem | Solution |
|---------|---------|
| `Token Manager` shows "No refresh token" | Set the `AULA_REFRESH_TOKEN` env var correctly |
| `Token Manager` shows "Refresh error" | Token is expired/invalid — obtain a fresh one from HA |
| `AULA Poller` shows "HTTP 401" | Access token is invalid — token refresh silently failed |
| `AULA Poller` shows "HTTP 410" | API version is outdated — set `AULA_API_VERSION` to a higher number |
| MQTT connection fails | Check host/port in the MQTT broker config node |

---

### How it works

```
[Inject (interval)]
       ↓
[Token Manager]      — reads AULA_REFRESH_TOKEN env var, calls OIDC token endpoint,
       ↓                stores access_token in flow context, auto-refreshes before expiry
[AULA Poller]        — fetches all thread pages via messaging.getThreads, then fetches
       ↓                messages per thread via messaging.getMessagesForThread
[MQTT Formatter]     — splits data into per-topic MQTT messages
       ↓
[MQTT Out]           — publishes to broker
```

Token management: the initial `AULA_REFRESH_TOKEN` env var seeds the first refresh. After that, the new refresh token returned by the OIDC endpoint is stored in Node-RED flow context and used for subsequent refreshes. The env var acts only as a seed — it is never stored in `flows.json`.

---

### Contributing

Issues and pull requests welcome at [GitHub](https://github.com/Dokport/nodered-aula-messages/issues).

---

### License

MIT — see [LICENSE](LICENSE)

### Acknowledgements

- [scaarup/aula](https://github.com/scaarup/aula) — Home Assistant integration that solved the MitID auth flow
- [zinen/node-red-contrib-aula-education](https://github.com/zinen/node-red-contrib-aula-education) — original Node-RED AULA node (UNI-login era)
