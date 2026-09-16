# Lokale KI auf einem MacBook M3 installieren

Das Ziel ist ein vollständig lokaler Stack aus **oMLX als Inference-Server**, einem **Qwen-Modell mit MTP**, **Open WebUI für Chats** und **OpenCode als Coding-Agent**.

> **Wichtige Begriffsklärung:**
> **MTP** bedeutet *Multi-Token Prediction* und beschleunigt die Textgenerierung. **MCP** bedeutet *Model Context Protocol* und bindet Werkzeuge oder Datenquellen an. Für die Modellgeschwindigkeit braucht ihr MTP. MCP ist optional.

---

## 1. Voraussetzungen prüfen

Empfohlen:

- Apple Silicon M3, M3 Pro oder M3 Max
- macOS 15 oder neuer
- mindestens 16 GB Unified Memory
- mindestens 30 bis 50 GB freier SSD-Speicher
- für ein 4B- oder 9B-Modell reichen meistens 16 bis 24 GB RAM
- für 27B-Modelle sollten mindestens 32 GB, besser 48 GB vorhanden sein

Im Terminal prüfen:

```bash
uname -m
sw_vers
system_profiler SPHardwareDataType | grep -E "Chip|Memory"
df -h /
```

Erwartet wird bei `uname -m`:

```text
arm64
```

---

## 2. Grundwerkzeuge installieren

### 2.1 Homebrew

Falls Homebrew noch nicht installiert ist:

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

Danach bei einem Apple-Silicon-Mac gegebenenfalls Homebrew in die Shell aufnehmen:

```bash
echo 'eval "$(/opt/homebrew/bin/brew shellenv)"' >> ~/.zprofile
eval "$(/opt/homebrew/bin/brew shellenv)"
```

Prüfen:

```bash
brew --version
```

### 2.2 Hilfsprogramme

```bash
brew install git curl jq
```

Für Open WebUI wird außerdem Docker Desktop empfohlen:

```bash
brew install --cask docker
```

Anschließend **Docker** einmal über Programme starten und warten, bis Docker betriebsbereit ist.

Prüfen:

```bash
docker version
```

---

## 3. oMLX installieren

### Empfohlene Variante: Homebrew

Für dieses Setup wird die Installation über Homebrew empfohlen. Dadurch lässt sich oMLX bequem über das Terminal installieren, als Hintergrunddienst starten und später mit Homebrew aktualisieren.

Zunächst das oMLX-Repository zu Homebrew hinzufügen:

```bash
brew tap jundot/omlx https://github.com/jundot/omlx
```

Anschließend oMLX installieren:

```bash
brew install jundot/omlx/omlx
```

Den oMLX-Dienst starten:

```bash
omlx start
```

Das standardmäßige Modellverzeichnis ist:

```text
~/.omlx/models
```

Das Admin-Interface ist anschließend erreichbar unter:

```text
http://localhost:8000/admin
```

Eine einfache eingebaute Chat-Oberfläche gibt es unter:

```text
http://localhost:8000/admin/chat
```

Status und API prüfen:

```bash
brew services info omlx
curl -s http://127.0.0.1:8000/v1/models | jq
```

Falls der Dienst nicht erreichbar ist, oMLX neu starten:

```bash
omlx restart
```

Die Protokolle können je nach Installation an einem der folgenden Orte liegen:

```bash
tail -f ~/.omlx/logs/server.log
```

Alternativ:

```bash
tail -f "$(brew --prefix)/var/log/omlx.log"
```

### Alternative: native macOS-App

Falls eine grafische Installation bevorzugt wird, kann stattdessen die signierte oMLX-App verwendet werden:

1. Die aktuelle `.dmg` von der offiziellen oMLX-Projektseite herunterladen.
2. oMLX nach **Programme** ziehen.
3. oMLX starten.
4. Als Modellverzeichnis `~/.omlx/models` übernehmen.
5. Den lokalen Server über die App starten.

> Es sollte nur eine Installationsvariante aktiv verwendet werden. Läuft oMLX gleichzeitig über Homebrew und über die App, kann es zu einem Portkonflikt auf Port `8000` kommen.

---

## 4. Passendes Qwen-Modell auswählen

