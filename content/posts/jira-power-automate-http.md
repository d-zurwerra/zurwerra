---
title: 'Jira trifft Power Automate – wenn der Standard Connector nicht reicht'
date: '2026-03-12'
draft: true
description: 'Wie man mit HTTP Requests statt dem Standard Connector Jira Formulare ausliest und die Daten in Power Automate weiterverarbeitet – inklusive Lösungsansatz und praktischem Beispiel.'
tags: ['Power Automate', 'Jira', 'HTTP Request', 'Tutorial', 'API']
categories: ['Power Platform']
---
# Ausgangssituation
Stell dir vor, du hast in deinem Unternehmen schon einen vollständig automatisierten Prozess, um ein Teams Team zu erstellen. Einfach ein Jira-Ticket ausfüllen, in dem ein paar Angaben abgefragt werden und schon steht wenig später das Teams Team bereit - immer vorausgesetzt, der Vorgang wird per Approval auch genehmigt.

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
Relativ schnell war klar, die Info zum Start (inkl. der GUID) muss von Jira kommen. Also habe ich mir angeschaut, wo die Info im Ticket steht und was Power Automate im Standard hergibt

Jira kann eigene Formulare erstellen und mit diesen Arbeiten. Das ist grossartig, wenn es ums standartisierte Eingaben geht, allerdings etwas hinderlich, wenn per Jira Connector in Power Automate die Information ausgelesen werden soll. Denn: Der Connector kennt keine Forms. 

Daher bin ich einen anderen Weg gegangen: Jira bietet eine REST API Schnittstelle an, die mir zur Lösung verhalf. Wenn auch nicht mit einem Klick. Aber: dann wäre es ja langweilig, oder?

# Die Lösung
## Der Start ist einfach
Zum Start erstelle ich einen scheduled Flow, der täglich um 9 Uhr los läuft.

## Nun kommt Jira ins Spiel
### Secret
Wenn ich auf Jira per REST API zugreifen will, benötige ich ein Secret. Dieses kann ich mir in meinem Fall direkt in Jira erstellen. Dazu gehe ich meine Accountsettings, wechsle in den Reiter Security und erstell mir dann ein neues Secret

_____ Bild Account Settings ____

### Variablen
Die Infos gebe ich dann auch direkt im Power Automate Flow als Variable ein, damit ich diese nicht an x Stellen immer wieder eingeben und ggf. ändern muss.
Ausserdem werde ich als weiteres noch die URL der REST-Gegenseite (https://contoso.atlassian.net/rest/servicedeskapi) häufiger benötigen, daher definiere ich diese ebenfalls direkt:


_____ Bild Variablen _____

### Alle Tickets abrufen
Jedes Teams- (und damit Projektmanagement-) Ticket hat den gleichen Betreff und landet im gleichen Team. Dadurch kann ich nun alle Tickets in unserem Team abrufen, die den Betreff 'Microsoft Teams erstellen' haben und noch nicht auf dem Status 'done' sind. Damit finde ich alle für uns relevanten Tickets und können mit diesen Informationen weiter arbeiten.

________ Bild Tickets abrufen _______

### Form abrufen
Zuerst benötige ich die Tickets, die als Vorlage Projektmanagement ausgewählt haben. Diese Info finde ich in der Jira Form und nicht direkt im Ticket. Daher besteht der nächste Step darin, dass ich aus jedem Ticket aus dem letzten Step herausauslese, wie die ID der verwendeten Form ist. Denn: nur so komm ich an die Antworten aus dieser Form heran.

Heisst: ich benötige zwei Schritte. 

1. um die ID zu finden

______ Bild get Anwer ID__________

2. um die Antwort auszulesen

_____ Bild Get Answer

### Filtern auf Projektmanagement-Tickets
Nun gehe ich die Antworten durch. Die Antwort auf die Frage des Templates ist im Body versteckt. Bei mir genauer gesagt genau hier:



