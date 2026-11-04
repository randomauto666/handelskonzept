# Trade-System – Konzept

> Sicherer Spieler-zu-Spieler-Handel per `/trade` für OPSUCHT (Paper 1.21.11)
> GUI im Button-Stil (Bedrock-kompatibel, wie beim Mail System)

---

## 1. Ziele

- Tauschen von Items und Geld **ohne Scam-Risiko** (kein „Drop & Pray")
- Einheitliche Bedienung für **Java und Bedrock** (Geyser/Floodgate)
- Vollständiges **Logging**, damit Streitfälle nachvollziehbar sind
- Gut anbindbar an das Report-System (Trade-Log als Beweis)

---

## 2. Befehle

| Befehl | Beschreibung | Permission |
|---|---|---|
| `/trade <Spieler>` | Handelsanfrage senden | `trade.use` |
| `/trade accept [Spieler]` | Anfrage annehmen | `trade.use` |
| `/trade deny [Spieler]` | Anfrage ablehnen | `trade.use` |
| `/trade cancel` | Laufenden Handel abbrechen | `trade.use` |
| `/trade toggle` | Anfragen an/aus | `trade.use` |
| `/trade block <Spieler>` | Spieler blockieren | `trade.use` |
| `/trade log <Spieler>` | Trade-Historie einsehen | `trade.admin` |
| `/trade reload` | Config neu laden | `trade.admin` |

---

## 3. Ablauf

```
Spieler A: /trade B
   │
   ▼
B bekommt Chat-Nachricht mit klickbaren [Annehmen] [Ablehnen] (60s gültig)
   │
   ▼
Beide Spieler sehen das Trade-Menü
   │
   ▼
Items/Geld hinzufügen → beide drücken [Bestätigen]
   │
   ▼
Bestätigungs-Screen (3s Countdown, Inhalt final)
   │
   ▼
Tausch wird atomar ausgeführt → Log-Eintrag
```

**Voraussetzungen für eine Anfrage:**
- Beide Spieler online, gleicher Server (z.b. CB01)
- Max. Distanz (Standard 30 Blöcke) – optional abschaltbar
- Ziel hat Anfragen nicht deaktiviert oder den Sender blockiert (/ignore)
- Cooldown zwischen Anfragen (Standard 10s)

---

## 4. GUI-Design

### 4.1 Java: Inventar-GUI (6 Reihen)

```
┌───────────────────────────────────────────┐
│ OPSUCHT » Trade mit B                     │
├────────────────┬──┬───────────────────────┤
│  DEINE ITEMS   │▒▒│  ITEMS VON B          │
│  [ ][ ][ ][ ]  │▒▒│  [ ][ ][ ][ ]         │
│  [ ][ ][ ][ ]  │▒▒│  [ ][ ][ ][ ]         │
│  [ ][ ][ ][ ]  │▒▒│  [ ][ ][ ][ ]         │
│  [ ][ ][ ][ ]  │▒▒│  [ ][ ][ ][ ]         │
├────────────────┼──┼───────────────────────┤
│ [💰 Geld]  [✔ Bestätigen]  [✖ Abbrechen] │
└────────────────┴──┴───────────────────────┘
```

- Mittlere Spalte = Trennlinie (Glasscheiben)
- Status-Item zeigt pro Seite: ⏳ *wartet* / ✔ *bestätigt*
- **Jede Änderung setzt beide Bestätigungen zurück**

### 4.2 Bedrock: Button-Formulare

Bedrock kann keine Drag-and-Drop-Inventare sinnvoll bedienen, daher läuft alles über **Forms (Buttons)**:

**Hauptmenü** (Titel: `OPSUCHT » Trade mit B`)

```
[ Item hinzufügen        ]
[ Geld festlegen         ]
[ Aktuelles Angebot      ]
[ Bestätigen             ]
[ Abbrechen              ]
```

**Item hinzufügen** → Liste der Inventar-Items als Buttons (Name + Menge)
→ danach **Slider/Input** für die Menge.

**Aktuelles Angebot** → Textübersicht beider Seiten:

```
Du bietest:      B bietet:
 • 32x Diamant    • 5.000 $
 • 1x Elytra      • 16x Netherit-Barren
```

**Bestätigen** → Abschluss-Form mit zwei Buttons `[Ja, Handel abschließen]` / `[Zurück]`.

> Erkennung via Floodgate (`FloodgateApi.isFloodgatePlayer(uuid)`).
> Java-Spieler können optional ebenfalls das Button-Menü nutzen (`/trade menu`).

---

## 5. Sicherheitsmechanismen

| Schutz | Beschreibung |
|---|---|
| **Atomarer Tausch** | Beide Inventare werden vor dem Tausch geprüft, dann in einem Schritt übertragen. Schlägt etwas fehl → Rollback |
| **Platzprüfung** | Handel wird verweigert, wenn das Inventar des Empfängers nicht reicht |
| **Reset bei Änderung** | Jede Änderung am Angebot entfernt beide Bestätigungen |
| **Bestätigungs-Countdown** | 3 Sekunden Sperre nach beidseitiger Bestätigung |
| **Item-Snapshot** | Items werden per Hash/NBT verglichen – kein Austausch in letzter Sekunde |
| **Verbotene Items** | Blacklist in der Config (z. B. Command-Blocks, bestimmte Custom-Items) |
| **Kampf/Tod/Disconnect** | Handel bricht ab, Items gehen zurück |
| **Crash-Schutz** | Items werden vorab in einer DB-Tabelle gesichert und beim nächsten Join zurückgegeben |
| **Geld-Limit** | Min/Max pro Trade konfigurierbar |
| **Warnung bei Wertgefälle** | Hinweis, wenn eine Seite deutlich mehr gibt als bekommt (optional) |

---

## 6. Wirtschafts-Integration

- Anbindung über **Vault** (`Economy`-Interface) → OPSUCHT Wirtschaftssystem
- Optionale **Handelssteuer** (z. B. 2 %) als Geldsenke
- Geld wird erst beim finalen Tausch abgebucht, nicht beim Eintragen

---

## 7. Datenbank

**Tabelle `trade_log`**

| Spalte | Typ | Beschreibung |
|---|---|---|
| `id` | INT PK | Trade-ID |
| `player_a` | UUID | Initiator |
| `player_b` | UUID | Partner |
| `items_a` | TEXT (Base64/JSON) | Items von A |
| `items_b` | TEXT (Base64/JSON) | Items von B |
| `money_a` | DOUBLE | Geld von A |
| `money_b` | DOUBLE | Geld von B |
| `status` | ENUM | `COMPLETED`, `CANCELLED`, `FAILED` |
| `created_at` | TIMESTAMP | Zeitpunkt |

**Tabelle `trade_pending`** – gesicherte Items für Crash-Rückgabe
**Tabelle `trade_settings`** – pro Spieler: `enabled`, Blockliste

---

## 8. Konfiguration (Auszug `config.yml`)

```yaml
trade:
  request-timeout: 60        # Sekunden
  request-cooldown: 10
  max-distance: 30           # -1 = unbegrenzt
  confirm-countdown: 3
  tax-percent: 2.0
  min-money: 1
  max-money: 100000000
  cancel-on-combat: true
  blacklist:
    - COMMAND_BLOCK
    - BEDROCK
    - BARRIER
  warn-value-difference: true
```

`messages.yml` mit **MiniMessage** (Gradients, Hover/Click für Annehmen/Ablehnen).

---

## 9. Permissions

| Permission | Beschreibung |
|---|---|
| `trade.use` | `/trade` nutzen |
| `trade.bypass.distance` | Distanzlimit ignorieren |
| `trade.bypass.cooldown` | Cooldown ignorieren |
| `trade.admin` | Logs einsehen, Reload |
| `trade.spy` | Laufende Trades live ansehen (Teamintern) |

---

## 10. Technische Umsetzung

- **Paper 1.21.11**, Java 21
- Package: `de.opsucht.trade`
- Struktur:

```
de.opsucht.trade
├── TradePlugin.java
├── command/TradeCommand.java
├── session/TradeSession.java        # Zustand eines Handels
├── session/TradeManager.java        # Anfragen & aktive Sessions
├── gui/JavaTradeGui.java            # Inventar-GUI
├── gui/BedrockTradeForm.java        # Floodgate-Forms
├── storage/TradeRepository.java     # DB (SQLite/MySQL)
├── economy/EconomyHook.java         # Vault
└── util/ItemSerializer.java
```

- Abhängigkeiten: Paper-API, Vault, Floodgate (soft-depend), Adventure/MiniMessage
- Asynchrone DB-Zugriffe, Tausch selbst auf dem Main-Thread

---

## 11. Erweiterungen (optional)

- Trade-Historie als GUI (`/trade history`)
- Bestätigungsregel für wertvolle Items (extra Warn-Screen)
- Discord-Webhook bei Trades über bestimmtem Wert
- Anbindung an das Report-System: *„Report wegen Scam"* hängt automatisch die letzten Trades an
- Statistik: Anzahl Trades, Gesamtwert pro Spieler

---

## 12. Roadmap

1. **MVP:** `/trade`, Anfrage/Annahme, Java-GUI, atomarer Tausch
2. Geld-Handel (Vault), Config, Messages
3. Bedrock-Forms
4. Logging + Admin-Befehle
5. Crash-Schutz, Blacklist, Steuer
6. Feinschliff & Tests (Disconnect, volle Inventare, Doppelklick-Exploits)
