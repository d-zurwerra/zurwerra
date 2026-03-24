---
title: 'Jira trifft Power Automate – wenn der Standard Connector nicht reicht'
date: '2026-03-24'
draft: false
description: 'Wie man mit HTTP Requests statt dem Standard Connector Jira Formulare ausliest und die Daten in Power Automate weiterverarbeitet – inklusive Lösungsansatz und praktischem Beispiel.'
tags: ['Power Automate', 'Jira', 'HTTP Request', 'Tutorial', 'API']
categories: ['Power Platform']
---
# Ausgangssituation

Stell dir vor, du hast in deinem Unternehmen schon einen vollständig automatisierten Prozess, um ein Teams Team zu erstellen. Einfach ein Jira-Ticket ausfüllen, in dem ein paar Angaben abgefragt werden - schon steht wenig später das Teams Team bereit - immer vorausgesetzt, der Vorgang wird per Approval auch genehmigt.

Wie cool ist das denn?

Bei meinem Kunden wurde genau das bereits realisiert - und nun gab es eine erweiterte Anforderung, die wir mit Power Automate und Azure Functions umgesetzt haben.

# Neue Anforderung

Wenn für das Team "Projektmanagement" als Vorlage ausgewählt wurde, sollen weitere Schritte erfolgen:

- Sharepoint Site nach Vorlage erstellen
- Kopieren der Ordner und Dokumente in die neue SharePoint Site
- Planner und zugehörige Buckets erstellen
- Die Webparts sollen auf die neuen Ordner bzw. Dokumente verweisen

Bislang wurden diese Steps von einem Admin erledigt, sobald das Team erstellt war, was nicht nur zeitaufwändig für den Admin ist, sondern auch den ganzen Prozess verzögert hat.
# Der neue Prozess in zwei Teilen

Auf Kundenseite wurde eine Azure Logic App erstellt, die die benötigten Schritte abdecken soll. Als Trigger fungiert ein HTTP Request, über welchen der Flow die Group ID des neuen Teams mitbekommt und alles weitere eigenständig erledigt - aber wie bekommt der Trigger die Info, dass ein neues Ticket zur Bearbeitung vorliegt?

> ℹ️ **Hinweis:** Auf die Azure Logic App gehe ich in einem späteren Post genauer ein

## Schon mal von Jira Forms gehört?

Relativ schnell war klar, die Info zum Start (inkl. der GUID) muss von Jira kommen. Also habe ich mir angeschaut, wo die Info im Ticket steht und was Power Automate im Standard hergibt.

Jira kann eigene Formulare erstellen und mit diesen arbeiten. Sieht dann wie folgt in Jira aus:

![Jira Form, in dem Vorlage, Besitzer etc. abgefragt werden](/images/Pasted_image_20260324143213.png "Screenshot Jira Form")

Das ist grossartig, wenn es ums standardisierte Eingaben geht, allerdings etwas hinderlich, wenn per Jira Connector in Power Automate die Information ausgelesen werden soll. Denn: Der Connector kennt keine Forms.

Daher bin ich einen anderen Weg gegangen: Jira bietet eine REST API Schnittstelle an, die mir zur Lösung verhalf. Wenn auch nicht mit einem Klick. Aber: dann wäre es ja langweilig, oder?
# Die Lösung

## Der Start ist einfach

Zum Start erstelle ich einen scheduled Flow, der täglich um 9 Uhr los läuft.
## Nun kommt Jira ins Spiel

### Secret

Wenn ich auf Jira per REST API zugreifen will, benötige ich ein Secret. Dieses kann ich mir in meinem Fall direkt in Jira erstellen. Dazu gehe ich meine Accountsettings, wechsle in den Reiter Security und erstell mir dann ein neues Secret
  ![Secret Erstellung über Jira, Accountsettings und Security](/images/Pasted_image_20260323192214.png "Screenshot Secret für Jira")
### Variablen

Die Infos gebe ich dann auch direkt im Power Automate Flow als Variable ein, damit ich diese nicht an x Stellen immer wieder eingeben und ggf. ändern muss.

Ausserdem werde ich als weiteres noch die URL der REST-Gegenseite (https://contoso.atlassian.net/rest/servicedeskapi) häufiger benötigen, daher definiere ich diese ebenfalls direkt:

![Definition der REST URL für Power Automate](/images/Pasted_image_20260323192333.png "Screenshot VariablenDefinition")

### Alle Tickets abrufen

Jedes Teams- (und damit Projektmanagement-) Ticket hat den gleichen Betreff und landet im gleichen Team. Dadurch kann ich nun alle Tickets in unserem Team abrufen, die den Betreff 'Microsoft Teams erstellen' haben und noch nicht auf dem Status 'done' sind. Damit finde ich alle für uns relevanten Tickets und kann mit diesen Informationen weiter arbeiten.

![POST JiraURL/search/jql](/images/Pasted_image_20260323192709.png "Screenshot URI und Method zum Ticket Abruf")

![Body zum Ticketabruf](/images/Pasted_image_20260323192729.png "Screenshot Body zum Ticketabruf")

Die Authentifizierung läuft bei allen HTTP-Aktionen über Basic Auth:
![BasicAuthentifizierung mit Jira Usernamen und Secret](/images/Pasted_image_20260323192801.png "Screenshot Authentifizierung")

Die Antwort auf diese Aktion parse ich einmal, damit die nachfolgenden Steps einfacher werden:

![Parse Ergebnis aus dem HTTP Request](/images/Pasted_image_20260323192958.png "Screenshot Parse HTTP Body")

### Form abrufen

Zuerst benötige ich die Tickets, die als Vorlage Projektmanagement ausgewählt haben. Diese Info finde ich in der Jira Form und nicht direkt im Ticket. Daher besteht der nächste Step darin, dass ich aus jedem Ticket aus dem letzten Step herausauslese, wie die ID der verwendeten Form ist. Denn: nur so komm ich an die Antworten aus dieser Form heran.

Heisst: ich benötige zwei Schritte.

1. um die ID zu finden:

![URI auf Jira/forms/cloud/CloudID/issue/key/form](/images/Pasted_image_20260323193033.png "Screenshot Abruf Form ID")

2. um dann die Antwort auszulesen:
![GET auf Jira/forms/cloud/CloudID/issue/key/form/FormID/format/answers](/images/Pasted_image_20260323193112.png "Screenshot Get Forms Answer")
### Filtern auf Projektmanagement-Tickets

Wenn ich die Antworten habe, gehe ich diese Step by Step durch und prüfe, ob es sich bei diesem Ticket um ein Ticket mit Vorlage "Projektmanagement" handelt. Bei mir sieht das dann so aus:

![Condition, in der geprüft wird, ob als Antwort "Projektmanagement" steht](/images/Pasted_image_20260323193215.png "Screenshot Condition template = Projektmanagement")

```bash
body('Filter_array:_Answers')[0]['answer']
```

Falls es sich nun um solches Ticket handelt, mache ich im Flow weiter - ansonsten ignoriere ich das Ticket.

### Approval vorhanden?
Jedes Ticket muss erst von einem festen Personenkreis approved werden, bevor Teams etc. erstellt werden kann. Heisst für mich: wenn kein Approval vorliegt, interessiert mich das Ticket ebenfalls nicht.

Das Approval steht in meinem Fall in den Ticketinformationen. Hierzu ruf ich die Ticketinformationen wie folgt ab:

![Tickets abrufen mit GET JiraURL/issue/key](/images/Pasted_image_20260323194017.png "Screenshot Ticket Infos abrufen")

In der Antwort steht die Approval-Information in einem Customfield (hier das customfield_10045 - das kann man sonst vorab auslesen)

```bash
body('HTTP:_GET_Ticket_Infos')['fields']['customfield_10045'][0]['finalDecision']
```

Wenn hier nun 'approved' drin steht, darf das zugehörige Teams etc. erstellt werden. Dies wird über eine Condition einmal geprüft:

![Condition, ob Final Desicion (code oben) = approved ist](/images/Pasted_image_20260323194550.png "Screenshot Approval")

## Nun nur noch die GUID rauslesen...
Nachdem das Approval durch ist, wird durch einen bereits vorhandenen Prozess das Teams Team erstellt. Dies passiert in drei Schritten, die jeweils in Jira als Kommentar notiert wird:

### Schritt 1: die Eingaben im Formular werden überprüft
Zuerst werden die getätigten Angaben im Formular überprüft, um sicher zu stellen, dass die Emailadressen korrekt sind, dass die User auch interne User sind und dass Owner und Co-Owner unterschiedliche User sind. In Jira sieht das dann wie folgt aus:

![Rückinfo zur Auswertung der genannten Punkte in einem Kommentar gegliedert. Hinter jedem OK ist ein grüner Haken](/images/Pasted_image_20260324144948.png "Screenshot Formular Check")

### Schritt 2: das Team wird erstellt
Der Name des Teams ist ja bereits im Formular eingetragen, dieser wird entsprechend übernommen und intern mit dem Standortkürzel für die M365-Gruppe noch ergänzt.
Der Schritt zur Erstellung des Teams sieht dann in der Doku so aus:

![Rückinfo, dass das Team erstellt wird](/images/Pasted_image_20260324145229.png "Screenshot Team wird erstellt")

### Schritt 3: Team ist erfolgreich erstellt worden
Wenn das Team bzw. die M365 Gruppe erfolgreich erstellt wurde, bekommen wir darüber eine letzte Notiz, die für den weiteren Prozess zwei wichtige Punkte enthält:
- die Info, dass der Prozess erfolgreich durchgeführt wurde
- die ID der M365 Gruppe
![Team erfolgreich erstellt. Inkl. Angaben zu ID, Displayname, Description etc.](/images/Pasted_image_20260324145605.png "Screenshot Team erstellt")

Und genau dieser Kommentar wird für das weitere Vorgehen benötigt. Um ihn auszulesen können, brauchen wir in Power Automate ein paar Steps

### Die GUID in Power Automate auslesen

Die Kommentare als solche werden direkt über die Ticket-Infos von oben ausgeliefert. Wir müssen nur noch aussortieren.

Zuerst prüfe ich, ob der Titel des Kommentars lautet "PROCESSING SUCCESS: Finished to create team" - denn dann weiss ich, ob dass ich beim korrekten Kommentar bin.

![Condition, um zu prüfen, ob im Kommentartitel "Porcessing success, Finished to create team" enthalten st](/images/Pasted_image_20260324150201.png "Screenshot Condition Finished to create team")

Danach muss ich noch überflüssige Zeichen ersetzen,
![Ersetzen von Sonderzeichen, zum weiteren Verarbeiten](/images/Pasted_image_20260324150242.png "Screenshot Replace Sonderzeichen")

das JSON parsen (nicht zwingend notwendig)

![Parse Json aus dem letzten Schritt](/images/Pasted_image_20260324150329.png "Screenshot Parse Json")

... und schon hab ich die GUID direkt aus dem Parse JSON, die ich dann entsprechend für alles weitere verwenden kann

# Wie gehts weiter?
Mit der GUID gehen wir aktuell auf eine Azure Logic App zu, mit der wir die weiteren Schritte durchführen - dieser Teil kommt in einem weiteren Blogartikel dann dran. Seid gespannt drauf.

Und wie immer gilt: wer Fragen, Anregungen oder Wünsche hat: gerne her damit!
