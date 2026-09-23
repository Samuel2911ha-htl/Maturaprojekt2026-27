# Bericht Lokale-KI
---

**Thema:** Lokale-LLM

**Datum:** 27.07.2026 - 29.07.2026

---

### Aufgabenstellung
- Ubuntu 24.04 auf dem leistungsstarken Grafikcomputer installieren und alle verfügbaren Systemupdates
  einspielen.
- Die passenden NVIDIA-Treiber installieren und prüfen, ob die Grafikkarten korrekt erkannt werden.
- Docker, Docker Compose und das NVIDIA Container Toolkit installieren, damit Container auf die
  Grafikkarten zugreifen können.
- Ollama als Docker-Container installieren und starten.
- Open WebUI oder AnythingLLM als Weboberfläche über Docker installieren und mit Ollama verbinden.
- Ein geeignetes Gemma-4-Modell herunterladen, laden und mit mehreren Beispielanfragen testen.
- Stable Diffusion über Docker installieren, ein geeignetes Modell laden und mehrere Testbilder erzeugen.
- Kontrollieren, ob Sprachmodell und Bildgenerierung tatsächlich die NVIDIA-Grafikkarten verwenden.
  
---

## 1. Vorbereitungen

Als ersten Schritt habe ich das frisch installierte Ubuntu-System aktualisiert:

```
sudo apt update && sudo apt upgrade -y
```

## 2. Docker installieren

Da alle Dienste (Ollama, Open WebUI, AnythingLLM) als Container laufen sollen, habe ich zuerst Docker installiert. 

```
sudo apt install ca-certificates curl -y
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
```

Anschließend habe ich die Paketquelle für Docker angelegt:

```
sudo nano /etc/apt/sources.list.d/docker.sources
```

Mit folgendem Inhalt:

```
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: noble
Components: stable
Architectures: amd64
Signed-By: /etc/apt/keyrings/docker.asc
```

Danach konnte ich Docker installieren und prüfen ob alles funktioniert:

```
sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y
sudo systemctl status docker
sudo docker run hello-world
```

### 2.1 Docker ohne sudo verwenden

```
sudo groupadd docker
sudo usermod -aG docker $USER
newgrp docker
```

## 3. NVIDIA-Treiber installieren

Damit die KI-Modelle später die GPU nutzen können, war zunächst der passende Grafiktreiber notwendig. Ich habe zuerst geprüft, ob die GPU vom System erkannt wird:

```
lspci | grep -i nvidia
```

Statt eine feste Treiberversion einzutragen, habe ich Ubuntu den empfohlenen Treiber automatisch ermitteln und installieren lassen.

```
sudo ubuntu-drivers install
sudo reboot
```

Nach dem Neustart habe ich mit folgendem Befehl kontrolliert, ob der Treiber korrekt geladen ist:

```
nvidia-smi
```

## 4. NVIDIA Container Toolkit installieren

Damit Docker-Container auf die GPU zugreifen können, ist zusätzlich das NVIDIA Container Toolkit notwendig. Zuerst habe ich die benötigten Grundpakete installiert:

```
sudo apt-get update && sudo apt-get install -y --no-install-recommends ca-certificates curl gnupg2
```

Anschließend habe ich den GPG-Schlüssel und die Paketquelle eingerichtet. Da die Paketquelle für Ubuntu 24.04 zum Zeitpunkt der Einrichtung noch nicht separat bereitgestellt wurde, habe ich bewusst die Paketquelle für Ubuntu 22.04 verwendet diese ist mit der aktuellen Ubuntu-Version voll kompatibel:

```
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg

curl -s -L https://nvidia.github.io/libnvidia-container/ubuntu22.04/libnvidia-container.list | \
  sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
  sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
```

Danach habe ich das Toolkit installiert und Docker entsprechend konfiguriert:

```
sudo apt-get update
sudo apt-get install -y nvidia-container-toolkit
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
```

Zum Abschluss habe ich mit einem Testcontainer überprüft, ob die GPU innerhalb eines Containers sichtbar ist:

```
docker run --rm --gpus all nvidia/cuda:12.3.0-base-ubuntu22.04 nvidia-smi
```

## 5. Ollama und Open WebUI mit Docker Compose

Für den eigentlichen KI-Betrieb habe ich mich für Ollama entschieden, da es lokale Sprachmodelle verwaltet und über eine einheitliche Schnittstelle bereitstellt. Als grafische Oberfläche dazu habe ich Open WebUI eingerichtet.
Zuerst habe ich einen Projektordner angelegt:

```
mkdir -p ~/ollama-stack && cd ~/ollama-stack
nano docker-compose.yml
```

Mit folgendem Inhalt:

```
services:
  ollama:
    image: ollama/ollama:latest
    runtime: nvidia
    environment:
      - NVIDIA_VISIBLE_DEVICES=all
    ports:
      - "11434:11434"
    volumes:
      - ./ollama:/root/.ollama

  open-webui:
    image: ghcr.io/open-webui/open-webui:main
    ports:
      - "3000:8080"
    environment:
      - OLLAMA_BASE_URL=http://ollama:11434
    depends_on:
      - ollama
```

