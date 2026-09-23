# Bericht N8N

---

**Thema:** N8N-Workflow
**Datum:** 29.07.2026 - 31.07.2026

---

### Aufgabenstellung

- Lokale n8n-Instanz aufsetzen
- Workflow zur automatischen Anonymisierung von NDAs erstellen
- Übergabe des anonymisierten NDAs an eine Flaggschiff-KI zur Zusammenfassung (nach bestimmten Kriterien)
- Rückgabe des fertigen Ergebnisses an das Ausgangs-Dashboard

---

### Vorgehen

Als Grundlage für die Automatisierung habe ich n8n per Docker Compose auf demselben Ubuntu-Rechner installiert, auf dem bereits Ollama, Open WebUI, AnythingLLM und OpenClaw laufen. Damit war sichergestellt, dass alle Dienste im selben lokalen Netzwerk kommunizieren können, ohne dass Daten den Rechner verlassen müssen, bevor sie anonymisiert sind.

Die grundlegende Architektur des Workflows sieht so aus:

1. **AnythingLLM (Chat-Oberfläche):** Der Nutzer lädt ein NDA in den Chat hoch und fordert die Anonymisierung samt Zusammenfassung an.
2. **AnythingLLM Agent Flow:** Ein selbst gebauter Flow übernimmt die Steuerung. Im ersten Schritt läuft ein lokales Ollama-Modell über einen „LLM Instruction"-Block und ersetzt alle personenbezogenen und geschäftsbezogenen Daten (Namen, Firmennamen, Adressen, Firmenbuchnummern, Telefonnummern, E-Mail-Adressen) durch Platzhalter.
3. **n8n-Workflow:** Der anonymisierte Text wird per Webhook an n8n übergeben. n8n leitet den Text mit einer definierten System-Prompt (den fachlichen Prüfkriterien für die NDA-Bewertung) an OpenClaw weiter.
4. **OpenClaw / Flaggschiff-Modell:** OpenClaw ruft über seine Gateway-API ein leistungsfähigeres Sprachmodell auf, das die eigentliche fachliche Zusammenfassung und Bewertung der Klauseln übernimmt.
5. **Rückgabe:** n8n verpackt das Ergebnis wieder in ein JSON-Format und schickt es an AnythingLLM zurück, wo es dem Nutzer im selben Chat-Fenster angezeigt wird.

Damit ist die Kernanforderung technisch abgesichert: Personenbezogene Daten verlassen den Rechner erst, nachdem sie lokal anonymisiert wurden. Die eigentliche fachliche Analyse läuft extern, aber ausschließlich mit bereits anonymisiertem Text.

---

### Herausforderungen und Lösungen

Der Weg dorthin war deutlich aufwändiger als ursprünglich angenommen, weil an mehreren Stellen unerwartete technische Hürden aufgetreten sind:

**Netzwerk- und Docker-Themen**
- Mehrfach musste ich zwischen `localhost` und der tatsächlichen LAN-IP des Rechners unterscheiden. Innerhalb eines Docker-Containers bedeutet `localhost` immer "dieser Container", nicht der Host-Rechner – das hat mehrere Verbindungsfehler verursacht, bis mir das bewusst wurde.
- OpenClaws Gateway war standardmäßig nur auf `127.0.0.1` gebunden (`bind: loopback`) und dadurch von n8n aus nicht erreichbar. Die Umstellung auf `bind: lan` hat das Problem gelöst, verlangt aber ein bewusstes Sicherheitsbewusstsein, weil der Dienst dadurch netzwerkweit erreichbar wird.
- Die HTTP-API von OpenClaw musste erst explizit über die Konfiguration aktiviert werden (`gateway.http.endpoints.chatCompletions.enabled`), da sie standardmäßig deaktiviert ist.

**JSON- und Formatierungsprobleme in n8n**
- Eine der hartnäckigsten Fehlerquellen war das korrekte Escaping von Text in n8n-Ausdrücken. Echte Zeilenumbrüche und Anführungszeichen in den Dokumenttexten haben das JSON-Format wiederholt "gebrochen", bis ich konsequent `JSON.stringify()` in den n8n-Ausdrücken verwendet habe, um Sonderzeichen automatisch korrekt zu escapen.
- Die Balance zwischen technisch zuverlässiger Ausgabe (als kompaktes JSON-Objekt) und optisch schön formatierter Ausgabe (mit echten Absätzen und Markdown im Chat) war ein durchgehendes Spannungsfeld. Jeder Versuch, eine schönere Formatierung zu erzwingen, hat entweder die Zuverlässigkeit verringert (Timeouts, Halluzinationen) oder ist an technischen Grenzen der Chat-Oberfläche gescheitert.

