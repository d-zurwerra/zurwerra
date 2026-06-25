---
title: 'M365 MCP Server selbst hosten — mit Managed Identity und OAuth'
date: 2026-06-24
draft: true
description: 'Wie ihr einen M365 MCP Server in Azure Container Apps betreibt, mit Managed Identity gegen die Graph API authentifiziert und per OAuth an Copilot Studio anbindet. Schritt für Schritt, mit allen Stolpersteinen.'
tags: ['M365', 'MCP', 'Copilot Studio', 'Azure Container Apps', 'Managed Identity', 'Microsoft Graph', 'OAuth', 'Tutorial']
categories: ['Azure']
cover:
  image: 'https://the-raccoon-way.de/images/Artikel_M365MCP.png'
  alt: 'M365 MCP Server mit Managed Identity'
  hiddenInList: true
---

Oskar sollte Teams-Gruppen erstellen, SharePoint-Seiten anlegen, Planner-Tasks verwalten — alles, was bei Projektstart so anfällt. Klingt machbar. Die Theorie ist schnell erklärt: MCP Server in Azure, Managed Identity gegen die Graph API, OAuth für den Zugang aus Copilot Studio. Die Praxis hatte dann noch ein paar Meinungen dazu.

Dieser Artikel ist die vollständige Installationsanleitung — Schritt für Schritt, in der richtigen Reihenfolge, mit allem was ich auf die harte Tour gelernt habe.

Am Ende läuft euer M365 MCP Server in Azure Container Apps, 29 Tools für Teams, SharePoint, Planner, Outlook, Kalender und Dateien — und Copilot Studio spricht sauber per OAuth dagegen.

## Was dieses Setup ausmacht

Zwei Authentifizierungsrichtungen, zwei unterschiedliche Lösungen:

```
GitHub Repo (d-zurwerra/mcp-m365-graph-server)
    │  GitHub Actions → Push auf main
    ▼
GHCR (ghcr.io/d-zurwerra/mcp-m365-graph-server:latest)
    ▼
Azure Container App  ca-mcp-m365
    │  System-assigned Managed Identity
    │  → Graph API (keine Secrets, kein Ablaufdatum)
    │
    │  OAuth2 Middleware (oauth_middleware.py)
    │  → validiert Entra ID Tokens eingehender Requests
    ▼
Microsoft Graph API
    ▼
Copilot Studio Agent (Oskar / M365 Sub-Agent)
    │  OAuth mit Client Secret
    ▼
Endnutzer
```

| Strecke | Methode | Läuft ab? |
|---|---|---|
| Container App → Graph API (ausgehend) | System-assigned Managed Identity | ❌ nie |
| Copilot Studio → Container App (eingehend) | OAuth 2.0 / Entra ID, Client Secret | ✅ nach 24 Monaten |

Der ausgehende Teil — die Graph API Anbindung — braucht keine Secrets. Die Azure-Infrastruktur verwaltet das selbst. Der eingehende Teil, damit Copilot Studio überhaupt mit dem Server reden darf, braucht ein Client Secret. Das läuft nach 24 Monaten ab und muss erneuert werden. Wer das nicht weiss, wacht irgendwann mit einem Agenten auf, der keine Antworten mehr gibt — ohne brauchbare Fehlermeldung.

## Reihenfolge

Die Phasen bauen aufeinander auf. Wer mittendrin anfängt, sitzt irgendwann ohne einen Wert da, den er früher hätte notieren sollen.

1. Repository forken
2. GitHub Actions prüfen, Image bauen
3. Classic PAT anlegen *(wird beim Anlegen der Container App gebraucht)*
4. Container App anlegen
5. Managed Identity aktivieren
6. Graph API Permissions per PowerShell vergeben
7. Entra App Registration anlegen inkl. Scope und Client Secret
8. Umgebungsvariablen in der Container App eintragen *(jetzt sind alle Werte bekannt)*
9. Copilot Studio Agent konfigurieren
10. Redirect URI in der Entra App eintragen
11. Testen

## Was wir brauchen

