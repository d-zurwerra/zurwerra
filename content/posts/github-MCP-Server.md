---
title: 'GitHub MCP Server selbst hosten — für Copilot Studio Agents'
date: 2026-05-20
draft: false
description: 'Wie ihr einen eigenen GitHub MCP Server in Azure Container Apps aufbaut und an Copilot Studio anbindet – ohne GitHub Copilot Lizenz. Schritt für Schritt, mit allen Stolpersteinen.'
tags: ['GitHub', 'MCP', 'Copilot Studio', 'Azure Container Apps', 'GitHub App', 'Tutorial']
categories: ['Power Platform', 'Copilot Studio', 'MCP']
cover:
  image: 'https://the-raccoon-way.de/images/Artikel_GitHubMCP.png'
  alt: 'GitHub MCP Server für Copilot Studio'
  hiddenInList: true
---

Mein Content-Assistent Oskar sollte eigentlich nur eines tun: meine GitHub Issues und Projects pflegen, damit ich nicht ständig zwischen Repo, Backlog und Editor hin und her springe. Klingt nach "anbinden, fertig, weiter". War es nicht. Drei Stunden Anleitungen lesen, fünf Stack-Overflow-Threads, am Ende doch wieder von vorne — weil irgendwo immer ein Detail klemmt, das in keiner Doku steht.

Damit ihr euch das spart, hier die ausführliche Variante. Mit allem, was mich aufgehalten hat.

Am Ende läuft euer eigener GitHub MCP Server in Azure Container Apps, sauber verbunden mit Microsoft Copilot Studio. Ohne GitHub Copilot Lizenz, mit voller Kontrolle. Bei mir spricht Oskar inzwischen problemlos mit GitHub.