**Die {{document}}-Variable in AnythingLLM**
- Eine zentrale Fehlerquelle war die Annahme, dass es in AnythingLLM Agent Flows eine automatisch befüllte Variable `{{document}}` gibt, die den Inhalt einer im Chat angehängten Datei enthält. Nach ausführlichem Debugging (unter anderem mit einem eigens gebauten Test-Flow) und einem Abgleich mit der offiziellen Dokumentation sowie einem bekannten GitHub-Issue im AnythingLLM-Projekt hat sich herausgestellt: Diese Variable existiert nicht. Stattdessen müssen sogenannte "Flow Variables" explizit definiert werden, die der aufrufende Chat-Agent selbst mit Inhalt befüllen muss.
- Nach dieser Korrektur (Definition einer eigenen Variable `dokumenttext` statt `{{document}}`) hat die Übergabe des tatsächlichen Dokumentinhalts zuverlässig funktioniert.

**Modellwahl**
- Im direkten Vergleich hat sich gezeigt, dass nicht jedes lokale Modell gleich gut für Agent-/Tool-Aufgaben geeignet ist. Während `qwen2.5:7b` bei der Ausführung des Flows wiederholt in ein Timeout gelaufen ist, hat `gemma4:e4b` den kompletten Workflow zuverlässig und innerhalb der Zeit abgeschlossen. Das zeigt, dass die reine Modellgröße kein verlässlicher Indikator für die Eignung bei strukturierten Agent-Aufgaben ist.

**Bekannte Systemgrenze: Zeitlimit**
- AnythingLLM begrenzt die Antwortzeit eines Agent-Aufrufs fest auf 5 Minuten (300.000 ms), ohne dass sich dieser Wert konfigurieren lässt. Bei kurzen bis mittellangen Dokumenten reicht die Zeit zuverlässig aus. Bei sehr langen, mehrseitigen Dokumenten in Kombination mit einer ausführlichen fachlichen Prüf-Anweisung kann dieses Limit überschritten werden, weil sowohl die lokale Anonymisierung als auch die externe Zusammenfassung Zeit benötigen. Diese Grenze ist dokumentiert und nachvollzogen, aber im Rahmen der verfügbaren Zeit nicht mehr vollständig behoben worden.

---

### Ergebnis

Der Workflow funktioniert nachweislich zuverlässig für kurze bis mittellange NDA-Dokumente: Ein im Chat hochgeladenes Dokument wird lokal anonymisiert, die anonymisierte Fassung wird an ein externes, leistungsfähigeres Modell zur fachlichen Bewertung anhand definierter Kriterien weitergegeben, und das Ergebnis erscheint automatisch im ursprünglichen Chat-Fenster. Die Reihenfolge - erst lokale Anonymisierung, dann externe Verarbeitung - ist durch den strukturellen Aufbau des Agent Flows sichergestellt und nicht nur zufällig korrekt.

Die Ausgabe erfolgt aktuell als kompakter Textblock, nicht als vollständig gerendertes Markdown mit Absätzen. Dieser Kompromiss wurde bewusst zugunsten der Zuverlässigkeit gewählt, nachdem mehrere Ansätze zur saubereren Formatierung entweder zu Timeouts oder zu inhaltlich fehlerhaften Antworten (Halluzinationen) des lokalen Modells geführt haben.

### Lessons Learned

- Bei Automatisierungen über mehrere lokale und externe Dienste hinweg lohnt es sich, jede Schnittstelle isoliert zu testen (z. B. per curl), bevor man Fehler im komplexen Gesamtsystem sucht.
- Dokumentation und tatsächliches Verhalten einer Software können auseinanderfallen; bei unklarem Verhalten hilft der Blick in die offizielle Dokumentation und gegebenenfalls in offene Bug-Reports der Community, um die eigene Fehlerquelle korrekt einzuordnen.
- Zuverlässigkeit und Formatierungskomfort stehen bei lokalen, kleineren Sprachmodellen in einem Zielkonflikt, den man bewusst und dokumentiert entscheiden muss, statt ihn zu ignorieren.