| Was | Wo |
|---|---|
| Global Admin im M365 Tenant | entra.microsoft.com |
| Azure Subscription mit Contributor-Rechten | portal.azure.com |
| Zugriff auf GitHub Repo `d-zurwerra/mcp-m365-graph-server` | github.com |
| Copilot Studio Lizenz (M365 Copilot oder Standalone) | admin.microsoft.com |
| PowerShell mit Microsoft.Graph Modul | lokal oder Azure Cloud Shell |

---

## Phase 1 — Repository forken

→ **github.com/d-zurwerra/mcp-m365-graph-server** → **Fork**

Das Repository landet damit in eurem eigenen GitHub Account. Alle weiteren Schritte beziehen sich auf euren Fork.

Den vollständigen Code gibt es hier: [d-zurwerra/mcp-m365-graph-server](https://github.com/d-zurwerra/mcp-m365-graph-server). Die wichtigsten Dateien im Überblick weiter unten im Abschnitt [Code](#code).

---

## Phase 2 — GitHub Actions & Image bauen

### 2.1 Workflow Permissions prüfen

→ Euer Fork → **Settings → Actions → General → Workflow permissions**

Einstellung muss sein: **Read and write permissions** ✅

Ohne diese Einstellung schlägt der Push nach GHCR mit einem 403 fehl.

![GitHub Repo Settings – Workflow permissions mit aktivierter "Read and write permissions" Option](images/M365MCP_GitHubPermissions.png "GitHub Actions Permissions")

### 2.2 Ersten Build triggern

Entweder einen leeren Commit pushen oder direkt manuell starten:

→ **Actions → Build & Deploy MCP Server → Run workflow → Run workflow**

### 2.3 Build prüfen

→ **Actions** → Build muss ✅ grün sein

→ `github.com/<dein-username>?tab=packages` → Paket `mcp-m365-graph-server` mit Tag `latest` muss sichtbar sein

![GitHub Packages Übersicht mit sichtbarem Paket mcp-m365-graph-server:latest](images/M365MCP_GHCR.png "Image in GHCR")

> **Erst wenn das Image in GHCR sichtbar ist — weiter mit Phase 3.**

---

## Phase 3 — Classic PAT anlegen

Der PAT wird an einer einzigen Stelle gebraucht: als Registry-Passwort, damit Azure Container Apps das Image aus GHCR ziehen kann — direkt im nächsten Schritt.

> **Warum Classic PAT?** Fine-grained PATs unterstützen den `packages:read` Scope nicht. Es muss ein Classic PAT sein.

→ **github.com → Profilbild oben rechts → Settings → Developer settings → Personal access tokens → Tokens (classic) → Generate new token (classic)**

| Feld | Wert |
|---|---|
| Note | `ghcr-read-aca` |
| Expiration | 90 days (oder länger) |
| Scopes | ☑ `read:packages` |

→ **Generate token** → Token sofort kopieren — wird gleich in Phase 4 gebraucht.

![GitHub – Classic PAT Erstellungsseite mit markiertem read:packages Scope](images/M365MCP_PAT.png "Classic PAT erstellen")

> ⚠️ Wenn der PAT irgendwann abläuft, schlägt der Image Pull beim nächsten Neustart der Container App fehl. Rechtzeitig erneuern und in der Container App unter **Containers → Edit and deploy** aktualisieren.

---

## Phase 4 — Container App anlegen

> **Wo:** https://portal.azure.com

### 4.1 Resource Group

Suche oben: **Resource groups** → **+ Create**

```
Name:   rg-oskar-mcp
Region: West Europe
```

→ **Review + Create → Create**

### 4.2 Container Apps Environment

Suche: **Container Apps Environments** → **+ Create**

| Feld | Wert |
|---|---|
| Name | `cap-env-oskar` |
| Region | West Europe |
| Log Analytics | neu erstellen: `law-oskar` |
| Workload profiles | Consumption only |
| Networking | Public, kein Custom VNet |

→ **Review + Create → Create**

### 4.3 Container App

Suche: **Container Apps** → **+ Create**

**Tab Basics:**

| Feld | Wert |
|---|---|
| Name | `ca-mcp-m365` |
| Region | West Europe |
| Environment | `cap-env-oskar` |

**Tab Container:**

Den vorausgefüllten Quickstart-Container entfernen und neu eintragen:

| Feld | Wert |
|---|---|
| Image source | Other registries |
| Image type | Private |
| Registry login server | `ghcr.io` |
| Username | `<dein GitHub Username>` |
| Password | Classic PAT aus Phase 3 |
| Image and tag | `<dein-username>/mcp-m365-graph-server:latest` |
| CPU | 0.5 |
| Memory | 1Gi |

> ⚠️ `ghcr.io` kommt **nur** ins Registry-Feld. Im Image-Feld steht **nur** `<username>/mcp-m365-graph-server:latest` — kein `ghcr.io/` davor. Andernfalls entsteht `ghcr.io/ghcr.io/...` und der Pull schlägt fehl.

![Container App Wizard – Tab Container mit korrekt ausgefüllten Registry- und Image-Feldern](images/M365MCP_Container.png "Container konfigurieren")

**Tab Ingress:**

| Feld | Wert |
|---|---|
| Ingress | Enabled |
| Traffic | Accepting traffic from anywhere |
| Type | HTTP |
| Transport | Auto |
| Insecure connections | Not allowed |
| Target port | `8000` |

**Tab Scale:**

| Feld | Wert |
|---|---|
| Min replicas | 0 |
| Max replicas | 1 |

→ **Review + Create → Create**

### 4.4 Application URL notieren

Nach der Erstellung auf der **Overview-Seite** der Container App die **Application URL** kopieren:

```
https://ca-mcp-m365.<random>.westeurope.azurecontainerapps.io
```

Wird in Phase 9 (Copilot Studio) gebraucht.

---

## Phase 5 — Managed Identity aktivieren

→ Azure Portal → `ca-mcp-m365` → **Settings → Identity**

→ Tab **System assigned** → Status auf **On** → **Save**

→ **Object (Principal) ID** notieren — wird direkt in Phase 6 gebraucht.

![Identity-Tab der Container App mit Status "On" und sichtbarer Object (Principal) ID](images/M365MCP_Identity.png "Managed Identity aktivieren")

---

## Phase 6 — Graph API Permissions vergeben

Application Permissions für Managed Identities lassen sich nicht über das Entra-Portal vergeben — dafür gibt es keine UI. Das Skript läuft lokal (mit installiertem Microsoft.Graph Modul) oder direkt in der Azure Cloud Shell unter https://shell.azure.com.

```powershell
# Microsoft Graph Modul installieren (falls nicht vorhanden)
Install-Module Microsoft.Graph -Scope CurrentUser

# Mit dem Tenant verbinden
Connect-MgGraph -Scopes "AppRoleAssignment.ReadWrite.All","Application.Read.All"

# Object ID der Managed Identity (aus Phase 5)
$managedIdentityId = "<OBJECT_ID_AUS_PHASE_5>"

# Microsoft Graph Service Principal holen
$graphSP = Get-MgServicePrincipal -Filter "AppId eq '00000003-0000-0000-c000-000000000000'"

# Permissions zuweisen
$permissions = @(
    "Channel.Create",
    "Chat.ReadWrite.All",
    "Group.ReadWrite.All",
    "Sites.Manage.All",
    "Sites.ReadWrite.All",
    "Tasks.ReadWrite.All",
    "Team.Create",
    "TeamMember.ReadWrite.All",
    "TeamSettings.ReadWrite.All",
    "TeamsAppInstallation.ReadWriteForTeam.All",
    "User.Read.All"
)

foreach ($permission in $permissions) {
    $appRole = $graphSP.AppRoles | Where-Object { $_.Value -eq $permission }
    if ($appRole) {
        New-MgServicePrincipalAppRoleAssignment `
            -ServicePrincipalId $managedIdentityId `
            -PrincipalId $managedIdentityId `
            -ResourceId $graphSP.Id `
            -AppRoleId $appRole.Id
        Write-Host "Assigned: $permission"
    } else {
        Write-Host "Nicht gefunden: $permission"
    }
}
```

> ℹ️ Falls `Connect-MgGraph` beim ersten Aufruf nach einer Consent-Seite fragt — Browser öffnen, einloggen, danach das Skript erneut ausführen.

---

## Phase 7 — Entra App Registration

> **Wo:** https://entra.microsoft.com → Applications → App registrations

Diese App Registration macht zwei Dinge: Sie schützt den MCP Endpoint (die OAuth Middleware prüft eingehende Tokens dagegen) und stellt den Rahmen für die Copilot Studio Authentifizierung.

### 7.1 App registrieren

→ **+ New registration**

| Feld | Wert |
|---|---|
| Name | `oskar-mcp-server-api` |
| Supported account types | Single tenant |

→ **Register**

Auf der Overview-Seite direkt notieren:
- **Application (client) ID** → wird als `ENTRA_CLIENT_ID` in Phase 8 gesetzt
- **Directory (tenant) ID** → wird als `ENTRA_TENANT_ID` in Phase 8 gesetzt

> ℹ️ Die **Application (client) ID** hier ist die ID der App Registration — nicht die Object ID der Managed Identity aus Phase 5. Zwei verschiedene Dinge.

### 7.2 App ID URI setzen

→ **Expose an API → Set**

```
Application ID URI: api://<ENTRA_CLIENT_ID>
```

→ **Save**

### 7.3 Scope anlegen

→ **Expose an API → + Add a scope**

| Feld | Wert |
|---|---|
| Scope name | `access_as_agent` |
| Who can consent | Admins and users |
| Admin consent display name | Access M365 MCP Server as agent |
| Admin consent description | Allows Copilot Studio to call the M365 MCP Server |
| State | Enabled |

→ **Add scope**

### 7.4 Copilot Studio als authorisierten Client eintragen

→ **Expose an API → + Add a client application**

| Feld | Wert |
|---|---|
| Client ID | `1fec8e78-bce4-4aaf-ab1b-5451cc387264` |
| Authorized scopes | `access_as_agent` anhaken |

→ **Add application**

> ℹ️ Die Client ID `1fec8e78-bce4-4aaf-ab1b-5451cc387264` ist die feste System-ID von Copilot Studio — nicht ändern.

### 7.5 Admin Consent erteilen

→ **API permissions → Grant admin consent for [Tenant]** → **Yes**

Alle Einträge zeigen danach ✅ **Granted for [Tenant]**.

### 7.6 Client Secret erstellen

→ **Certificates & secrets → Client secrets → + New client secret**

| Feld | Wert |
|---|---|
| Description | `copilot-studio` |
| Expires | 24 months |

→ **Add** → **Secret Value sofort kopieren** — wird in Phase 9 gebraucht und danach nicht mehr angezeigt.

> ⚠️ Wenn das Secret nach 24 Monaten abläuft, verliert Copilot Studio die Verbindung zum MCP Server — ohne brauchbare Fehlermeldung im Agent, nur eine 401 im Container Log. Ablaufdatum irgendwo notieren.

---

## Phase 8 — Umgebungsvariablen in der Container App

Jetzt haben wir alle Werte beisammen und können die Container App vollständig konfigurieren.

> **Wo:** Azure Portal → `ca-mcp-m365` → **Containers → Edit and deploy → Tab Container → Environment variables**

| Variable | Typ | Wert | Woher |
|---|---|---|---|
| `ENTRA_TENANT_ID` | Value | Directory (tenant) ID | Entra → `oskar-mcp-server-api` → Overview |
| `ENTRA_CLIENT_ID` | Value | Application (client) ID | Entra → `oskar-mcp-server-api` → Overview |
| `PORT` | Value | `8000` | fest |

→ **Create** (unten) → neue Revision wird automatisch erstellt und deployed.

> ⚠️ Wenn `ENTRA_TENANT_ID` oder `ENTRA_CLIENT_ID` fehlt, startet der Container ohne Fehlermeldung — aber auch ohne OAuth-Schutz. In den Logs erkennbar an: `ENTRA_TENANT_ID oder ENTRA_CLIENT_ID nicht gesetzt – OAuth deaktiviert`

---

## Phase 9 — Copilot Studio Agent konfigurieren

> **Wo:** https://copilotstudio.microsoft.com

### 9.1 Neuen Agent anlegen

→ **Agents → + New agent**

| Feld | Wert |
|---|---|
| Name | `Oskar-M365` |
| Solution | Oskar (bestehende Solution) oder neu anlegen |

### 9.2 MCP Server als Action hinzufügen

→ Agent `Oskar-M365` → **Actions → + Add an action → Model Context Protocol**

| Feld | Wert | Woher |
|---|---|---|
| Server URL | `https://ca-mcp-m365.<random>.westeurope.azurecontainerapps.io/mcp` | Application URL aus Phase 4.4 + `/mcp` |
| Authentication | OAuth | – |
| Auth type | Manuell | – |

Danach erscheint das OAuth-Formular:

| Feld | Wert | Woher |
|---|---|---|
| Client-ID | `<ENTRA_CLIENT_ID>` | Entra → `oskar-mcp-server-api` → Overview |
| Geheimer Clientschlüssel | Secret Value aus Phase 7.6 | Zwischengespeicherter Wert |
| Autorisierungs-URL | `https://login.microsoftonline.com/<ENTRA_TENANT_ID>/oauth2/v2.0/authorize` | Tenant ID aus Phase 7.1 |
| Token-URL-Vorlage | `https://login.microsoftonline.com/<ENTRA_TENANT_ID>/oauth2/v2.0/token` | wie oben |
| URL aktualisieren | `https://login.microsoftonline.com/<ENTRA_TENANT_ID>/oauth2/v2.0/token` | gleicher Wert wie Token-URL |
| Bereiche | `api://<ENTRA_CLIENT_ID>/access_as_agent` | Client ID aus Phase 7.1 |
| Umleitungs-URL | *(nicht ausfüllen)* | wird automatisch generiert → Phase 10 |

→ **Speichern**

> ℹ️ **Warum Client Secret und nicht PKCE?** Copilot Studio müsste sich für PKCE zuerst am `/register` Endpoint des MCP Servers anmelden — die OAuth Middleware blockt diesen Call aber, weil er selbst keinen Token mitbringt. Henne-Ei-Problem. Copilot Studio bietet keine Option ohne Client Secret, das ist eine Plattformbeschränkung.

![Copilot Studio – MCP Action OAuth-Formular mit ausgefüllten Feldern](images/M365MCP_CopilotStudio_OAuth.png "OAuth Konfiguration")

### 9.3 Tools aktivieren & publizieren

→ Alle Tools aktivieren (Master-Toggle oben links) → **Save → Publish**

![Copilot Studio – MCP Action mit aufgelöster Tool-Liste (alle aktiviert)](images/M365MCP_CopilotStudio_Tools.png "Tools aktiviert")

---

## Phase 10 — Redirect URI in Entra eintragen

Entra ID erlaubt einen OAuth-Login nur, wenn die Ziel-URL explizit in der App Registration eingetragen ist. Copilot Studio generiert diese URL automatisch — sie ist erst nach dem Speichern der Action in Phase 9 sichtbar.

**Schritt 1:** In der gespeicherten MCP Action die **Umleitungs-URL** kopieren. Das Format sieht ungefähr so aus:

```
https://europe.api.advisor.microsoft.com/redirection
```

**Schritt 2:** → entra.microsoft.com → `oskar-mcp-server-api` → **Authentication**

Falls noch keine Web-Plattform vorhanden: **+ Add a platform → Web** → URI einfügen → **Configure**

Falls Web-Plattform bereits vorhanden: **Redirect URIs → + Add URI** → URI einfügen → **Save**

> ⚠️ Ohne diesen Schritt schlägt jeder Login-Versuch in Copilot Studio mit `AADSTS50011: The redirect URI specified in the request does not match` fehl. Der Agent zeigt dabei nur eine generische Fehlermeldung — der eigentliche Grund steckt tief im Entra-Log.

![Entra – Authentication-Seite mit eingetragener Redirect URI aus Copilot Studio](images/M365MCP_RedirectURI.png "Redirect URI nachtragen")

---

## Phase 11 — Testen

Im Browser:

```
https://ca-mcp-m365.<random>.westeurope.azurecontainerapps.io/mcp
```

Erwartete Antwort (OAuth Middleware aktiv):

```json
{"jsonrpc":"2.0","id":"auth-error","error":{"code":-32600,"message":"Unauthorized: Bearer token required"}}
```

> ℹ️ Diese 401-Antwort ist das **erwartete Verhalten** — kein Fehler. Wenn stattdessen eine leere Seite oder ein 500 kommt, in den Container Logs nachschauen.

In Copilot Studio:

→ **Test** → Nachricht: `Zeig mir alle M365 Gruppen`

Wenn der Agent nach einer Anmeldung fragt: anmelden — danach sollte die Gruppenübersicht kommen.

### Als Sub-Agent im Oskar Orchestrator einbinden

Im Oskar Orchestrator-Agent:

→ **Actions → + Add an action → Agent** → `Oskar-M365`

Der Orchestrator leitet M365-bezogene Anfragen automatisch weiter.

---

## Code

Den vollständigen Code gibt es im Repository: [d-zurwerra/mcp-m365-graph-server](https://github.com/d-zurwerra/mcp-m365-graph-server). Die wichtigsten Dateien:

### auth.py

```python
"""
auth.py – Managed Identity Auth für Microsoft Graph

Der MCP Server läuft in Azure Container Apps mit System-assigned Managed Identity.
Die Managed Identity hat die benötigten Graph API Permissions (Phase 6 des Setups).

Kein Client Secret, kein Tenant ID, keine Umgebungsvariablen nötig –
Azure verwaltet die Identität automatisch.
"""

import logging
from azure.identity import ManagedIdentityCredential
from azure.core.exceptions import ClientAuthenticationError

logger = logging.getLogger("oskar-mcp-server.auth")

GRAPH_SCOPE = "https://graph.microsoft.com/.default"

_credential = ManagedIdentityCredential()


async def get_graph_token() -> str:
    """Holt einen Graph Access Token via Managed Identity."""
    try:
        token = _credential.get_token(GRAPH_SCOPE)
        return token.token
    except ClientAuthenticationError as e:
        logger.error(f"Managed Identity Auth Fehler: {e}")
        raise RuntimeError(f"Token Exchange fehlgeschlagen: {e}") from e


async def get_graph_headers() -> dict:
    """Gibt fertige Authorization Headers für Graph API Calls zurück."""
    token = await get_graph_token()
    return {
        "Authorization": f"Bearer {token}",
        "Content-Type": "application/json",
    }
```

### Dockerfile

```dockerfile
FROM python:3.12-slim

WORKDIR /app

RUN apt-get update && apt-get install -y --no-install-recommends \
    curl \
    && rm -rf /var/lib/apt/lists/*

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app/ ./app/

RUN useradd -m -u 1000 mcpuser
USER mcpuser

EXPOSE 8000

HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:8000/health || exit 1

CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000", "--forwarded-allow-ips", "*"]
```

### requirements.txt

```
mcp[cli]>=1.5.0
httpx>=0.27.0
azure-identity>=1.16.0
uvicorn>=0.29.0
starlette>=0.37.0
PyJWT>=2.8.0
cryptography>=41.0.0
```

---

## Laufender Betrieb

### Client Secret erneuern (alle 24 Monate)

1. Entra → `oskar-mcp-server-api` → **Certificates & secrets → + New client secret** → Value kopieren
2. copilotstudio.microsoft.com → Agent `Oskar-M365` → **Actions → MCP Action → Edit**
3. Client Secret Feld überschreiben → **Save → Publish**
4. Altes Secret in Entra löschen

### Classic PAT erneuern

1. Neuen Classic PAT mit `read:packages` generieren (wie Phase 3)
2. Azure Portal → `ca-mcp-m365` → **Containers → Edit and deploy**
3. Container auswählen → Passwort-Feld mit neuem PAT überschreiben → **Create**

### Neues Image deployen

Der Deploy-Job im Workflow ist bewusst auskommentiert — Azure wird manuell aktualisiert:

1. Code auf `main` pushen → GitHub Actions Build abwarten ✅
2. Azure Portal → `ca-mcp-m365` → **Revisions and replicas → + Create new revision**
3. **Name/Suffix** eintragen (z.B. `v2`) → **Create**

> ⚠️ Der Create-Button ist ausgegraut, bis das Name/Suffix-Feld ausgefüllt ist — auch wenn es optional wirkt.

### Logs einsehen

| Wo | Was |
|---|---|
| `ca-mcp-m365` → Revision → Logs → **Real-time** | Live-Output (Tool-Aufrufe, Auth-Logs) |
| `ca-mcp-m365` → Revision → Logs → **Historical → Application** | Container-Logs (Fehler, Starts) |
| `ca-mcp-m365` → Revision → Logs → **Historical → System** | Image-Pull, Startfehler |

---

## Stolpersteine im Überblick

**Fine-grained PAT für GHCR.**
Funktioniert nicht für `packages:read`. Classic PAT, nicht Fine-grained.

**`ghcr.io` doppelt im Image-Pfad.**
Registry-Feld = `ghcr.io`, Image-Feld = `<username>/mcp-m365-graph-server:latest`. Kein `ghcr.io/` im Image-Feld.

**OAuth Middleware startet ohne Fehler, aber auch ohne Schutz.**
Wenn `ENTRA_TENANT_ID` oder `ENTRA_CLIENT_ID` fehlt, meldet der Container nichts — er startet einfach ungeschützt. In den Logs erkennbar.

**Graph API gibt 401.**
Managed Identity aktiviert, aber PowerShell-Skript noch nicht ausgeführt — oder Container App nach der Ausführung nicht neu gestartet.

**Copilot Studio OAuth-Login schlägt fehl (AADSTS50011).**
Redirect URI fehlt in der App Registration. Phase 10 nacharbeiten.

**Bereiche-Feld in Copilot Studio.**
Nicht nur `access_as_agent` eintragen, sondern `api://<ENTRA_CLIENT_ID>/access_as_agent`. Ohne den `api://`-Prefix und die Client ID kommt kein gültiger Token.

**Create-Button für neue Revision ausgegraut.**
Name/Suffix-Feld muss ausgefüllt sein, auch wenn es optional wirkt.

---

## Infrastruktur-Übersicht

| Ressource | Name | Region |
|---|---|---|
| Resource Group | `rg-oskar-mcp` | West Europe |
| Container Apps Environment | `cap-env-oskar` | West Europe |
| Log Analytics Workspace | `law-oskar` | West Europe |
| Container App | `ca-mcp-m365` | West Europe |
| Entra App Registration | `oskar-mcp-server-api` | – |
| GHCR Image | `ghcr.io/<username>/mcp-m365-graph-server:latest` | – |

---

Bei mir erstellt Oskar inzwischen auf Zuruf Teams-Gruppen, legt SharePoint-Seiten an und pflegt Planner-Boards. Bis es so weit war, hat er vor allem Fehlermeldungen produziert — aber das ist der Lerneffekt, den dieser Artikel euch hoffentlich erspart.

Was ihr mit eurem M365 Agent baut, interessiert mich. Schreibt mir auf [LinkedIn](https://www.linkedin.com/in/danielazurwerra/) — ich beisse auch nicht.

![Oskar in Aktion – z.B. beim Erstellen einer Teams-Gruppe via Chat-Nachricht](images/M365MCP_Oskar_Action.png "Oskar erstellt eine Teams-Gruppe")
