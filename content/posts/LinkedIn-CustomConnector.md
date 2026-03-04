---
title: 'LinkedIn CustomConnector'
date: '2026-03-03'
draft: true
description: 'Wie kann ich LinkedIn-Posts per PowerAutomate erstellen'
tags: []
categories: []
cover:
  image: '/zurwerra/images/Artikel_CustomConnector_LinkedIn.png'
  alt: 'LinkedIn Custom Connector'
  hiddenInList: true
---
Mit diesem Custom Connector ist es möglich, ganz normale LinkedIn Post zu erstellen. Sehr von Vorteil, wenn der Content zum Beispiel in einer SharePoint Liste vorbereitet und nur noch hochgepusht werden soll.

> **Hinweis:** Wer den Custom Connector nicht selbst erstellen möchte, findet ihn in Kürze auch in Github zum herunterladen. Die Voraussetzungen müssen allerdings dennoch geschaffen werden, da ansonsten die Gegenseite fehlt

# Voraussetzungen

Um LinkedIn Posts per Custom Connector erstellen zu können, benötigen wir eine registrierte App auf Seiten von LinkedIn, die über die benötigen Berechtigungen verfügt. Dazu gehen wir wie folgt vor:

Wir benötigen Zugang zu einer Company Site. Wer das aktuell noch nicht hat, kann sich über die Seite LinkedIn Unternehmensseite erstellen auch eine erstellen.
Danach benötigen wir eine registrierte App. Hierzu wechseln wir auf die Seite [LinkedIn Developer](https://developer.linkedin.com "LinkedIn Developer Seite") und gehen dann auf **My Apps**. Hier müsst ihr eine neue App erstellen - und dabei auch das Unternehmen unter Punkt 1 verlinken. Ausserdem benötigt ihr einen Namen und ein Icon
Nachdem die App erstellt wurde, speichert euch bitte im Reiter **Auth** die Client ID und das Secret
Letzter Schritt in der Vorbereitung: die Berechtigungen. Ihr benötigt zum Post erstellen die Berechtigungen **Share on LinkedIn** und **Sign In with LinkedIn using OpenID Connect**. Beide Punkte findet ihr unter **Products**.
Damit sind die Vorraussetzungen aus LinkedIn Perspektive schon erledigt.

# Custom Connector

Um den Custom Connector zu erstellen nutzen wir als Grundlage eine Solution.
Darin erstellen wir einen neuen Custom Connector.

![Screenshot vom Erstellen eines Custom Connectors. New, dann Automation und dann Custom Connector auswählen](/zurwerra/images/CustomConnector_erstellen.png "Screenshot Custom Connector erstellen")

## General
Auf der ersten Seite füllen wir den Host und Base aus

| Feld | Wert |
|---|---|
|Host|api.linkedin.com|
|Base|/v2|‚

CustomConnector_General
![Screenshot von der General Seite des Custom Connectors. Man sieht einen Waschbären als Icon, darunter die Description "Custom Connector for posting to LinkedIn via API" und unten drunter die Felder Host und Base, die mit den oben genannten Werten ausgefüllt sind.](/zurwerra/images/CustomConnector_General.png "Screenshot Custom Connector General Reiter")

## Security
Im Reiter **Security** wir **OAuth 2.0** als **Authentication Type**.

Die darauf folgenden Felder werden wie folgt gefüllt:

| Feld | Wert |
|---|---|
|Identity Provider|Generic OAuth 2|
|Client ID| *Wert aus der registrierten App*|
|Client Secret|*Wert aus der registrierten App*|
|Authorization URL|https://www.linkedin.com/oauth/v2/authorization|
|Token URL|https://www.linkedin.com/oauth/v2/accessToken|
|Refresh URL|https://www.linkedin.com/oauth/v2/accessToken|
|Scope|openid profile w_member_social|

Danach wird der CustomConnector per **Create Connector** erstellt, da nur so die Redirect URL am unteren Ende generiert wird.

![Screenshot von der erstellten CustomConnector Redirect URL](/zurwerra/images/CustomConnector_RedirectURL.png "Screenshot Custom Connector Redirect URL")

Diese URL bitte einmal kopieren und in die Redirect URL in der App auf LinkedIn hinzufügen.

![Screenshot von der der eingetragenen Redirect URL in der LinkedIn App](/zurwerra/images/LinkedIn_RedirectURL.png "Screenshot LinkedIn App Redirect URL")

## Definitionen

Als nächsten Step geht es an die Definitionen. Heisst: Welche Aktionen und/oder Träger stellen wir zur Verfügung. In unserem Fall sind es 3 Aktionen:

- **GetUserInfo**: Damit können wir den aktuellen User abfragen und erfahren so unter anderem die UserID, die wir zum Posten eines Artikels benötigen
- **CreatePost**: Damit erstellen wir einen LinkedIn Post mit einem mitgeschickten JSON
- **CreatePostSimple**: Auch hier wird ein LinkedIn Post erstellt. Das JSON wird allerdings benutzerfreundlicher aufbereitet, so dass der User die notwendigen Informationen direkt in Power Automate ausfüllen kann. Wichtig an der Stelle: die Aktion ist in diesem Step noch genau die gleiche - die "Magie" passiert über den Code

![Screenshot von der der Aktion GetUserInfo](/zurwerra/images/CustomConnector_GetUserInfo.png "Screenshot GetUserInfo")

![Screenshot von der der Aktion CreatePost](/zurwerra/images/CustomConnector_CreatePost.png "Screenshot CreatePost")

![Screenshot von der der Aktion CreatePostSimple](/zurwerra/images/CustomConnector_CreatePost_Simple.png "Screenshot CreatePostSimple")

> **Hinweis:** Die einzelnen Aktionen könnt ihr über den Swagger-Editor mit den nachfolgenden Zeilen einfügen. Hierzu einfach den Punkt Path mit dem Code ersetzen. Den kompletten Swagger-Eintrag gibt es ebenfalls im Repository auf Github


```yaml
paths:
  /userinfo:
    get:
      summary: Get User Info
      description: Retrieve user profile information including person ID
      operationId: GetUserInfo
      parameters: []
      responses:
        '200':
          description: Success
          schema:
            type: object
            properties:
              sub:
                type: string
                description: User ID (Person ID)
                x-ms-summary: Person ID
              name:
                type: string
                description: Full name
                x-ms-summary: Name
              email:
                type: string
                description: Email address
                x-ms-summary: Email
              given_name:
                type: string
                description: First name
                x-ms-summary: First Name
              family_name:
                type: string
                description: Last name
                x-ms-summary: Last Name
              picture:
                type: string
                description: Profile picture URL
                x-ms-summary: Picture
        default:
          description: Error
          schema:
            type: object
  /ugcPosts:
    post:
      summary: Create LinkedIn Post
      description: Post content to LinkedIn. Provide the complete JSON body for the post.
      operationId: CreatePost
      consumes:
        - application/json
      produces:
        - application/json
      parameters:
        - name: Content-Type
          in: header
          required: true
          type: string
          default: application/json
          x-ms-visibility: internal
        - name: X-Restli-Protocol-Version
          in: header
          required: true
          type: string
          default: 2.0.0
          x-ms-visibility: internal
        - name: body
          in: body
          required: true
          schema:
            type: object
            x-ms-visibility: important
            description: LinkedIn post JSON body
            example:
              author: urn:li:person:YOUR_PERSON_ID
              lifecycleState: PUBLISHED
              specificContent:
                com.linkedin.ugc.ShareContent:
                  shareCommentary:
                    text: Your post text here
                  shareMediaCategory: NONE
              visibility:
                com.linkedin.ugc.MemberNetworkVisibility: PUBLIC
      responses:
        '201':
          description: Post created successfully
          schema:
            type: object
            properties:
              id:
                type: string
                description: The URN of the created post
                x-ms-summary: Post ID
        default:
          description: Error
          schema:
            type: object
            properties:
              message:
                type: string
                description: Error message
              status:
                type: integer
                description: HTTP status code
      x-ms-visibility: important
  /ugcPosts/simple:
    post:
      summary: Create LinkedIn Post (Simple)
      description: >-
        Post to LinkedIn with simple input fields - ideal for non-technical
        users.  Supports text posts and link sharing with preview cards.
      operationId: CreatePostSimple
      consumes:
        - application/json
      produces:
        - application/json
      parameters:
        - name: Content-Type
          in: header
          required: true
          type: string
          default: application/json
          x-ms-visibility: internal
        - name: X-Restli-Protocol-Version
          in: header
          required: true
          type: string
          default: 2.0.0
          x-ms-visibility: internal
        - name: personId
          in: query
          required: true
          type: string
          description: Your Person ID (the "sub" value from Get User Info, e.g., abc123xyz)
          x-ms-summary: Person ID
        - name: postText
          in: query
          required: true
          type: string
          description: >-
            The text content of your LinkedIn post. URLs in text are
            automatically clickable  (e.g., "Check this out:
            https://example.com"). For link preview cards, use Article URL
            below.
          x-ms-summary: Post Text
        - name: articleUrl
          in: query
          required: false
          type: string
          description: >-
            Optional: URL to share with a preview card (image, title,
            description).  Leave empty for text-only posts. Example:
            https://example.com/my-article
          x-ms-summary: Article URL (optional)
          x-ms-visibility: advanced
        - name: articleTitle
          in: query
          required: false
          type: string
          description: >-
            Optional: Title for the article preview card. Only used if Article
            URL is provided. Example: "The Future of AI in Business"
          x-ms-summary: Article Title (optional)
          x-ms-visibility: advanced
        - name: articleDescription
          in: query
          required: false
          type: string
          description: >-
            Optional: Description for the article preview card. Only used if
            Article URL is provided. Example: "A comprehensive guide to
            leveraging AI..."
          x-ms-summary: Article Description (optional)
          x-ms-visibility: advanced
        - name: visibility
          in: query
          required: false
          type: string
          default: PUBLIC
          enum:
            - PUBLIC
            - CONNECTIONS
          description: Who can see this post
          x-ms-summary: Visibility
        - name: lifecycleState
          in: query
          required: false
          type: string
          default: PUBLISHED
          enum:
            - PUBLISHED
            - DRAFT
          description: Post state
          x-ms-summary: Lifecycle State
          x-ms-visibility: advanced
      responses:
        '201':
          description: Post created successfully
          schema:
            type: object
            properties:
              id:
                type: string
                description: The URN of the created post
                x-ms-summary: Post ID
        default:
          description: Error
          schema:
            type: object
            properties:
              message:
                type: string
                description: Error message
              status:
                type: integer
                description: HTTP status code
      x-ms-visibility: important
```
## Code

Nun kommen wir zum Code. Claude war mir hier eine Hilfe, da ich nicht der Coder bin. Was macht der Code? Vereinfacht gesagt: er bildet die zusätzlichen Felder, die normalerweise im JSON abgefragt werden, damit ein User diese ausfüllen kann, ohne das JSON erstellen zu müssen. Nachfolgend also einmal die "Magie".

```csharp
using System;
using System.Net;
using System.Net.Http;
using System.Text;
using System.Threading.Tasks;
using System.Web;
using Newtonsoft.Json;
using Newtonsoft.Json.Linq;

public class Script : ScriptBase
{
    public override async Task<HttpResponseMessage> ExecuteAsync()
    {
        try
        {
            // Prüfe welche Operation aufgerufen wird
            if (this.Context.OperationId == "CreatePostSimple")
            {
                return await this.HandleCreatePostSimple().ConfigureAwait(false);
            }

            // Für alle anderen Operations: Standard-Verhalten
            return await this.Context.SendAsync(this.Context.Request, this.CancellationToken).ConfigureAwait(false);
        }
        catch (Exception ex)
        {
            // Fehler abfangen und saubere Response zurückgeben
            var errorResponse = new HttpResponseMessage(HttpStatusCode.InternalServerError)
            {
                Content = new StringContent(
                    JsonConvert.SerializeObject(new { error = "Connector error", message = ex.Message }),
                    Encoding.UTF8,
                    "application/json"
                )
            };
            return errorResponse;
        }
    }

    private async Task<HttpResponseMessage> HandleCreatePostSimple()
    {
        // Query-Parameter auslesen
        var query = HttpUtility.ParseQueryString(this.Context.Request.RequestUri.Query);
        var personId = query.Get("personId");
        var postText = query.Get("postText");
        var visibility = query.Get("visibility") ?? "PUBLIC";
        var lifecycleState = query.Get("lifecycleState") ?? "PUBLISHED";

        // Input-Validierung
        if (string.IsNullOrWhiteSpace(personId))
        {
            return CreateErrorResponse(HttpStatusCode.BadRequest, "Person ID is required");
        }

        if (string.IsNullOrWhiteSpace(postText))
        {
            return CreateErrorResponse(HttpStatusCode.BadRequest, "Post text is required");
        }

        // Person ID Format validieren (nur alphanumerische Zeichen und Bindestriche)
        if (!System.Text.RegularExpressions.Regex.IsMatch(personId, @"^[a-zA-Z0-9\-_]+$"))
        {
            return CreateErrorResponse(HttpStatusCode.BadRequest, "Invalid Person ID format");
        }

        // Post-Text Länge prüfen (LinkedIn Limit: 3000 Zeichen)
        if (postText.Length > 3000)
        {
            return CreateErrorResponse(HttpStatusCode.BadRequest, "Post text exceeds 3000 character limit");
        }

        // Visibility validieren
        if (visibility != "PUBLIC" && visibility != "CONNECTIONS")
        {
            return CreateErrorResponse(HttpStatusCode.BadRequest, "Visibility must be PUBLIC or CONNECTIONS");
        }

        // Lifecycle State validieren
        if (lifecycleState != "PUBLISHED" && lifecycleState != "DRAFT")
        {
            return CreateErrorResponse(HttpStatusCode.BadRequest, "Lifecycle state must be PUBLISHED or DRAFT");
        }

        // Sichere JSON-Struktur erstellen mit JObject (verhindert Injection)
        var requestBody = new JObject
        {
            ["author"] = $"urn:li:person:{personId}",
            ["lifecycleState"] = lifecycleState,
            ["specificContent"] = new JObject
            {
                ["com.linkedin.ugc.ShareContent"] = new JObject
                {
                    ["shareCommentary"] = new JObject
                    {
                        ["text"] = postText // Text wird automatisch escaped
                    },
                    ["shareMediaCategory"] = "NONE"
                }
            },
            ["visibility"] = new JObject
            {
                ["com.linkedin.ugc.MemberNetworkVisibility"] = visibility
            }
        };

        // Zu JSON serialisieren mit sicheren Einstellungen
        var jsonBody = requestBody.ToString(Newtonsoft.Json.Formatting.None);

        // Request URI anpassen (entferne /simple und Query-String)
        var originalUri = this.Context.Request.RequestUri.ToString();
        var baseUri = originalUri.Split('?')[0].Replace("/simple", "");

        try
        {
            this.Context.Request.RequestUri = new Uri(baseUri);
        }
        catch (UriFormatException)
        {
            return CreateErrorResponse(HttpStatusCode.InternalServerError, "Invalid URI format");
        }

        // Content mit UTF-8 Encoding setzen
        this.Context.Request.Content = new StringContent(jsonBody, Encoding.UTF8, "application/json");

        // Request an LinkedIn senden
        HttpResponseMessage response;
        try
        {
            response = await this.Context.SendAsync(this.Context.Request, this.CancellationToken).ConfigureAwait(false);
        }
        catch (TaskCanceledException)
        {
            return CreateErrorResponse(HttpStatusCode.RequestTimeout, "Request timed out");
        }
        catch (HttpRequestException ex)
        {
            return CreateErrorResponse(HttpStatusCode.BadGateway, $"LinkedIn API error: {ex.Message}");
        }

        return response;
    }

    /// <summary>
    /// Hilfsmethode zum Erstellen von Fehler-Responses
    /// </summary>
    private HttpResponseMessage CreateErrorResponse(HttpStatusCode statusCode, string message)
    {
        var errorObject = new JObject
        {
            ["error"] = true,
            ["message"] = message,
            ["status"] = (int)statusCode
        };

        return new HttpResponseMessage(statusCode)
        {
            Content = new StringContent(
                errorObject.ToString(Newtonsoft.Json.Formatting.None),
                Encoding.UTF8,
                "application/json"
            )
        };
    }
}
```

Nun wird der Custom Connector gespeichert - und schon können wir in Power Automate testen.

# Power Automate

Da die Grundlage mit dem Custom Connector nun erstellt wurde, können wir direkt mit Power Automate testen.

Hierzu erstellen wir in der Solution einen neuen Flow.
Der Trigger des Flows kann zum test gern auf "Manually trigger flow" stehen - später ist es zum Beispiel auch denkbar, dass wir täglich prüfen, ob in einer SharePointliste vorbereiteter Content für heute verfügbar ist.

Bevor mit dem eigentlichen Posten starten können, benötigen wir die **Person ID**. Diese erhalten wir über die Aktion **Get User Info** aus unserem Custom Connector.

## Userinformationen abrufen

Um die Aktion hinzuzufügen, klicken wir auf das Plus, wählen dann als Connector "Custom".

![Bereich Custom auswählen, um den neuen Custom Connector zu finden](/zurwerra/images/PowerAutomate_CustomConnector_Auswahl.png "Screenshot Auswahl Custom Connector")

und dann die Aktion **Get User Info**

![Es werden die drei erstellten Aktionen angezeigt. Hier dann die Aktion Get User Info auswählen](/zurwerra/images/PowerAutomate_CustomConnector_Aktionen.png "Screenshot Aktionen Custum Connector")

Wenn wir die Aktion das erste Mal durchführen bzw. noch keine Connection-Informationen vorhanden sind, müssen wir uns einmal anmelden.

![Aufforderung, eine Connection zu erstellen (oder auszuwählen)](/zurwerra/images/PowerAutomate_CustomConnector_Connection.png "Screenshot Connection erstellen")

![LinkedIn Loginfenster, mit dem die Connection erstellt wird](/zurwerra/images/PowerAutomate_LinkedIn_Login.png "Screenshot LinkedIn LoginFenster")

Für die Aktion "Get User Info" sind keine weiteren Einträge notwendig.

## LinkedIn Post erstellen
Jetzt können wir unseren Post erstellen. Wie im Vorfeld schon definiert, haben wir zwei Möglichkeiten dazu: den "normalen" Post und eine vereinfachte Oberfläche.

### Aktion: Create LinkedIn Post
Wenn wir einen Post per JSON hinzufügen wollen, dann können wir die Aktion **Create LinkedIn Post** nutzen. Diese wie gewohnt hinzufügen und das JSON ausfüllen

![Screenshot der CreatePost Aktion](/zurwerra/images/PowerAutomate_CreatePost.png "Screenshot CreatePost")

Als Beispiel hier direkt ein mögliches JSON

```JSON
{
  "author": "urn:li:person:{DEINE_SUB_ID}",
  "lifecycleState": "PUBLISHED",
  "specificContent": {
    "com.linkedin.ugc.ShareContent": {
      "shareCommentary": {
        "text": "Dein Post-Text hier! 🚀"
      },
      "shareMediaCategory": "NONE"
    }
  },
  "visibility": {
    "com.linkedin.ugc.MemberNetworkVisibility": "PUBLIC"
  }
}
```
Anschliessend abschicken - und schon ist dein Post erstellt

### LinkedIn Post einfacher erstellen
Damit es auch für Menschen, die vielleicht kein JSON erstellen wollen, einfacher ist, gibt es die Aktion **Create LinkedIn Post (simple)**

![Screenshot der Aktion CreatePost Simple.](/zurwerra/images/PowerAutomate_CustomConnector_PostSimple.png "Screenshot CreatePost (Simple)")

Zugegeben, es sind viele Optionen, die ausgefüllt werden können und teilweise müssen. Jede davon hat allerdings eine eigene Description, die erklärt, was hier reingehört.

Fangen wir mit der Person ID an. Diese ist Pflicht und verweist auf die Person, die den Post erstellen soll. Die Info zur ID bekommen wir über die Aktion "Get User Info"

![Ausgefüllte Person ID. Information kommt aus der Aktion Get User Info und wird hier nur eingefügt](/zurwerra/images/PowerAutomate_PersonID.png "Screenshot Option Person ID")

Als nächsts müssen wir den eigentlichen Text eingeben. Auch. hierbei handelt es sich logischerweise um ein Pflichtfeld. Und das gute: der Text kann so eingegeben werden, wie er nachher erscheinen soll (Emoji sind also erlaubt)

![Ausgefülltes Textfeld mit einem Text, der genau auf diesen Artikel verweist](/zurwerra/images/PPowerAutomate_LinkedIn-Text.png "Screenshot Option Text")
