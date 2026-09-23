# Dokumentation

## Ollama Installation

Link: `https://ollama.com/download/linux`
Mit Befehl auf Ubuntu heruntergeladen:
```
curl -fsSL https://ollama.com/install.sh | sh
```

### Einschub Node.js Installation

```
sudo apt update
sudo apt install nodejs npm -y
```

### Fortsetzung Ollama

```
ollama
```
`OpenClaw` auswählen und mit `Yes` bestätigen

Berechtigungsfehler tritt auf

Lösung durch erstellen eines eigenen Ordners für das Projekt
```
mkdir -p ~/ollama-stack && cd ~/ollama-stack
nano docker-compose.yml
```
Nun haben wir einen eigenen Ordner mit der Docker Konfigurationsdatei.
Jetzt kann der Container gestartet werden.
```
docker compose up -d
```
Zur Überprüfung
```
docker compose ps
```
Nun kann ein LLM installiert werden
```
docker exec -it ollama-stack-ollama-1 ollama pull llama3.2
```
Mit 
```
http://localhost:3000
```
Kann jetzt die Open WebGUI geöffnet werden