Das Modell muss ausdrücklich:

1. im **MLX-Format** vorliegen,
2. zu oMLX passen,
3. einen **MTP-Checkpoint beziehungsweise integrierte MTP-Heads** enthalten,
4. einschließlich KV-Cache in den Unified Memory passen.

### Orientierung nach Arbeitsspeicher

| Unified Memory | Empfohlene Modellgröße |
|---|---:|
| 16 GB | Qwen 4B, quantisiert |
| 18 bis 24 GB | Qwen 4B oder 9B, quantisiert |
| 32 bis 36 GB | Qwen 9B oder eventuell 27B mit aggressiver Quantisierung |
| 48 GB oder mehr | Qwen 27B, quantisiert |
| 64 GB oder mehr | Qwen 27B komfortabel, längerer Kontext |

### Empfehlung für ein normales M3 MacBook

- **16 oder 18 GB:** Qwen 4B, 4-Bit oder oQ4, mit MTP
- **24 GB:** Qwen 9B, 4-Bit oder oQ4, mit MTP
- **36 GB:** zunächst Qwen 9B; 27B nur mit niedriger Quantisierung und begrenztem Kontext
- **48 GB oder mehr:** Qwen 27B, 4-Bit oder oQ4, mit MTP

### Modell über oMLX herunterladen

1. `http://localhost:8000/admin` öffnen.
2. Zu **Models** beziehungsweise **Download Model** wechseln.
3. Nach einem aktuellen Qwen-Modell suchen.
4. Darauf achten, dass im Modellnamen oder in der Beschreibung `MLX` und `MTP` vorkommen.
5. Modell herunterladen.
6. Das Modell in den Einstellungen öffnen.
7. **MTP** beziehungsweise **Lightning MTP** aktivieren, sofern nicht automatisch erkannt.
8. Für den ersten Test die Kontextlänge auf `16384` oder `32768` begrenzen.

> Nicht irgendein Qwen-MLX-Modell auswählen. Ein normales MLX-Modell ohne MTP-Heads bekommt durch Aktivieren eines Schalters nicht nachträglich MTP-Unterstützung.

---

## 5. oMLX und MTP testen

Zuerst nachsehen, welche Modell-ID der Server meldet:

```bash
curl -s http://127.0.0.1:8000/v1/models | jq -r '.data[].id'
```

Eine der ausgegebenen IDs als Variable setzen:

```bash
MODEL_ID="HIER-DIE-MODELL-ID-EINTRAGEN"
```

Testanfrage:

```bash
curl -s http://127.0.0.1:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d "{
    \"model\": \"${MODEL_ID}\",
    \"messages\": [
      {
        \"role\": \"user\",
        \"content\": \"Antworte mit genau einem kurzen deutschen Satz: Läuft das Modell lokal?\"
      }
    ],
    \"temperature\": 0.2,
    \"max_tokens\": 100
  }" | jq
```

Während der Anfrage im oMLX-Dashboard oder Log prüfen, ob MTP aktiv ist.

Logs bei einer Homebrew-Installation:

```bash
tail -f ~/.omlx/logs/server.log
```

Oder:

```bash
tail -f "$(brew --prefix)/var/log/omlx.log"
```

Nach Begriffen wie `MTP`, `native MTP`, `Lightning MTP`, `accepted tokens` oder `draft acceptance` suchen:

```bash
grep -iE "mtp|lightning|accepted|draft" ~/.omlx/logs/server.log | tail -50
```

---

## 6. Open WebUI installieren

Open WebUI läuft bequem als Docker-Container. Ein persistentes Volume speichert Benutzer, Chats und Einstellungen.

### 6.1 Schlüssel erzeugen (Optional)
Im Terminal
```bash
WEBUI_SECRET_KEY="$(openssl rand -hex 32)"
echo "$WEBUI_SECRET_KEY"
```

Den Wert sicher speichern. Er sollte bei späteren Container-Neuanlagen gleich bleiben.

### 6.2 Container starten

```bash
docker run -d \
  -p 3000:8080 \
  --add-host=host.docker.internal:host-gateway \
  -v open-webui:/app/backend/data \
  -e WEBUI_SECRET_KEY="$WEBUI_SECRET_KEY" \
  --name open-webui \
  --restart always \
  ghcr.io/open-webui/open-webui:main
```

