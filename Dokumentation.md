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

## Text-to-Speech

Es wird gearbeitet im VS Code. Hier wird ein Python script erstellt um Text-to-Speech für Reachy zu ermöglichen.



```python
import asyncio
import edge_tts
import pygame
from openai import OpenAI

# 1. OpenRouter Setup (Kostenloses Modell nutzen)
client = OpenAI(
    base_url="https://openrouter.ai/api/v1",
    api_key="HIER_KOMMT_DER_OPENROUTER_APIKEY_REIN", 
)

def get_llm_response(user_text):
    print("\nReachy denkt nach...")
    response = client.chat.completions.create(
        model="apodex/apodex-1.1-mini:free", # Das gratis Modell
        messages=[
            {"role": "system", "content": "Du bist Reachy, ein freundlicher Messe-Roboter der HTL Braunau. Antworte charmant und in maximal zwei kurzen Sätzen."},
            {"role": "user", "content": user_text}
        ]
    )
    return response.choices[0].message.content

async def speak_text(text):
    print(f"Reachy sagt: {text}")
    # Stimme generieren (de-DE-KillianNeural als männliche Stimme)
    communicate = edge_tts.Communicate(text, "de-DE-KillianNeural")
    await communicate.save("antwort.mp3")

    # Audio abspielen
    pygame.mixer.init()
    pygame.mixer.music.load("antwort.mp3")
    pygame.mixer.music.play()
    
    # Warten, bis der Roboter fertig gesprochen hat
    while pygame.mixer.music.get_busy():
        pygame.time.Clock().tick(10)
    pygame.mixer.quit()

# Hauptschleife für die Interaktion
async def main():
    print("=== Reachy Mini Software-Loop gestartet ===")
    while True:
        user_input = input("\nDu: (Schreibe etwas oder 'exit'): ")
        if user_input.lower() == 'exit':
            break
        
        # 1. Text an LLM senden
        antwort = get_llm_response(user_input)
        
        # 2. Antwort vorlesen
        await speak_text(antwort)

if __name__ == "__main__":
    asyncio.run(main())
```

Im VS Code Terminal
```
uv venv
.\.venv\Scripts\activate
uv pip install edge-tts openai pygame
python reachy_brain.py
```

# Reachy Mini

## Installation der App

Die App kann für den Desktop runtergealden werden
```
https://pollen-robotics.com/reachy-mini/getting-started/
```

Oder über die Shell mit phyton runtergeladen werden, hierzu muss nur diese Anleitung befolgt werden
```
https://huggingface.co/docs/reachy_mini/SDK/installation
```
