# Hinweise für Agents

Dieser Fork betreibt ein reales Tor (Nice Robus + BiDi-WiFi). Projektstand, Hardware, Build, Flashen und Zugriff stehen in [TOR.md](TOR.md); zuerst dort lesen. Allgemeine Doku der Komponente: [README.md](README.md).

Regeln:

- Arbeitsbranch ist `tor-sugam`; `main` bleibt identisch mit Upstream.
- Keine Befehle oder Entitäten einbauen, die Parameter der Steuerung schreiben (SET), ohne ausdrückliche Freigabe. Gründe und Plan: TOR.md, Abschnitte „Funktionsumfang“ und „Geplant“.
- `secrets.yaml` und die Firmware-Sicherung unter `../backup/` nie committen, ausgeben oder weitergeben.
- Firmware auf den Stick (OTA oder UART) und Fahrbefehle an das Tor nur nach Rückfrage.