Die Oberfläche öffnen:

```text
http://localhost:3000
```

Der erste angelegte Benutzer wird üblicherweise Administrator.

### 6.3 oMLX mit Open WebUI verbinden

In Open WebUI:

1. **Admin Panel** öffnen.
2. **Settings** auswählen.
3. **Connections** öffnen.
4. **OpenAI API** auswählen.
5. Eine neue Verbindung anlegen.

Eintragen:

```text
URL:
http://host.docker.internal:8000/v1

API key:
local-omlx
```

Der API-Key kann bei einem ausschließlich lokalen oMLX-Server ein beliebiger nichtleerer Wert sein, sofern in oMLX keine API-Key-Prüfung aktiviert wurde.

> Innerhalb eines Docker-Containers zeigt `localhost` auf den Container selbst. Deshalb muss hier `host.docker.internal` statt `localhost` verwendet werden.

Danach sollte das Qwen-Modell im Modellmenü erscheinen.

Falls es nicht erscheint:

```bash
docker logs --tail 100 open-webui
```

Verbindung aus dem Container testen:

```bash
docker exec open-webui \
  curl -s http://host.docker.internal:8000/v1/models
```

---

## 7. OpenCode installieren

Über Homebrew:

```bash
brew install anomalyco/tap/opencode
```

Alternativ:

```bash
curl -fsSL https://opencode.ai/install | bash
```

Prüfen:

```bash
opencode --version
```

---

## 8. OpenCode mit oMLX verbinden

Die globale Konfiguration liegt auf macOS unter:

```text
~/.config/opencode/opencode.json
```

Verzeichnis anlegen:

```bash
mkdir -p ~/.config/opencode
```

Modell-ID erneut abfragen:

```bash
curl -s http://127.0.0.1:8000/v1/models | jq -r '.data[].id'
```

Dann `~/.config/opencode/opencode.json` anlegen:

```json
{
  "$schema": "https://opencode.ai/config.json",
  "model": "omlx-local/HIER-DIE-MODELL-ID-EINTRAGEN",
  "provider": {
    "omlx-local": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "oMLX Local",
      "options": {
        "baseURL": "http://127.0.0.1:8000/v1",
        "apiKey": "local-omlx"
      },
      "models": {
        "HIER-DIE-MODELL-ID-EINTRAGEN": {
          "name": "Qwen lokal via oMLX"
        }
      }
    }
  }
}
```

Wichtig:

- Beide Vorkommen von `HIER-DIE-MODELL-ID-EINTRAGEN` müssen exakt durch die Ausgabe von `/v1/models` ersetzt werden.
- Die Modellreferenz folgt dem Muster `provider-id/model-id`.
- Die URL verwendet hier `127.0.0.1`, weil OpenCode nativ auf dem Mac und nicht im Docker-Container läuft.

JSON prüfen:

```bash
jq . ~/.config/opencode/opencode.json
```

---

## 9. OpenCode ausprobieren

In ein Testprojekt wechseln:

```bash
mkdir -p ~/Developer/opencode-test
cd ~/Developer/opencode-test
git init
printf '# OpenCode-Test\n' > README.md
opencode
```

In OpenCode:

```text
/models
```

Das lokale oMLX-Modell auswählen, falls es nicht bereits aktiv ist.

Danach:

```text
/init
```

OpenCode analysiert dabei das Projekt und erzeugt eine `AGENTS.md`, welche die Projektstruktur und Arbeitsregeln für den Coding-Agent beschreibt.

Erster ungefährlicher Test:

```text
Erstelle einen Plan für ein kleines Python-CLI-Programm, das eine
Markdown-Datei einliest und Überschriften ausgibt. Ändere noch keine Dateien.
```

Anschließend:

```text
Implementiere den Plan und erkläre jede erzeugte Datei kurz.
```

---

## 10. Empfohlene Sicherheitskonfiguration

### Nur lokal lauschen

oMLX sollte auf Folgendem gebunden bleiben:

```text
127.0.0.1:8000
```

