# Bericht OpenClaw
---
**Thema:** OpenClaw
**Datum:** 29.07.2026 - 31.07.2026
---
### Aufgabenstellung

- OpenClaw installieren, mit OpenRouter verbinden, AnythingLLM mit OpenRouter verbinden, OpenClaw mit Discord verbinden.

---


### 1. Installation OpenClaw

Installation schlug zuerst fehl (Berechtigungsfehler):
```bash
npm install -g openclaw@latest
```

Fix: eigenen npm-Ordner einrichten
```bash
mkdir -p ~/.npm-global
npm config set prefix ~/.npm-global
echo 'export PATH=~/.npm-global/bin:$PATH' >> ~/.bashrc
source ~/.bashrc
```

Danach erfolgreich installiert:
```bash
npm install -g openclaw@latest
```

### 2. OpenClaw mit OpenRouter verbinden

Setup gestartet:
```bash
openclaw onboard --install-daemon
```
- Gateway-Bindung: Loopback (127.0.0.1)
- AI Provider: OpenRouter, API-Key eingegeben
- Modell: `openrouter/auto`

Dienst läuft als:
```bash
systemctl --user status openclaw-gateway
```

Getestet mit:
```bash
openclaw tui
```

Später Fehler HTTP 401 (Key ungültig), behoben mit:
```bash
openclaw configure --section provider
systemctl --user restart openclaw-gateway
```

### 3. AnythingLLM mit OpenRouter verbinden

In den AnythingLLM-Settings (`http://localhost:3001`) unter **LLM Preference**:
- Provider: OpenRouter
- API-Key eingetragen
- Modell aus Liste gewählt

### 4. OpenClaw mit Discord verbinden

Discord-Plugin installieren:
```bash
openclaw plugins install clawhub:@openclaw/discord
```

Bot-Token eintragen:
```bash
openclaw configure --section channels
```

Gateway neu starten:
```bash
systemctl --user restart openclaw-gateway
```

Getestet in Discord mit `@Reachy-Bot` – Bot antwortet.

### Ergebnis

OpenClaw läuft, ist mit OpenRouter verbunden und über Discord ansprechbar. AnythingLLM ist unabhängig davon ebenfalls mit OpenRouter verbunden.
