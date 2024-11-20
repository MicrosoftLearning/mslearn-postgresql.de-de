---
lab:
  title: Analysieren von Stimmungen
  module: Perform Sentiment Analysis and Opinion Mining using Azure Database for PostgreSQL
---

# Analysieren von Stimmungen

Als Teil der KI-gestützten App, die Sie für Margie's Travel entwickeln, möchten Sie den Benutzenden Informationen über die Stimmung in einzelnen Bewertungen und die allgemeine Stimmung in allen Bewertungen für ein bestimmtes Mietobjekt zur Verfügung stellen. Um dies zu erreichen, verwenden Sie die Erweiterung `azure_ai` in einem flexiblen Azure Database for PostgreSQL-Server, um die Stimmungsanalyse-Funktionalität in Ihre Datenbank zu integrieren.

## Vor der Installation

Hierfür benötigen Sie ein [Azure-Abonnement](https://azure.microsoft.com/free) mit Administratorrechten.

### Bereitstellen von Ressourcen in Ihrem Azure-Abonnement

Dieser Schritt führt Sie durch die Verwendung von Azure CLI-Befehlen aus der Azure Cloud Shell, um eine Ressourcengruppe zu erstellen und ein Bicep-Skript auszuführen, um die Azure-Dienste bereitzustellen, die zum Abschließen dieser Übung in Ihrem Azure-Abonnement erforderlich sind.

1. Öffnen Sie einen Webbrowser, und navigieren Sie zum [Azure-Portal](https://portal.azure.com/).

2. Wählen Sie das Symbol **Cloud Shell** in der Symbolleiste des Azure-Portals aus, um einen neuen [Cloud Shell](https://learn.microsoft.com/azure/cloud-shell/overview)-Bereich am unteren Rand Ihres Browserfensters zu öffnen.

    ![Screenshot der Azure-Symbolleiste mit dem Cloud Shell-Symbol, das durch einen roten Rahmen hervorgehoben ist.](media/11-portal-toolbar-cloud-shell.png)

    Wählen Sie bei Aufforderung die erforderlichen Optionen aus, um eine *Bash*-Shell zu öffnen. Wenn Sie zuvor eine *PowerShell*-Konsole verwendet haben, wechseln Sie zu einer *Bash*-Shell.

3. Geben Sie an der Cloud Shell-Eingabeaufforderung Folgendes ein, um das GitHub-Repository mit den Übungsressourcen zu klonen:

    ```bash
    git clone https://github.com/MicrosoftLearning/mslearn-postgresql.git
    ```

4. Als Nächstes führen Sie drei Befehle aus, um Variablen zu definieren und so die redundante Eingabe zu reduzieren, wenn Sie Azure-CLI-Befehle zum Erstellen von Azure-Ressourcen verwenden. Die Variablen stehen für den Namen, den Sie Ihrer Ressourcengruppe zuweisen (`RG_NAME`), für die Azure-Region (`REGION`), in der die Ressourcen bereitgestellt werden, und für ein zufällig generiertes Kennwort für den PostgreSQL-Admin-Login (`ADMIN_PASSWORD`).

    Im ersten Befehl ist die der entsprechenden Variablen zugewiesene Region `eastus`, aber Sie können sie auch durch einen Ort Ihrer Voreinstellung ersetzen. Wenn Sie jedoch die Standardeinstellung ersetzen, müssen Sie eine andere [Azure-Region auswählen, die die abstrakte Zusammenfassung unterstützt](https://learn.microsoft.com/azure/ai-services/language-service/summarization/region-support), um sicherzustellen, dass Sie alle Aufgaben in den Modulen in diesem Lernpfad erledigen können.

    ```bash
    REGION=eastus
    ```

    Mit dem folgenden Befehl weisen Sie den Namen für die Ressourcengruppe zu, die alle in dieser Übung verwendeten Ressourcen enthalten wird. Der Name der Ressourcengruppe, die der entsprechenden Variablen zugewiesen ist, lautet `rg-learn-postgresql-ai-$REGION`, wobei `$REGION` der Speicherort ist, den Sie oben angegeben haben. Sie können ihn jedoch in einen beliebigen anderen Ressourcengruppennamen ändern, der Ihren Vorstellungen entspricht.

    ```bash
    RG_NAME=rg-learn-postgresql-ai-$REGION
    ```

    Der letzte Befehl generiert nach dem Zufallsprinzip ein Kennwort für das PostgreSQL-Admin-Login. **Kopieren Sie sie** an einen sicheren Ort, um sie später für die Verbindung mit ihrem flexiblen PostgreSQL-Server zu verwenden.

    ```bash
    a=()
    for i in {a..z} {A..Z} {0..9}; 
        do
        a[$RANDOM]=$i
        done
    ADMIN_PASSWORD=$(IFS=; echo "${a[*]::18}")
    echo "Your randomly generated PostgreSQL admin user's password is:"
    echo $ADMIN_PASSWORD
    ```

5. Wenn Sie Zugriff auf mehr als ein Azure-Abonnement haben und Ihr Standardabonnement nicht das Abonnement ist, in dem Sie die Ressourcengruppe und andere Ressourcen für diese Übung erstellen möchten, führen Sie diesen Befehl aus, um das entsprechende Abonnement festzulegen, und ersetzen Sie dabei das Token `<subscriptionName|subscriptionId>` durch den Namen oder die ID des Abonnements, das Sie verwenden möchten:

    ```azurecli
    az account set --subscription <subscriptionName|subscriptionId>
    ```

6. Führen Sie den folgenden Azure CLI-Befehl aus, um Ihre Ressourcengruppe zu erstellen:

    ```azurecli
    az group create --name $RG_NAME --location $REGION
    ```

7. Verwenden Sie schließlich die Azure CLI, um ein Bicep-Bereitstellungsskript auszuführen, um Azure-Ressourcen in Ihrer Ressourcengruppe bereitzustellen:

    ```azurecli
    az deployment group create --resource-group $RG_NAME --template-file "mslearn-postgresql/Allfiles/Labs/Shared/deploy.bicep" --parameters restore=false adminLogin=pgAdmin adminLoginPassword=$ADMIN_PASSWORD
    ```

    Das Bicep-Bereitstellungsskript stellt die Azure-Dienste bereit, die zum Abschließen dieser Übung in Ihrer Ressourcengruppe erforderlich sind. Die bereitgestellten Ressourcen umfassen einen flexiblen Azure Database for PostgreSQL-Server, Azure OpenAI und ein Azure KI Language-Dienst. Das Bicep-Skript führt auch einige Konfigurationsschritte aus, wie z. B. das Hinzufügen der Erweiterungen `azure_ai` und `vector` zur _Positivliste_ des PostgreSQL-Servers (über den Serverparameter azure.extensions), das Erstellen einer Datenbank mit dem Namen `rentals` auf dem Server und das Hinzufügen einer Bereitstellung mit dem Namen `embedding` unter Verwendung des Modells `text-embedding-ada-002` zu Ihrem Azure OpenAI-Service. Beachten Sie, dass die Bicep-Datei von allen Modulen in diesem Lernpfad gemeinsam genutzt wird, sodass Sie möglicherweise nur einige der bereitgestellten Ressourcen in einigen Übungen verwenden.

    Die Bereitstellung dauert in der Regel mehrere Minuten. Sie können es von der Cloud Shell aus überwachen oder zur Seite **Bereitstellungen** für die oben erstellte Ressourcengruppe navigieren und dort den Bereitstellungsfortschritt beobachten.

8. Schließen Sie den Cloud Shell-Bereich, sobald die Ressourcenbereitstellung abgeschlossen ist.

### Problembehandlung bei der Bereitstellung

Beim Ausführen des Bicep-Bereitstellungsskripts treten möglicherweise einige Fehler auf. Die häufigsten Meldungen und die Schritte zu ihrer Behebung sind:

- Wenn Sie zuvor das Bicep-Bereitstellungsskript für diesen Lernpfad ausgeführt und anschließend die Ressourcen gelöscht haben, erhalten Sie möglicherweise eine Fehlermeldung wie die folgende, wenn Sie versuchen, das Skript innerhalb von 48 Stunden nach dem Löschen der Ressourcen erneut auszuführen:

    ```bash
    {"code": "InvalidTemplateDeployment", "message": "The template deployment 'deploy' is not valid according to the validation procedure. The tracking id is '4e87a33d-a0ac-4aec-88d8-177b04c1d752'. See inner errors for details."}
    
    Inner Errors:
    {"code": "FlagMustBeSetForRestore", "message": "An existing resource with ID '/subscriptions/{subscriptionId}/resourceGroups/rg-learn-postgresql-ai-eastus/providers/Microsoft.CognitiveServices/accounts/{accountName}' has been soft-deleted. To restore the resource, you must specify 'restore' to be 'true' in the property. If you don't want to restore existing resource, please purge it first."}
    ```

    Wenn Sie diese Meldung erhalten, ändern Sie den obigen Befehl `azure deployment group create`, um den Parameter `restore` auf `true` zu setzen, und führen Sie ihn erneut aus.

- Wenn die Bereitstellung bestimmter Ressourcen in der ausgewählten Region eingeschränkt ist, müssen Sie die Variable `REGION` auf einen anderen Speicherort festlegen und die Befehle erneut ausführen, um die Ressourcengruppe zu erstellen und das Bicep-Bereitstellungsskript auszuführen.

    ```bash
    {"status":"Failed","error":{"code":"DeploymentFailed","target":"/subscriptions/{subscriptionId}/resourceGroups/{resourceGrouName}/providers/Microsoft.Resources/deployments/{deploymentName}","message":"At least one resource deployment operation failed. Please list deployment operations for details. Please see https://aka.ms/arm-deployment-operations for usage details.","details":[{"code":"ResourceDeploymentFailure","target":"/subscriptions/{subscriptionId}/resourceGroups/{resourceGrouName}/providers/Microsoft.DBforPostgreSQL/flexibleServers/{serverName}","message":"The resource write operation failed to complete successfully, because it reached terminal provisioning state 'Failed'.","details":[{"code":"RegionIsOfferRestricted","message":"Subscriptions are restricted from provisioning in this region. Please choose a different region. For exceptions to this rule please open a support request with Issue type of 'Service and subscription limits'. See https://review.learn.microsoft.com/en-us/azure/postgresql/flexible-server/how-to-request-quota-increase for more details."}]}]}}
    ```

- Wenn das Skript aufgrund der Anforderung, die Vereinbarung über die verantwortungsvolle KI zu akzeptieren, keine KI-Ressource erstellen kann, kann der folgende Fehler auftreten. Verwenden Sie in diesem Fall die Benutzeroberfläche des Azure-Portals, um eine Azure KI Services-Ressource zu erstellen, und führen Sie das Bereitstellungsskript dann erneut aus.

    ```bash
    {"code": "InvalidTemplateDeployment", "message": "The template deployment 'deploy' is not valid according to the validation procedure. The tracking id is 'f8412edb-6386-4192-a22f-43557a51ea5f'. See inner errors for details."}
     
    Inner Errors:
    {"code": "ResourceKindRequireAcceptTerms", "message": "This subscription cannot create TextAnalytics until you agree to Responsible AI terms for this resource. You can agree to Responsible AI terms by creating a resource through the Azure Portal then trying again. For more detail go to https://go.microsoft.com/fwlink/?linkid=2164190"}
    ```

## Herstellen einer Verbindung mit der Datenbank mithilfe von psql in Azure Cloud Shell

Bei dieser Aufgabe stellen Sie über das [psql-Befehlszeilenprogramm](https://www.postgresql.org/docs/current/app-psql.html) von der [Azure Cloud Shell](https://learn.microsoft.com/azure/cloud-shell/overview) aus eine Verbindung zur `rentals`-Datenbank auf Ihrem Azure Database for PostgreSQL-Server her.

1. Navigieren Sie im [Azure-Portal](https://portal.azure.com/) zu Ihrer neu erstellten flexiblen Azure Database for PostgreSQL-Serverinstanz.

2. Wählen Sie im Ressourcenmenü unter **Einstellungen** die Option **Datenbanken** und wählen Sie **Verbinden** für die Datenbank `rentals`.

    ![Screenshot der Azure Database for PostgreSQL-Datenbankseite. Datenbanken und Verbinden für die Mietdatenbank werden in roten Feldern hervorgehoben.](media/17-postgresql-rentals-database-connect.png)

3. Geben Sie beim Prompt „Kennwort für den Benutzer pgAdmin“ in der Cloud Shell das zufällig generierte Kennwort für die **pgAdmin**-Anmeldung ein.

    Sobald Sie angemeldet sind, wird der `psql`-Prompt für die `rentals`-Datenbank angezeigt.

## Füllen der Datenbank mit Beispieldaten

Bevor Sie die Stimmung von Mietobjektüberprüfungen mithilfe der `azure_ai`-Erweiterung analysieren können, müssen Sie Ihrer Datenbank Beispieldaten hinzufügen. Fügen Sie der `rentals`-Datenbank eine Tabelle hinzu, und füllen Sie sie mit Kundenrezensionen auf, sodass Sie Daten haben, zu denen eine Stimmungsanalyse durchgeführt werden soll.

1. Führen Sie den folgenden Befehl aus, um eine Tabelle mit dem Namen `reviews` für die Speicherung der von der Kundschaft eingereichten Immobilienbewertungen zu erstellen:

    ```sql
    DROP TABLE IF EXISTS reviews;

    CREATE TABLE reviews (
        id int,
        listing_id int, 
        date date,
        comments text
    );
    ```

2. Verwenden Sie als Nächstes den Befehl `COPY`, um die Tabelle mit Daten aus einer CSV-Datei zu füllen. Führen Sie den folgenden Befehl aus, um Kundenrezensionen in die `reviews`-Tabelle zu laden:

    ```sql
    \COPY reviews FROM 'mslearn-postgresql/Allfiles/Labs/Shared/reviews.csv' CSV HEADER
    ```

    Die Befehlsausgabe sollte `COPY 354` lauten, was bedeutet, dass 354 Zeilen aus der CSV-Datei in die Tabelle geschrieben wurden.

## Installieren und Konfigurieren der `azure_ai`-Erweiterung

Bevor Sie die `azure_ai`-Erweiterung verwenden, müssen Sie sie in Ihrer Datenbank installieren und konfigurieren, um eine Verbindung mit Ihren Azure KI Services-Ressourcen herzustellen. Die `azure_ai`-Erweiterung ermöglicht es Ihnen, die Dienste Azure OpenAI und Azure KI Language in Ihre Datenbank zu integrieren. Um die Erweiterung in Ihrer Datenbank zu aktivieren, führen Sie die folgenden Schritte aus:

1. Führen Sie den folgenden Befehl beim `psql`-Prompt aus, um zu überprüfen, ob die Erweiterungen `azure_ai` und `vector` erfolgreich zur _Positivliste_ Ihres Servers hinzugefügt wurden, und zwar durch das Bicep-Bereitstellungsskript, das Sie ausgeführt haben, als Sie Ihre Umgebung einrichteten:

    ```sql
    SHOW azure.extensions;
    ```

    Der Befehl zeigt die Liste der Erweiterungen in der _Positivliste_ des Servers an. Wenn alles richtig installiert wurde, muss Ihre Ausgabe `azure_ai` und `vector` enthalten, wie hier dargestellt:

    ```sql
     azure.extensions 
    ------------------
     azure_ai,vector
    ```

    Bevor eine Erweiterung in einer flexiblen Serverdatenbank von Azure Database for PostgreSQL installiert und verwendet werden kann, muss sie zur _Positivliste_ des Servers hinzugefügt werden, wie in [Verwendung von PostgreSQL-Erweiterungen](https://learn.microsoft.com/azure/postgresql/flexible-server/concepts-extensions#how-to-use-postgresql-extensions) beschrieben.

2. Jetzt können Sie die `azure_ai`-Erweiterung mit dem Befehl [CREATE EXTENSION](https://www.postgresql.org/docs/current/sql-createextension.html) installieren.

    ```sql
    CREATE EXTENSION IF NOT EXISTS azure_ai;
    ```

    `CREATE EXTENSION` lädt eine neue Erweiterung in die Datenbank, indem die Skriptdatei ausgeführt wird. Dieses Skript erstellt in der Regel neue SQL-Objekte wie Funktionen, Datentypen und Schemata. Wenn bereits eine Erweiterung desselben Namens vorhanden ist, wird ein Fehler ausgelöst. Durch Hinzufügen von `IF NOT EXISTS` kann der Befehl ausgeführt werden, ohne dass eine Fehlermeldung ausgegeben wird, wenn er bereits installiert ist.

## Verbinden Ihres Azure KI Services-Kontos

Die im `azure_cognitive` Schema der `azure_ai` Erweiterung enthaltenen Azure KI Services-Integrationen bieten eine umfangreiche Sammlung von KI-Sprachfeatures, die direkt über die Datenbank zugänglich sind. Die Funktionen für die Stimmungsanalyse werden über den [Azure KI Language-Service](https://learn.microsoft.com/azure/ai-services/language-service/overview) aktiviert.

1. Um erfolgreich Aufrufe über Ihre Azure AI Language-Dienste mit der `azure_ai`-Erweiterung zu tätigen, müssen Sie deren Endpunkt und Schlüssel für die Erweiterung bereitstellen. Navigieren Sie im [Azure-Portal](https://portal.azure.com/) auf derselben Browser-Registerkarte, auf der Cloud Shell geöffnet ist, zu Ihrer Sprachdienstressource und wählen Sie im linken Navigationsmenü unter **Ressourcenverwaltung** das Element **Schlüssel und Endpunkt** aus.

    ![Es wird ein Screenshot der Seite „Schlüssel und Endpunkte“ des Azure-Sprachdienstes angezeigt, wobei die Schaltflächen „Schlüssel 1“ und „Endpunkt kopieren“ durch rote Kästchen hervorgehoben sind.](media/16-azure-language-service-keys-endpoints.png)

> [!Note]
>
> Wenn Sie bei der Installation der Erweiterung `azure_ai` die Meldung `NOTICE: extension "azure_ai" already exists, skipping CREATE EXTENSION` erhalten haben und die Erweiterung bereits mit Ihrem Sprachdienst-Endpunkt und -Schlüssel konfiguriert ist, können Sie die Funktion `azure_ai.get_setting()` verwenden, um zu überprüfen, ob diese Einstellungen korrekt sind und dann Schritt 2 überspringen, wenn dies der Fall ist.

2. Kopieren Sie Ihre Endpunkt- und Zugriffsschlüsselwerte und ersetzen Sie dann in den folgenden Befehlen die Token `{endpoint}` und `{api-key}` durch die Werte, die Sie aus dem Azure-Portal kopiert haben. Führen Sie die Befehle von der Eingabeaufforderung `psql` in der Cloud Shell aus, um Ihre Werte der `azure_ai.settings`-Tabelle hinzuzufügen.

    ```sql
    SELECT azure_ai.set_setting('azure_cognitive.endpoint', '{endpoint}');
    ```

    ```sql
    SELECT azure_ai.set_setting('azure_cognitive.subscription_key', '{api-key}');
    ```

## Überprüfen der Stimmungsanalyse-Funktionen der Erweiterung

In dieser Aufgabe verwenden Sie die `azure_cognitive.analyze_sentiment()`-Funktion, um Bewertungen von Mietimmobilienlisten auszuwerten.

1. Für den Rest dieser Übung arbeiten Sie ausschließlich in der Cloud Shell. Daher kann es hilfreich sein, den Bereich in Ihrem Browserfenster zu erweitern, indem Sie auf die Schaltfläche zum **Maximieren** oben rechts im Cloud Shell-Bereich klicken.

    ![Screenshot des Azure Cloud Shell-Fensters mit der Schaltfläche „Maximieren“, die durch eine rote Box hervorgehoben ist.](media/16-azure-cloud-shell-pane-maximize.png)

2. Bei der Arbeit mit `psql` in der Cloud Shell kann es hilfreich sein, die erweiterte Anzeige für Abfrageergebnisse zu aktivieren, da dies die Lesbarkeit der Ausgabe für nachfolgende Befehle verbessert. Führen Sie den folgenden Befehl aus, damit die erweiterte Anzeige automatisch angewendet werden kann.

    ```sql
    \x auto
    ```

3. Die Funktionen zur Stimmungsanalyse der `azure_ai`-Erweiterung befinden sich im `azure_cognitive`-Schema. Verwenden der `analyze_sentiment()`-Funktion. Verwenden Sie den [`\df`-Metabefehl](https://www.postgresql.org/docs/current/app-psql.html#APP-PSQL-META-COMMAND-DF-LC), um die Funktion zu untersuchen, indem Sie Folgendes ausführen:

    ```sql
    \df azure_cognitive.analyze_sentiment
    ```

    Die Metabefehlsausgabe zeigt das Schema, den Namen, den Ergebnisdatentyp und die Argumente der Funktion an. Diese Informationen helfen Ihnen zu verstehen, wie Sie mit der Funktion aus Ihren Abfragen interagieren.

    Die Ausgabe zeigt drei Überladungen der `analyze_sentiment()`-Funktion, sodass Sie deren Unterschiede überprüfen können. Die `Argument data types`-Eigenschaft in der Ausgabe zeigt die Liste der Argumente, die die drei Funktionsüberladungen erwarten:

    | Argument | Typ | Standard | Beschreibung |
    | -------- | ---- | ------- | ----------- |
    | Text | `text` oder `text[]` || Die Texte, für die die Stimmung analysiert werden soll. |
    | language_text | `text` oder `text[]` || Sprachcode (oder eine Reihe von Sprachcodes), der die Sprache des Textes angibt, der auf seine Stimmung hin analysiert werden soll. Überprüfen Sie die [Liste der unterstützten Sprachen](https://learn.microsoft.com/azure/ai-services/language-service/sentiment-opinion-mining/language-support), um die erforderlichen Sprachcodes abzurufen. |
    | batch_size | `integer` | 10 | Nur für die beiden Überladungen, die eine Eingabe von `text[]` erwarten. Gibt die Anzahl der gleichzeitig zu verarbeitenden Datensätze an. |
    | disable_service_logs | `boolean` | false | Flag, das angibt, ob Dienstprotokolle deaktiviert werden sollen. |
    | timeout_ms | `integer` | NULL | Timeout in Millisekunden, nach dem der Vorgang beendet wird. |
    | throw_on_error | `boolean` | true | Flag, das angibt, ob die Funktion beim Fehler eine Ausnahme auslösen soll, was zu einem Rollback der Umbruchtransaktionen führt. |
    | max_attempts | `integer` | 1 | Anzahl der Wiederholungen des Aufrufs an Azure KI Services im Falle eines Fehlers. |
    | retry_delay_ms | `integer` | 1.000 | Die Zeit (in Millisekunden), die gewartet werden muss, bevor versucht wird, den Azure KI Services-Endpunkt erneut aufzurufen. |

4. Es ist auch zwingend erforderlich, die Struktur des Datentyps zu verstehen, den eine Funktion zurückgibt, damit Sie die Ausgabe in Ihren Abfragen ordnungsgemäß verarbeiten können. Führen Sie den folgenden Befehl aus, um den `sentiment_analysis_result`-Typ zu überprüfen:

    ```sql
    \dT+ azure_cognitive.sentiment_analysis_result
    ```

5. Die Ausgabe des obigen Befehls zeigt, dass der `sentiment_analysis_result`-Typ ein `tuple` ist. Sie können die Struktur des `tuple`-Vorgangs weiter untersuchen, indem Sie den folgenden Befehl ausführen, um die Spalten im `sentiment_analysis_result`-Typ anzuzeigen:

    ```sql
    \d+ azure_cognitive.sentiment_analysis_result
    ```

    Die Ausgabe des Befehls sollte in etwa wie folgt aussehen:

    ```sql
                     Composite type "azure_cognitive.sentiment_analysis_result"
         Column     |     Type         | Collation | Nullable | Default | Storage  | Description 
    ----------------+------------------+-----------+----------+---------+----------+-------------
     sentiment      | text             |           |          |         | extended | 
     positive_score | double precision |           |          |         | plain    | 
     neutral_score  | double precision |           |          |         | plain    | 
     negative_score | double precision |           |          |         | plain    |
    ```

    Der `azure_cognitive.sentiment_analysis_result` ist ein zusammengesetzter Typ, der die Stimmungsvorhersagen des eingegebenen Textes enthält. Er enthält die Stimmung, die positiv, negativ, neutral oder gemischt sein kann, und die Bewertungen für positive, neutrale und negative Aspekte im Text. Die Ergebnisse werden als reale Zahlen zwischen 0 und 1 dargestellt. Beispielsweise ist die Stimmung in (neutral, 0,26, 0,64, 0,09) neutral, mit einem positiven Ergebnis von 0,26, neutral von 0,64 und negativ bei 0,09.

## Analysieren der Stimmung von Bewertungen

1. Nachdem Sie die `analyze_sentiment()`-Funktion und die `sentiment_analysis_result`-Rückgabe überprüft haben, setzen wir die zu verwendende Funktion ein. Führen Sie die folgende einfache Abfrage aus, die eine Stimmungsanalyse für eine Handvoll Kommentare in der `reviews`-Tabelle durchführt:

    ```sql
    SELECT
        id,
        azure_cognitive.analyze_sentiment(comments, 'en') AS sentiment
    FROM reviews
    WHERE id <= 10
    ORDER BY id;
    ```

    Beachten Sie in den beiden analysierten Datensätzen die `sentiment`-Werte in der Ausgabe, `(mixed,0.71,0.09,0.2)` und `(positive,0.99,0.01,0)`. Diese stellen die `sentiment_analysis_result` von der `analyze_sentiment()`-Funktion in der obigen Abfrage zurückgegebenen Werte dar. Die Analyse wurde über das `comments`-Feld in der `reviews`-Tabelle durchgeführt.

    > [!Note]
    >
    > Mit der `analyze_sentiment()`-Inline-Funktion können Sie die Stimmung des Textes in Ihren Abfragen schnell analysieren. Dies eignet sich zwar gut für eine kleine Anzahl von Datensätzen, aber es ist möglicherweise nicht ideal, die Stimmung einer großen Anzahl von Datensätzen zu analysieren oder alle Datensätze in einer Tabelle zu aktualisieren, die zehntausende Rezensionen oder mehr enthalten kann.

2. Ein weiterer Ansatz, der für längere Rezensionen nützlich sein kann, besteht darin, die Stimmung der einzelnen Sätze darin zu analysieren. Verwenden Sie dazu die Überladung der `analyze_sentiment()`-Funktion, die ein Text-Array akzeptiert.

    ```sql
    SELECT
        azure_cognitive.analyze_sentiment(ARRAY_REMOVE(STRING_TO_ARRAY(comments, '.'), ''), 'en') AS sentence_sentiments
    FROM reviews
    WHERE id = 1;
    ```

    In der obigen Abfrage haben Sie die Funktion `STRING_TO_ARRAY` von PostgreSQL verwendet. Darüber hinaus wurde die `ARRAY_REMOVE`-Funktion verwendet, um alle Arrayelemente zu entfernen, die leere Zeichenfolgen sind, da diese Fehler mit der `analyze_sentiment()`-Funktion verursachen.

    Die Ausgabe aus der Abfrage ermöglicht es Ihnen, ein besseres Verständnis der Stimmung zu erhalten, die `mixed` der Gesamtüberprüfung zugewiesen ist. Die Sätze sind eine Mischung aus positiven, neutralen und negativen Stimmungen.

3. Die beiden vorherigen Abfragen haben die `sentiment_analysis_result` direkt aus der Abfrage zurückgegeben. Allerdings werden Sie wahrscheinlich die zugrunde liegenden Werte innerhalb der `sentiment_analysis_result``tuple` abrufen wollen. Führen Sie die folgende Abfrage aus, die nach überwältigend positiven Rezensionen sucht, und extrahiert die Stimmungskomponenten in einzelne Felder:

    ```sql
    WITH cte AS (
        SELECT id, comments, azure_cognitive.analyze_sentiment(comments, 'en') AS sentiment FROM reviews
    )
    SELECT
        id,
        (sentiment).sentiment,
        (sentiment).positive_score,
        (sentiment).neutral_score,
        (sentiment).negative_score,
        comments
    FROM cte
    WHERE (sentiment).positive_score > 0.98
    LIMIT 5;
    ```

    Die obige Abfrage verwendet einen allgemeinen Tabellenausdruck oder CTE, um die Stimmungsbewertungen für alle Datensätze in der `reviews`-Tabelle abzurufen. Anschließend wählt es die `sentiment` zusammengesetzten Spalten aus den `sentiment_analysis_result` vom CTE zurückgegebenen Spalten aus, um die einzelnen Werte aus `tuple.` zu extrahieren.

## Speichern der Stimmung in der Bewertungstabelle

Für das Empfehlungssystem für Mietobjekte, das Sie für Margie's Travel erstellen, möchten Sie Stimmungsbewertungen in der Datenbank speichern, damit Sie nicht jedes Mal, wenn Stimmungsbewertungen angefordert werden, Anrufe tätigen und Kosten verursachen müssen. Das Ausführen von Stimmungsanalysen kann hervorragend für kleine Datensätze oder die Analyse von Daten in nahezu Echtzeit sein. Dennoch ist das Hinzufügen der Stimmungsdaten zur Verwendung in Ihrer Anwendung für Ihre gespeicherten Rezensionen sinnvoll. Dazu möchten Sie die `reviews`-Tabelle ändern, um Spalten zum Speichern der Stimmungsbewertung und der positiven, neutralen und negativen Bewertungen hinzuzufügen.

1. Führen Sie die folgende Abfrage aus, um die `reviews`-Tabelle zu aktualisieren, damit sie Stimmungsdetails speichern kann:

    ```sql
    ALTER TABLE reviews
    ADD COLUMN sentiment varchar(10),
    ADD COLUMN positive_score numeric,
    ADD COLUMN neutral_score numeric,
    ADD COLUMN negative_score numeric;
    ```

2. Als Nächstes möchten Sie die vorhandenen Datensätze in der `reviews`-Tabelle mit ihrem Stimmungswert und den zugehörigen Punktzahlen aktualisieren.

    ```sql
    WITH cte AS (
        SELECT id, azure_cognitive.analyze_sentiment(comments, 'en') AS sentiment FROM reviews
    )
    UPDATE reviews AS r
    SET
        sentiment = (cte.sentiment).sentiment,
        positive_score = (cte.sentiment).positive_score,
        neutral_score = (cte.sentiment).neutral_score,
        negative_score = (cte.sentiment).negative_score
    FROM cte
    WHERE r.id = cte.id;
    ```

    Das Ausführen dieser Abfrage dauert eine lange Zeit, da die Kommentare für jede Überprüfung in der Tabelle zur Analyse einzeln an den Endpunkt des Sprachdiensts gesendet werden. Das Senden von Datensätzen in Batches ist beim Umgang mit vielen Datensätzen effizienter.

3. Starten wir mit der folgenden Abfrage die gleiche Aktualisierungsaktion, senden diesmal jedoch Kommentare aus der Tabelle `reviews` in Stapeln von 10 (dies ist die maximal zulässige Stapelgröße) und bewerten die Leistungsunterschiede.

    ```sql
    WITH cte AS (
        SELECT azure_cognitive.analyze_sentiment(ARRAY(SELECT comments FROM reviews ORDER BY id), 'en', batch_size => 10) as sentiments
    ),
    sentiment_cte AS (
        SELECT
            ROW_NUMBER() OVER () AS id,
            sentiments AS sentiment
        FROM cte
    )
    UPDATE reviews AS r
    SET
        sentiment = (sentiment_cte.sentiment).sentiment,
        positive_score = (sentiment_cte.sentiment).positive_score,
        neutral_score = (sentiment_cte.sentiment).neutral_score,
        negative_score = (sentiment_cte.sentiment).negative_score
    FROM sentiment_cte
    WHERE r.id = sentiment_cte.id;
    ```

    Obwohl diese Abfrage etwas komplexer ist, ist die Verwendung von zwei CTEs viel leistungsfähiger. In dieser Abfrage analysiert der erste CTE die Stimmung von Stapeln von Überprüfungskommentaren und der zweite extrahiert die resultierende Tabelle von `sentiment_analysis_results` in eine neue Tabelle, die eine `id` basierend auf der Ordnungszahl und „sentiment_analysis_result“ für jede Zeile enthält. Der zweite CTE kann dann in der Update-Anweisung verwendet werden, um die Werte in die Datenbank zu schreiben.

4. Führen Sie als Nächstes eine Abfrage durch, um die Aktualisierungsergebnisse zu beobachten, und suchen Sie nach Bewertungen mit einer **negativen** Stimmung, beginnend mit der negativsten zuerst.

    ```sql
    SELECT
        id,
        negative_score,
        comments
    FROM reviews
    WHERE sentiment = 'negative'
    ORDER BY negative_score DESC;
    ```

## Bereinigung

Nachdem Sie diese Übung abgeschlossen haben, löschen Sie die Azure-Ressourcen, die Sie erstellt haben. Die Abrechnung basiert auf der konfigurierten Kapazität und nicht auf der Verwendung der Datenbank. Befolgen Sie diese Anweisungen, um Ihre Ressourcengruppe und alle Ressourcen zu löschen, die Sie für diese Übung erstellt haben.

1. Öffnen Sie einen Webbrowser und navigieren Sie zum [Azure-Portal](https://portal.azure.com/). Wählen Sie auf der Startseite **Ressourcengruppen** unter Azure-Dienste aus.

    ![Screenshot von Ressourcengruppen, die durch ein rotes Feld unter Azure-Diensten im Azure-Portal hervorgehoben sind.](media/16-azure-portal-home-azure-services-resource-groups.png)

2. Geben Sie in das Suchfeld für ein beliebiges Feld den Namen der Ressourcengruppe ein, die Sie für dieses Lab erstellt haben, und wählen Sie dann Ihre Ressourcengruppe aus der Liste aus.

3. Wählen Sie auf der Seite **Übersicht** der Ressourcengruppe die Option **Ressourcengruppe löschen** aus.

    ![Screenshot des Übersichtsblatts der Ressourcengruppe mit der Schaltfläche „Ressourcengruppe löschen“, die durch ein rotes Kästchen hervorgehoben ist.](media/16-resource-group-delete.png)

4. Geben Sie im Bestätigungsdialogfeld den Namen der Ressourcengruppe ein, die Sie löschen möchten, und wählen Sie dann **Löschen** aus.