Nicht ohne Authentifizierung auf `0.0.0.0` oder im Firmennetz freigeben.

### OpenCode-Rechte vorsichtig verwenden

Für die ersten Tests:

- Änderungen vor Ausführung kontrollieren
- Shell-Befehle bestätigen lassen
- keine Produktions-Repositories verwenden
- keine Secrets oder `.env`-Dateien in Prompts aufnehmen
- keine automatischen Commits oder Pushes zulassen
- Projekt zunächst in einem separaten Testverzeichnis starten

Ein lokales Modell verhindert zwar die Übertragung an einen Cloud-Modellanbieter, aber OpenCode kann weiterhin Dateien verändern und Terminalbefehle ausführen.

---

## 11. Start, Stopp und Updates

### oMLX

```bash
omlx start
omlx stop
omlx restart
```

Update bei Homebrew:

```bash
brew update
brew upgrade omlx
```

### Open WebUI

Starten:

```bash
docker start open-webui
```

Stoppen:

```bash
docker stop open-webui
```

Logs:

```bash
docker logs -f open-webui
```

Update:

```bash
docker pull ghcr.io/open-webui/open-webui:main
docker rm -f open-webui
```

Danach den ursprünglichen `docker run`-Befehl mit demselben `WEBUI_SECRET_KEY` erneut ausführen. Die Daten bleiben wegen des Volumes `open-webui` erhalten.

### OpenCode

```bash
brew update
brew upgrade opencode
```

---

## 12. Häufige Fehler

### `Connection refused` auf Port 8000

```bash
omlx start
curl -v http://127.0.0.1:8000/v1/models
```

Außerdem prüfen:

```bash
lsof -nP -iTCP:8000 -sTCP:LISTEN
```

### Open WebUI erreicht oMLX nicht

In Open WebUI muss die URL lauten:

```text
http://host.docker.internal:8000/v1
```

Nicht:

```text
http://localhost:8000/v1
```

### OpenCode zeigt kein Modell

```bash
jq . ~/.config/opencode/opencode.json
curl -s http://127.0.0.1:8000/v1/models | jq
```

Dann kontrollieren, ob die Modell-ID in der Konfiguration exakt mit der API-Ausgabe übereinstimmt.

### Mac wird langsam oder Anwendungen schließen sich

- kleineres Modell verwenden
- Kontext auf 8K oder 16K reduzieren
- parallele Anfragen auf 1 setzen
- Open-WebUI-RAG und lokale Embedding-Modelle zunächst deaktivieren
- Browser-Tabs und speicherintensive Anwendungen schließen

### MTP scheint nicht aktiv zu sein

- Modellname und Modellbeschreibung auf `MTP` prüfen
- model-specific settings in oMLX kontrollieren
- Logs nach `MTP` durchsuchen
- MTP nur mit einem dafür vorgesehenen Checkpoint aktivieren
- oMLX auf die aktuelle stabile Version aktualisieren

---

## Abschließender Funktionstest

Diese drei Tests sollten funktionieren:

```bash
# 1. Server erreichbar
curl -s http://127.0.0.1:8000/v1/models | jq

# 2. Open WebUI läuft
open http://localhost:3000

# 3. OpenCode im Projekt starten
cd ~/Developer/opencode-test
opencode
```

Die fertige Architektur sieht dann so aus:

```text
Open WebUI im Docker-Container
        |
        | OpenAI-kompatible API
        v
oMLX auf macOS, Port 8000
        |
        v
Qwen MLX-Modell mit MTP

OpenCode auf macOS
        |
        | OpenAI-kompatible API
        v
oMLX auf macOS, Port 8000
```

## Warum diese Struktur sinnvoll ist

- oMLX ist die einzige Inference-Schicht.
- Open WebUI und OpenCode verwenden dasselbe Modell.
- Das Modell wird nicht doppelt geladen.
- Chats und Coding laufen über einen einheitlichen lokalen API-Endpunkt.
- Docker wird nur für die Weboberfläche verwendet, nicht für die MLX-Inference.
- MTP bleibt eine Eigenschaft von Modell und oMLX und muss in Open WebUI oder OpenCode nicht separat konfiguriert werden.