Ich habe bewusst auf die veraltete "version"-Angabe am Anfang der Datei verzichtet, da diese in aktuellen Docker-Compose-Versionen nicht mehr benötigt wird und nur eine Warnung erzeugt. 
Gestartet habe ich den Stack mit:

```
docker compose up -d
docker compose ps
```

Nach dem Start liefen beide Container fehlerfrei (Status "Up", Open WebUI zusätzlich als "healthy" markiert).

### 5.1 Erreichbarkeit im Netzwerk

Open WebUI ist unter `http://localhost:3000` erreichbar, sofern man direkt am Rechner arbeitet, auf dem Docker läuft. Da ich in meinem Fall von einem anderen Gerät im selben Netzwerk zugegriffen habe, musste ich stattdessen die tatsächliche Netzwerk-IP-Adresse des Ubuntu-Rechners verwenden, da sich "localhost" immer auf das Gerät bezieht.

```
http://192.168.30.103:3000
```

## 6. Sprachmodelle herunterladen

Nachdem die Umgebung lief, habe ich verschiedene Sprachmodelle über Ollama heruntergeladen. Der Name des Ollama-Containers ergibt sich automatisch aus dem Ordnernamen des Compose-Projekts (bei mir `ollama-stack-ollama-1`), abrufbar über `docker ps`.

```

docker exec -it ollama-stack-ollama-1 ollama pull gemma4:e4b
docker exec -it ollama-stack-ollama-1 ollama pull qwen2.5:7b
```
Alle installierten Modelle lassen sich mit folgendem Befehl auflisten:

```
docker exec -it ollama-stack-ollama-1 ollama list
```

## 7. AnythingLLM als Chat-Oberfläche mit Dokumentenanbindung

```
mkdir -p ~/anythingllm-storage
touch ~/anythingllm-storage/.env

docker run -d -p 3001:3001 \
  --cap-add SYS_ADMIN \
  --name anythingllm \
  -v ~/anythingllm-storage:/app/server/storage \
  -v ~/anythingllm-storage/.env:/app/server/.env \
  -e STORAGE_DIR="/app/server/storage" \
  mintplexlabs/anythingllm:latest
```

Ohne die Angabe von `STORAGE_DIR` bricht der Container bereits beim Start ab, da die Anwendung dann keinen gültigen Speicherort findet. Mit der obigen, vollständigen Konfiguration lief der Container von Anfang an stabil (Status "healthy"). Kontrolliert habe ich das mit:

```
docker ps
docker logs anythingllm --tail 20
```

AnythingLLM ist danach ebenfalls über die Netzwerk-IP des Rechners erreichbar, z. B.:

```
http://192.168.30.103:3001
```

Als LLM-Anbindung habe ich in AnythingLLM Gemini ausgewählt:

```
http://192.168.30.103:3001
```

## 8. Optionale Cloud-Anbindung über die Google Gemini API

Neben den rein lokalen Modellen habe ich zusätzlich die Möglichkeit einer optionalen Cloud-Anbindung über die Google Gemini API getestet, um beide Varianten – lokal und cloudbasiert – vergleichen zu können. Dafür habe ich mir in Google AI Studio einen kostenlosen API-Schlüssel erstellt:

- Aufruf von aistudio.google.com und Anmeldung mit einem Google-Konto
- Klick auf "Get API key" und anschließend "Create API key"
- Sicheres Abspeichern des generierten Schlüssels, da er später nicht erneut im Klartext angezeigt wird

Getestet habe ich den Schlüssel direkt über die Kommandozeile:

```
export GEMINI_API_KEY="mein_schluessel"
curl "https://generativelanguage.googleapis.com/v1beta/models?key=${GEMINI_API_KEY}"
```

## 9. Stable Diffusion über Docker installieren

Für die Bildgenerierung habe ich zusätzlich Stable Diffusion eingerichtet.
Da Ollama ausschließlich Sprachmodelle unterstützt und keine Bildgenerierung beherrscht, habe ich dafür ein eigenständiges Docker-Projekt verwendet:
```
git clone https://github.com/AbdBarho/stable-diffusion-webui-docker.git
cd stable-diffusion-webui-docker
```
Anschließend habe ich zunächst die Modelle heruntergeladen und danach das eigentliche Image gebaut und gestartet:
```
docker compose --profile download up
docker compose --profile auto up --build
```
Erreichbar war die Stable-Diffusion-WebUI danach ebenfalls über die Netzwerk-IP des Rechners:
```
http://192.168.30.103:7860
```
Um sicherzustellen, dass sowohl das Sprachmodell als auch die Bildgenerierung tatsächlich die NVIDIA-Grafikkarte verwenden und nicht auf der CPU laufen, habe ich die GPU-Auslastung während des Betriebs live beobachtet:
```
watch -n 1 nvidia-smi
```

