> **Lieber Video als Text?** Den Kurz-Überblick gibt es auch bei mir auf YouTube: [Github MCP Server – Nutzung mit Copilot Custom Agent](https://www.youtube.com/watch?v=NdmYQ0zuo04). Dieser Artikel ist die Langfassung mit Code und allen Klickpfaden.

![Screenshot des fertig konfigurierten Copilot Studio Agents mit dem angebundenen MCP Tool](/images/MCP_Cover.png "Github MCP Server in Copilot Studio")

# Warum überhaupt selbst hosten?

GitHub bietet einen offiziellen, fertig gehosteten MCP Server unter `https://api.githubcopilot.com/mcp/`. Auf den ersten Blick klingt das nach der einfachsten Lösung — URL eintragen, fertig.

Beim ersten Verbindungsversuch merkt man dann allerdings: Der GitHub Remote MCP Server verlangt eine aktive GitHub Copilot Lizenz für die OAuth-Authentifizierung. Ohne Lizenz keine Verbindung. Punkt.

Wer also GitHub Copilot nicht im Einsatz hat — oder wer einfach selbst entscheiden möchte, was unter der Haube passiert — braucht eine Alternative. Genau die bauen wir hier: einen eigenen MCP Server in Azure Container Apps. Funktioniert mit jedem MCP-fähigen Client (Copilot Studio, Claude Desktop, VS Code, Cursor), läuft in unserer eigenen Azure-Umgebung und lässt sich beliebig erweitern.

Was wir davon haben:

- Keine GitHub Copilot Lizenz nötig
- Volle Kontrolle über Authentifizierung und Konfiguration
- Funktioniert mit jedem MCP Client
- Läuft in unserer eigenen Azure-Infrastruktur
- Beliebig erweiterbar

# Architektur

```
MCP Client (Copilot Studio, Claude Desktop, VS Code, ...)
    ↓ HTTPS POST /mcp  (Authorization: Bearer <Fine-Grained PAT>)
GitHub MCP Wrapper (Azure Container Apps)
    ↓ GitHub App JWT → Installation Token (alle 50 Min. erneuert)
github-mcp-server Binary (HTTP Modus, Port 8080)
    ↓ GitHub API
GitHub
```

Kurze Erklärung zum Wrapper: Der offizielle `github-mcp-server` Binary akzeptiert im HTTP-Modus nur klassische PATs (`github_pat_...`) als Authorization-Token. GitHub App Installation Tokens (`ghs_...`) lehnt er kategorisch ab. Unser kleiner Python-Wrapper übernimmt die GitHub App Authentifizierung und reicht dem Binary im Hintergrund immer ein frisches Token unter.

# Voraussetzungen

Bevor wir starten, brauchen wir:

- Einen GitHub Account mit einer **Organisation** (kein persönlicher Account – dazu gleich mehr)
- Eine Azure Subscription mit einer bestehenden Azure Container Apps Environment
- Zugang zur GitHub Container Registry (GHCR)

> **Hinweis:** Ich gehe hier den Weg über die Organisation, weil Oskar bei mir auch **GitHub Projects** pflegen soll – und Projects-Zugriff zwingt einen auf Organisations-Ebene. Wer nur Issues, Repos und Actions ansprechen will, kommt mit einem persönlichen Account aus und kann die drei folgenden Hinweise sowie alles rund um „Organization permissions" überspringen. Der Rest der Anleitung bleibt identisch.

# Drei Dinge vorab, die viel Zeit sparen

Bevor wir irgendwo klicken, drei Punkte, die ich auf die harte Tour gelernt habe:

> **Hinweis 1:** Die GitHub App muss auf Organisations-Ebene erstellt werden. Eine App, die im persönlichen Account angelegt wird, kommt nicht an Organisations-Repos – auch nicht, wenn ihr Owner der Organisation seid.

> **Hinweis 2:** Der PAT muss auf die Organisation ausgestellt sein. Ein Fine-Grained PAT mit Resource Owner = persönlicher Account hat keinen Zugriff auf Organisations-Repos und Projects.

> **Hinweis 3:** Alle relevanten Repos und Projects müssen in der Organisation liegen. Persönliche User-Repos und Organisations-Repos lassen sich nicht mit demselben PAT abdecken.

Wer das verinnerlicht hat – auf geht's.

# GitHub App in der Organisation erstellen

Als erstes erstellen wir eine GitHub App in unserer Organisation. Dazu gehen wir wie folgt vor:

Wir öffnen **github.com**, klicken oben rechts auf unser Profilbild und dann auf **Your organizations**. Wir wählen unsere Organisation aus und gehen oben auf den Reiter **Settings**. Links in der Sidebar scrollen wir runter bis zu **Developer settings** und klicken dort auf **GitHub Apps**. Rechts oben dann der Button **New GitHub App**.

![Settings-Sidebar der Organisation mit dem markierten Eintrag "GitHub Apps" unter "Developer settings".](/images/MCP_GitHubApps_Sidebar.png "GitHub Apps Menü")

Alternativ funktioniert auch direkt: `github.com/organizations/EURE-ORG/settings/apps` → **New GitHub App**.

Auf der Folgeseite füllen wir folgende Felder aus:

| Feld | Wert |
|---|---|
| **GitHub App name** | `meine-org-mcp-server` (muss GitHub-weit eindeutig sein) |
| **Homepage URL** | eure Website (oder einfach das Org-Repo) |
| **Webhook** | das **Active**-Häkchen entfernen |

Den ganzen Block unter **Identifying and authorizing users** (Callback URL, Device Flow, User Authorization) können wir überspringen – wir machen App-to-App Auth, da loggt sich niemand ein.

Bei den **Repository permissions** brauchen wir:

| Permission | Level |
|---|---|
| Contents | Read-only |
| Issues | Read and write |
| Actions | Read-only |
| Metadata | Read-only (wird automatisch gesetzt) |

Und bei den **Organization permissions**:

| Permission | Level |
|---|---|
| Projects | Read and write |

Bei **Where can this GitHub App be installed?** wählen wir **Only on this account**.

![Ausgefüllte Permissions-Sektion der GitHub App.](/images/MCP_App_Permissions.png "Permissions")


Anschliessend ganz unten auf **Create GitHub App** klicken. Die App-Konfigurationsseite öffnet sich – und schon haben wir die App.

# Private Key generieren

Wir bleiben auf der App-Konfigurationsseite (falls nicht: **Org Settings → Developer settings → GitHub Apps → unsere App → Edit**) und scrollen runter bis zum Abschnitt **Private keys**. Dort klicken wir auf **Generate a private key**.

GitHub lädt automatisch eine `.pem`-Datei herunter.

> **Hinweis:** Die `.pem`-Datei kann nicht erneut heruntergeladen werden. Bitte sicher aufbewahren – ist sie weg, müssen wir einen neuen Key generieren.

Was wir uns notieren:

- **App ID** (steht ganz oben auf der App-Seite, eine Zahl)
- Den Pfad zur `.pem`-Datei

![PLATZHALTER: Bereich "Private keys" mit dem "Generate a private key"-Button und/oder oben die App ID.](/images/MCP_PrivateKey.png "Private Key")

# GitHub App in der Organisation installieren

Die App existiert jetzt, ist aber noch nicht in der Organisation installiert. Das holen wir nach.

Auf der App-Konfigurationsseite klicken wir links in der Sidebar auf **Install App** und dann neben unserer Organisation auf **Install**. Im nächsten Screen wählen wir bei **Repository access** den Punkt **All repositories** und bestätigen mit **Install**.

Nach der Installation landen wir auf einer URL der Form:

```
https://github.com/settings/installations/XXXXXXXX
```

Die Zahl am Ende ist unsere **Installation ID**. Auch die notieren wir uns.

![Browser-Adressleiste mit markierter Installation ID nach der Installation.](/images/MCP_InstallationID.png " Installation ID")

# Fine-Grained PAT erstellen

Bevor wir loslegen, eine wichtige Voraussetzung:

> **Hinweis:** Die Organisation muss Fine-Grained PATs erlauben. Falls nicht: in der Orga unter **Settings → Personal access tokens → Settings** den Punkt **Allow access via fine-grained personal access tokens** aktivieren. Ohne diese Einstellung schlägt jeder Versuch fehl, einen PAT auf die Orga auszustellen.

Jetzt zum PAT selbst: Wir klicken oben rechts auf unser Profilbild und gehen auf **Settings**. In der Sidebar scrollen wir ganz nach unten zu **Developer settings**. Im Menü links wählen wir **Personal access tokens** und dann **Fine-grained tokens**. Rechts oben klicken wir auf **Generate new token**.

![Developer-Settings-Sidebar mit hervorgehobenem Eintrag "Fine-grained tokens".](/images/MCP_FineGrainedTokens.png "Fine-grained tokens")

Die Felder füllen wir wie folgt:

| Feld | Wert |
|---|---|
| **Token name** | `meine-org-mcp-server` |
| **Expiration** | 1 year (oder kürzer, je nach Policy) |
| **Resource owner** | unsere Organisation (Dropdown – nicht der persönliche Account!) |
| **Repository access** | All repositories |

**Repository permissions:**

| Permission | Level |
|---|---|
| Contents | Read |
| Issues | Read & Write |
| Actions | Read |
| Metadata | Read |

**Organization permissions:**

| Permission | Level |
|---|---|
| Projects | Read & Write |

Anschliessend auf **Generate token** klicken.

> **Hinweis:** Der PAT wird genau einmal angezeigt. Bitte sofort kopieren und sicher ablegen – ist der Tab erst zu, hilft nur noch ein neuer Token.

![Erfolgsseite mit generiertem PAT.](/images/MCP_PAT_generated.png "PAT generiert")

# Wrapper-Code erstellen

Jetzt wird es technisch. Wir brauchen fünf Dateien in folgender Struktur:

```
github-mcp-wrapper/
├── .github/
│   └── workflows/
│       └── deploy.yml
├── app/
│   └── main.py
├── Dockerfile
├── requirements.txt
├── .gitignore
└── README.md
```

> **Hinweis:** Den Code gibt es auch im Repository auf GitHub: [github-mcp-wrapper Repository](https://github.com/d-zurwerra/github-mcp-wrapper) – wer nicht alles selbst tippen möchte, kann sich das einfach klonen.

## `requirements.txt`

```
httpx==0.27.2
PyJWT==2.9.0
cryptography==43.0.1
```

## `Dockerfile`

```dockerfile
FROM python:3.12-slim

# Lädt das github-mcp-server Binary von GitHub Releases.
# Achtung: Dateiname ist github-mcp-server_Linux_x86_64.tar.gz
# (grosses L in Linux, x86_64 - NICHT amd64).
RUN apt-get update && apt-get install -y curl && rm -rf /var/lib/apt/lists/* \
    && curl -fsSL \
       "https://github.com/github/github-mcp-server/releases/latest/download/github-mcp-server_Linux_x86_64.tar.gz" \
       | tar -xz -C /usr/local/bin/ github-mcp-server \
    && chmod +x /usr/local/bin/github-mcp-server

WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY app/ ./app/

EXPOSE 8080

# -u = unbuffered output, damit Logs sofort in ACA erscheinen
CMD ["python", "-u", "app/main.py"]
```

## `app/main.py`

Das Skript holt das Installation Token von der GitHub App, startet den `github-mcp-server` und rotiert das Token alle 50 Minuten. Hier muss ich ehrlich sein: Beim Python-Teil war Claude eine echte Hilfe – ich bin nicht der Coder. Aber das Ergebnis tut, was es soll.

```python
#!/usr/bin/env python3
"""
GitHub MCP Wrapper
Holt ein GitHub App Installation Token und startet den
github-mcp-server im HTTP Modus mit automatischer Token-Erneuerung.
"""

import os, sys, time, logging, threading, subprocess, signal
import httpx
import jwt

logging.basicConfig(level=logging.INFO, format="%(asctime)s [%(levelname)s] %(message)s")
logger = logging.getLogger(__name__)

# Konfiguration aus Umgebungsvariablen
GITHUB_APP_ID          = os.environ["GITHUB_APP_ID"]
GITHUB_APP_PRIVATE_KEY = os.environ["GITHUB_APP_PRIVATE_KEY"].replace("\\n", "\n")
GITHUB_INSTALLATION_ID = os.environ["GITHUB_INSTALLATION_ID"]
MCP_AUTH_TOKEN         = os.environ["MCP_AUTH_TOKEN"]
GITHUB_TOOLSETS        = os.environ.get("GITHUB_TOOLSETS", "repos,issues,actions,projects")
MCP_PORT               = os.environ.get("MCP_PORT", "8080")
GITHUB_MCP_BINARY      = os.environ.get("GITHUB_MCP_BINARY", "/usr/local/bin/github-mcp-server")

def _generate_jwt() -> str:
    now = int(time.time())
    payload = {"iat": now - 60, "exp": now + 600, "iss": GITHUB_APP_ID}
    return jwt.encode(payload, GITHUB_APP_PRIVATE_KEY, algorithm="RS256")

def fetch_installation_token() -> str:
    app_jwt = _generate_jwt()
    url = f"https://api.github.com/app/installations/{GITHUB_INSTALLATION_ID}/access_tokens"
    headers = {
        "Authorization": f"Bearer {app_jwt}",
        "Accept": "application/vnd.github+json",
        "X-GitHub-Api-Version": "2022-11-28",
    }
    with httpx.Client() as client:
        resp = client.post(url, headers=headers)
        resp.raise_for_status()
        token = resp.json()["token"]
        logger.info("Installation token fetched successfully")
        return token

_current_token: str | None = None
_mcp_process: subprocess.Popen | None = None

def _start_mcp_server():
    global _mcp_process
    env = {
        **os.environ,
        "GITHUB_PERSONAL_ACCESS_TOKEN": _current_token,
        "GITHUB_TOOLSETS": GITHUB_TOOLSETS,
        "GITHUB_MCP_SERVER_AUTH_TOKEN": MCP_AUTH_TOKEN,
    }
    cmd = [GITHUB_MCP_BINARY, "http", "--port", MCP_PORT]
    logger.info("Starting github-mcp-server on port %s", MCP_PORT)
    _mcp_process = subprocess.Popen(cmd, env=env)

def _refresh_and_restart(interval_seconds: int = 50 * 60):
    global _current_token, _mcp_process
    while True:
        time.sleep(interval_seconds)
        try:
            _current_token = fetch_installation_token()
            if _mcp_process and _mcp_process.poll() is None:
                _mcp_process.terminate()
                _mcp_process.wait(timeout=10)
            _start_mcp_server()
            logger.info("Token refreshed and server restarted")
        except Exception as e:
            logger.error("Token refresh failed: %s", e)

def _handle_signal(signum, frame):
    if _mcp_process and _mcp_process.poll() is None:
        _mcp_process.terminate()
    sys.exit(0)

def main():
    global _current_token
    signal.signal(signal.SIGTERM, _handle_signal)
    signal.signal(signal.SIGINT, _handle_signal)
    _current_token = fetch_installation_token()
    _start_mcp_server()
    threading.Thread(target=_refresh_and_restart, daemon=True).start()
    sys.exit(_mcp_process.wait())

if __name__ == "__main__":
    main()
```

## `.github/workflows/deploy.yml`

```yaml
name: Build and Push

on:
  push:
    branches: [main]
  workflow_dispatch:

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Log in to GHCR
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=sha,prefix=sha-
            type=raw,value=latest,enable={{is_default_branch}}

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
```

# Repository anlegen und Code pushen

Jetzt brauchen wir ein Zuhause für den Code. In der Organisation klicken wir oben rechts auf **+** und wählen **New repository**. Als Name nehmen wir z.B. `github-mcp-wrapper`. Wichtig: **Private** auswählen – wir wollen den Wrapper-Code nicht öffentlich liegen haben. Beim Erstellen aktivieren wir noch **Add .gitignore** mit dem Template **Python** und klicken auf **Create repository**.

Anschliessend klonen wir das Repo lokal, legen die fünf Dateien in der oben gezeigten Struktur ab, committen und pushen.

Sobald wir nach `main` pushen, läuft die GitHub Action automatisch los, baut das Image und pusht es nach GHCR. Beim ersten Mal lohnt sich ein Blick in den Reiter **Actions** – falls dort etwas rot ist, klemmt es noch und Azure würde später nichts ziehen können.

![Actions-Tab des Repos mit einem grünen Build – der "es funktioniert"-Beweis.](/images/MCP_Actions_Build.png "GitHub Actions Build")

> **Hinweis zum CI/CD-Prinzip:** GitHub Actions baut das Image und pusht es nach GHCR. Azure Container Apps zieht das Image direkt von GHCR – GitHub pusht nichts nach Azure. Nach einem neuen Build erstellen wir in ACA einfach eine neue Revision.

# Azure Container App erstellen

Jetzt nach Azure. Wir öffnen **portal.azure.com**, loggen uns ein und tippen in die Suchleiste oben **Container Apps**. Dort klicken wir oben links auf **+ Create** und wählen **Container App**.

## Basics

| Feld | Wert |
|---|---|
| **Subscription / Resource Group** | wie bei euren anderen Apps |
| **Container App Name** | `ca-mcp-github` |
| **Region** | dieselbe wie eure bestehenden Apps |
| **Container Apps Environment** | bestehendes Environment wiederverwenden |

## Container

Hier wird es spannend, weil wir das Image aus GHCR ziehen.

| Feld | Wert |
|---|---|
| **Image source** | Docker Hub or other registries |
| **Image type** | Private |
| **Registry login server** | `ghcr.io` |
| **Registry username** | euer GitHub Username |
| **Registry password** | ein GitHub PAT mit Scope `read:packages` |
| **Image and tag** | `<username>/<repo-name>:latest` |

> **Hinweis:** Im Feld **Image and tag** tragen wir **nur** `<username>/<repo-name>:latest` ein – **ohne** `ghcr.io/` davor. Der Login Server liefert das Präfix bereits. Andernfalls entsteht `ghcr.io/ghcr.io/...` und Azure beschwert sich mit einer entsprechenden Fehlermeldung. Diese Falle hat mich überraschend lange aufgehalten.

![Korrekt ausgefülltes Container-Formular mit dem richtigen Image-Pfad.](/images/MCP_Azure_Container.png "Container-Formular")

## Environment Variables

Diese Variablen brauchen wir – Achtung beim Private Key, der ist ein Sonderfall.

| Name | Wert |
|---|---|
| `GITHUB_APP_ID` | unsere App ID (die Zahl aus dem App-Schritt) |
| `GITHUB_INSTALLATION_ID` | unsere Installation ID |
| `MCP_AUTH_TOKEN` | unser Fine-Grained PAT (`github_pat_...`) |
| `GITHUB_MCP_SERVER_AUTH_TOKEN` | derselbe Fine-Grained PAT |
| `GITHUB_PERSONAL_ACCESS_TOKEN` | derselbe Fine-Grained PAT |
| `GITHUB_TOOLSETS` | `repos,issues,actions,projects` |

> **Hinweis:** Für `GITHUB_APP_PRIVATE_KEY` müssen wir ein separates ACA Secret anlegen. Der PEM-Key ist mehrzeilig, und als normale Env-Variable killt Azure die Zeilenumbrüche. Wir gehen in der Container App im linken Menü auf **Secrets**, klicken auf **+ Add**, vergeben den Namen `github-app-private-key`, fügen den kompletten PEM-Inhalt (inklusive `-----BEGIN ...` und `-----END ...`) ein und speichern. Anschliessend in der Revision die Env-Variable `GITHUB_APP_PRIVATE_KEY` anlegen und bei **Source** auf **Reference a secret** umstellen.

![Secrets-Seite der Container App mit dem Private-Key-Secret.](/images/MCP_Azure_Secret.png "Azure Secret")

## Ingress

| Feld | Wert |
|---|---|
| **Ingress** | aktivieren |
| **Ingress traffic** | Accept traffic from anywhere |
| **Ingress type** | HTTP |
| **Target port** | `8080` |

Unten auf **Review + create** klicken, dann auf **Create**. Azure deployt jetzt unsere App – das dauert je nach Tag und Wetterlage zwei bis fünf Minuten.

# Endpoint testen

Bevor wir Copilot Studio (und damit Oskar) einbinden, prüfen wir kurz, ob der Server überhaupt antwortet. Die ACA-URL finden wir auf der **Overview**-Seite der Container App ganz oben rechts unter `Application Url`.

```bash
curl -X POST https://<eure-aca-url>/mcp \
  -H 'Authorization: Bearer github_pat_...' \
  -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","method":"tools/list","params":{},"id":1}'
```

Wenn alles passt, kommt JSON mit einer Liste aller verfügbaren MCP Tools zurück. Falls nicht: einmal in die Logs schauen unter **ca-mcp-github → Monitoring → Log stream**. Da steht meistens schon, was klemmt.

# Copilot Studio verbinden

Der letzte Schritt. Wir öffnen **make.powerapps.com** (oder direkt `copilotstudio.microsoft.com`), wählen unseren Agent aus, gehen oben auf den Reiter **Tools** und klicken auf **+ Add a tool**. In der Auswahl wählen wir **Model Context Protocol (MCP)**.

!["Add a tool"-Menü mit hervorgehobener MCP-Option.](/images/MCP_CopilotStudio_AddTool.png "Add Tool")


Die Felder füllen wir wie folgt:

| Feld | Wert |
|---|---|
| **Server name** | `github-mcp-server` |
| **Server description** | kurze Beschreibung, z.B. "Eigener GitHub MCP Server für Org X" |
| **Server URL** | `https://<eure-aca-url>/mcp` |
| **Authentication** | API key |
| **Type** | Header |
| **Header name** | `Authorization` |
| **API Key value** | `Bearer github_pat_...` |

> **Hinweis:** Im Feld **API Key value** muss `Bearer ` (mit Leerzeichen!) **vor** dem PAT stehen. Copilot Studio übernimmt den Wert 1:1 als Header – es ergänzt kein automatisches `Bearer ` davor. Also **nicht** nur `github_pat_xxx`, sondern `Bearer github_pat_xxx`. Wenn die Tools nach dem Speichern nicht aufgelöst werden, ist das in den meisten Fällen die Ursache. Auch dieser Punkt hat mich Zeit gekostet.

Nach dem Speichern dauert es ein paar Sekunden, dann werden alle Tools automatisch in der Liste aufgelöst. Falls nicht: einmal neu laden, dann in Azure und Copilot Studio in die Logs schauen.

![Konfigurierter MCP-Server in Copilot Studio mit aufgelöster Tool-Liste.](/images/MCP_CopilotStudio_Tools.png "Tools aufgelöst")

# Teams Connection zurücksetzen (falls nötig)

Wer den Agent in Microsoft Teams nutzt, kennt vielleicht das Phänomen: Nach einer Konfigurationsänderung verwendet der Bot trotzdem noch den alten Token. Teams cached den Conversation State. Die Lösung ist ein kurzer Befehl im Teams-Chat mit dem Bot:

```
/debug clearstate
```

Damit ist der Cache geleert und die neue Konfiguration greift sofort.

**FERTIG**

Das war's. Wir haben einen eigenen GitHub MCP Server, der in Azure läuft, von Copilot Studio aus erreichbar ist – und das ohne GitHub Copilot Lizenz.

# Warum GitHub App UND PAT?

Klingt erstmal überflüssig, hat aber einen Grund.

Die GitHub App löst das Token-Ablauf-Problem: sie generiert kurzlebige Installation Tokens (60 Minuten Gültigkeit), die unser Wrapper alle 50 Minuten automatisch erneuert. So bleibt der Server beliebig lang produktiv ohne manuelle Token-Pflege.

Den PAT brauchen wir aus einem rein technischen Grund: Der `github-mcp-server` Binary akzeptiert im HTTP-Modus nur PATs (`github_pat_...`). Installation Tokens (`ghs_...`) lehnt er ab. Der PAT übernimmt dabei zwei Aufgaben gleichzeitig:

1. **Endpoint-Schutz** – wer den PAT nicht kennt, kommt nicht an den MCP Server
2. **GitHub-API-Fallback** – der Server kann ihn theoretisch auch für API-Calls nutzen

Teams sieht von der GitHub App übrigens gar nichts. Was Teams (bzw. Copilot Studio) braucht, ist ausschliesslich der PAT als Bearer Token. Die GitHub App ist eine reine Backend-Mechanik im Wrapper – wichtig wegen des Repo-Zugriffs, nicht wegen Teams.

# Stolpersteine im Überblick

Falls ihr irgendwo hängt, hier nochmal eine Sammlung an Dingen, die mich überrascht haben:

- **Binary-Dateiname.** Das Release heisst `github-mcp-server_Linux_x86_64.tar.gz` – grosses **L** in `Linux`, `x86_64` statt `amd64`. Falscher Name = 404 im Build.
- **HTTP-Modus Flags.** `--port` funktioniert. `--host` gibt es nicht (Server lauscht automatisch auf `0.0.0.0`). `--auth-token` gibt es auch nicht – der Token-Schutz läuft über die Env-Variable `GITHUB_MCP_SERVER_AUTH_TOKEN`.
- **Authentifizierung.** Der Server akzeptiert im HTTP-Modus nur echte GitHub Tokens (`ghp_...` oder `github_pat_...`). Eigene zufällige Bearer-Tokens werden mit `bad request: Authorization header is badly formatted` abgelehnt.
- **GitHub App auf persönlichem Account.** Sieht nach einem funktionierenden Setup aus – tut aber nicht. Auch wenn ihr Owner der Organisation seid: persönliche Apps kommen nicht an Org-Repos.
- **PAT mit persönlichem Resource Owner.** Gleiche Geschichte. Resource Owner muss die Organisation sein. Falls die Orga das verbietet: zuerst in den Org-Einstellungen Fine-Grained PATs aktivieren.
- **Repos und Projects in der Organisation.** Persönliche User-Repos und Organisations-Repos lassen sich nicht mit einem PAT abdecken. Bestehende Repos können über **Repo → Settings → Danger Zone → Transfer repository** in die Org geschoben werden. **GitHub Projects lassen sich nicht transferieren** – die müssen in der Org neu angelegt werden.
- **Markdown-Rendering.** Manche Editoren machen aus Underscores in URLs Kursivschrift. Wenn der Download-Link plötzlich nicht mehr geht: im Raw-View prüfen.

# Umgebungsvariablen-Übersicht

| Variable | Beschreibung |
|---|---|
| `GITHUB_APP_ID` | GitHub App ID (Zahl) |
| `GITHUB_APP_PRIVATE_KEY` | RSA Private Key (PEM) – als ACA Secret speichern |
| `GITHUB_INSTALLATION_ID` | App Installation ID |
| `GITHUB_MCP_SERVER_AUTH_TOKEN` | Fine-Grained PAT – Endpoint-Schutz und API-Fallback |
| `GITHUB_PERSONAL_ACCESS_TOKEN` | Fine-Grained PAT – wird vom Wrapper an den Server gereicht |
| `MCP_AUTH_TOKEN` | Fine-Grained PAT – wird intern vom Wrapper verwendet |
| `GITHUB_TOOLSETS` | aktivierte Tool-Gruppen (Standard: `repos,issues,actions,projects`) |
| `MCP_PORT` | Port des MCP Servers (Standard: `8080`) |

# Weitere Punkte für eine nächste Version

- **Authentifizierung verfeinern:** Aktuell ist der PAT gleichzeitig Endpoint-Schutz und GitHub-API-Token. Sauberer wäre eine getrennte Auth-Ebene vor dem MCP Server (z.B. APIM), damit der GitHub-Token nie das Frontend erreicht.
- **Monitoring & Alerts:** Aktuell verlasse ich mich auf Azure Log Stream. Eine kleine Alerting-Regel auf "Token Refresh failed" wäre sinnvoll.
- **Webhook-basierter Restart:** Statt fix alle 50 Minuten zu rotieren, wäre eine Token-Erneuerung knapp vor Ablauf eleganter.
- **Mehr Tools:** Aktuell habe ich `repos,issues,actions,projects` aktiviert. Je nach Use Case lassen sich noch weitere Toolsets ergänzen.

Bei mir spricht Oskar inzwischen ohne Murren mit GitHub – triagiert neue Issues, fasst Threads zusammen und legt Projects-Einträge an, ohne dass ich GitHub überhaupt aufmachen muss. Genau dafür hat sich der Aufwand gelohnt.

Was ihr mit eurem MCP Server macht, würde mich interessieren – schreibt mir gerne auf [LinkedIn](https://www.linkedin.com/in/danielazurwerra/), woran ihr arbeitet. Ich beisse auch nicht.

![Abschlussbild – Oskar in Aktion, z.B. beim Zusammenfassen eines Issues oder Anlegen eines Project-Eintrags.](/images/MCP_Oskar_Action.png "Oskar in Aktion")
