---
lab:
  title: Analysieren von Stimmungen
  module: Perform Sentiment Analysis and Opinion Mining using Azure Database for PostgreSQL
---

# Analysieren von Stimmungen

Als Teil der KI-gesteuerten App, die Sie für Margie's Travel entwickeln, möchten Sie den Nutzerinnen und Nutzern Informationen über die Stimmung einzelner Bewertungen und die Gesamtstimmung aller Bewertungen für ein bestimmtes Mietobjekt zur Verfügung stellen. Verwenden Sie dazu die `azure_ai`-Erweiterung in einem Azure Database for PostgreSQL – Flexibler Server, um Sentiment-Analysefunktionen in Ihre Datenbank zu integrieren.

## Vor der Installation

Sie benötigen ein [Azure-Abonnement](https://azure.microsoft.com/free) mit administrativen Rechten und müssen für den Azure OpenAI-Zugang in diesem Abonnement zugelassen sein. Wenn Sie Zugriff auf Azure OpenAI benötigen, bewerben Sie sich auf der Seite [Eingeschränkter Zugriff auf Azure OpenAI](https://learn.microsoft.com/legal/cognitive-services/openai/limited-access).

### Bereitstellen von Ressourcen in Ihrem Azure-Abonnement

Dieser Schritt führt Sie durch die Verwendung von Azure CLI-Befehlen aus der Azure Cloud Shell, um eine Ressourcengruppe zu erstellen und ein Bicep-Skript auszuführen, um die für diese Übung erforderlichen Azure-Services in Ihrem Azure-Abonnement bereitzustellen.

1. Öffnen Sie einen Webbrowser, und navigieren Sie zum [Azure-Portal](https://portal.azure.com/).

2. Wählen Sie das Symbol **Cloud Shell** in der Symbolleiste des Azure-Portals aus, um einen neuen [Cloud Shell](https://learn.microsoft.com/azure/cloud-shell/overview)-Bereich am unteren Rand Ihres Browserfensters zu öffnen.

    ![Screenshot der Azure-Symbolleiste mit dem Cloud Shell-Symbol, das durch eine rote Box hervorgehoben ist.](media/11-portal-toolbar-cloud-shell.png)

    Wählen Sie bei Aufforderung die erforderlichen Optionen aus, um eine *Bash*-Shell zu öffnen. Wenn Sie zuvor eine *PowerShell*-Konsole verwendet haben, wechseln Sie zu einer *Bash*-Shell.

3. Geben Sie an der Cloud Shell-Eingabeaufforderung Folgendes ein, um das GitHub-Repository mit den Übungsressourcen zu klonen:

    ```bash
    git clone https://github.com/MicrosoftLearning/mslearn-postgresql.git
    ```

4. Als Nächstes führen Sie drei Befehle aus, um Variablen zu definieren und so die redundante Eingabe zu reduzieren, wenn Sie Azure-CLI-Befehle zum Erstellen von Azure-Ressourcen verwenden. Die Variablen stehen für den Namen, den Sie Ihrer Ressourcengruppe zuweisen (`RG_NAME`), für die Azure-Region (`REGION`), in der die Ressourcen bereitgestellt werden und für ein zufällig generiertes Kennwort für den PostgreSQL-Administrator-Login (`ADMIN_PASSWORD`).

    Im ersten Befehl ist die Region, die der entsprechenden Variablen zugewiesen ist, `eastus`, aber Sie können sie auch durch einen Ort Ihrer Wahl ersetzen. Wenn Sie jedoch die Standardeinstellung ersetzen, müssen Sie eine andere [Azure-Region auswählen, die die abstrakte Zusammenfassung unterstützt](https://learn.microsoft.com/azure/ai-services/language-service/summarization/region-support), um sicherzustellen, dass Sie alle Aufgaben in den Modulen in diesem Lernpfad erledigen können.

    ```bash
    REGION=eastus
    ```

    Mit dem folgenden Befehl weisen Sie den Namen für die Ressourcengruppe zu, die alle in dieser Übung verwendeten Ressourcen enthalten wird. Der Name der Ressourcengruppe, der der entsprechenden Variablen zugewiesen ist, lautet `rg-learn-postgresql-ai-$REGION`, wobei `$REGION` der Ort ist, den Sie oben angegeben haben. Sie können den Namen jedoch auch in einen anderen Namen für die Ressourcengruppe ändern, der Ihren Wünschen entspricht.

    ```bash
    RG_NAME=rg-learn-postgresql-ai-$REGION
    ```

    Der letzte Befehl generiert nach dem Zufallsprinzip ein Kennwort für das PostgreSQL-Admin-Login. **Stellen Sie sicher, dass Sie sie** an einen sicheren Ort kopieren, um sie später für die Verbindung mit Ihrem PostgreSQL – Flexibler Server zu verwenden.

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

5. Wenn Sie Zugriff auf mehr als ein Azure-Abonnement haben und Ihr Standardabonnement nicht dasjenige ist, in dem Sie die Ressourcengruppe und andere Ressourcen für diese Übung erstellen möchten, führen Sie diesen Befehl aus, um das entsprechende Abonnement festzulegen. Ersetzen Sie dabei das Token `<subscriptionName|subscriptionId>` durch den Namen oder die ID des Abonnements, das Sie verwenden möchten:

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

    Das Bicep-Bereitstellungsskript stellt die für diese Übung erforderlichen Azure-Service in Ihrer Ressourcengruppe bereit. Zu den bereitgestellten Ressourcen gehören ein Azure Database for PostgreSQL – Flexibler Server, Azure OpenAI und ein Azure KI Language-Service. Das Bicep-Skript führt auch einige Konfigurationsschritte aus, wie z. B. das Hinzufügen der Erweiterungen `azure_ai` und `vector` zur _Positivliste_ des PostgreSQL-Servers (über den Serverparameter azure.extensions), das Erstellen einer Datenbank mit dem Namen `rentals` auf dem Server und das Hinzufügen einer Bereitstellung mit dem Namen `embedding` unter Verwendung des Modells `text-embedding-ada-002` zu Ihrem Azure OpenAI-Service. Beachten Sie, dass die Bicep-Datei von allen Modulen in diesem Lernpfad gemeinsam genutzt wird, sodass Sie möglicherweise nur einige der bereitgestellten Ressourcen in einigen Übungen verwenden.

    Die Bereitstellung dauert in der Regel mehrere Minuten. Sie können es von der Cloud Shell aus überwachen oder zur Seite **Bereitstellungen** für die oben erstellte Ressourcengruppe navigieren und dort den Bereitstellungsfortschritt beobachten.

8. Schließen Sie den Cloud Shell-Bereich, sobald Ihre Ressourcenbereitstellung abgeschlossen ist.

### Problembehandlung bei der Bereitstellung

Beim Ausführen des Bicep-Bereitstellungsskripts können einige Fehler auftreten. Die häufigsten Meldungen und die Schritte zu ihrer Behebung sind:

- Wenn Sie zuvor das Bicep-Bereitstellungsskript für diesen Lernpfad ausgeführt und anschließend die Ressourcen gelöscht haben, erhalten Sie möglicherweise eine Fehlermeldung wie die folgende, wenn Sie versuchen, das Skript innerhalb von 48 Stunden nach dem Löschen der Ressourcen erneut auszuführen:

    ```bash
    {"code": "InvalidTemplateDeployment", "message": "The template deployment 'deploy' is not valid according to the validation procedure. The tracking id is '4e87a33d-a0ac-4aec-88d8-177b04c1d752'. See inner errors for details."}
    
    Inner Errors:
    {"code": "FlagMustBeSetForRestore", "message": "An existing resource with ID '/subscriptions/{subscriptionId}/resourceGroups/rg-learn-postgresql-ai-eastus/providers/Microsoft.CognitiveServices/accounts/{accountName}' has been soft-deleted. To restore the resource, you must specify 'restore' to be 'true' in the property. If you don't want to restore existing resource, please purge it first."}
    ```

    Wenn Sie diese Meldung erhalten, ändern Sie den obigen Befehl `azure deployment group create`, um den Parameter `restore` auf `true` zu setzen und führen Sie ihn erneut aus.

- Wenn die ausgewählte Region für die Bereitstellung bestimmter Ressourcen eingeschränkt ist, müssen Sie die Variable `REGION` auf einen anderen Speicherort setzen und die Befehle erneut ausführen, um die Ressourcengruppe zu erstellen und das Bicep-Bereitstellungsskript auszuführen.

    ```bash
    {"status":"Failed","error":{"code":"DeploymentFailed","target":"/subscriptions/{subscriptionId}/resourceGroups/{resourceGrouName}/providers/Microsoft.Resources/deployments/{deploymentName}","message":"At least one resource deployment operation failed. Please list deployment operations for details. Please see https://aka.ms/arm-deployment-operations for usage details.","details":[{"code":"ResourceDeploymentFailure","target":"/subscriptions/{subscriptionId}/resourceGroups/{resourceGrouName}/providers/Microsoft.DBforPostgreSQL/flexibleServers/{serverName}","message":"The resource write operation failed to complete successfully, because it reached terminal provisioning state 'Failed'.","details":[{"code":"RegionIsOfferRestricted","message":"Subscriptions are restricted from provisioning in this region. Please choose a different region. For exceptions to this rule please open a support request with Issue type of 'Service and subscription limits'. See https://review.learn.microsoft.com/en-us/azure/postgresql/flexible-server/how-to-request-quota-increase for more details."}]}]}}
    ```

- Wenn das Skript aufgrund der Anforderung, die Vereinbarung über die verantwortungsvolle KI zu akzeptieren, keine KI-Ressource erstellen kann, kann der folgende Fehler auftreten. Verwenden Sie in diesem Fall die Benutzeroberfläche des Azure-Portals, um eine Azure-KI-Services-Ressource zu erstellen und führen Sie das Bereitstellungsskript dann erneut aus.

    ```bash
    {"code": "InvalidTemplateDeployment", "message": "The template deployment 'deploy' is not valid according to the validation procedure. The tracking id is 'f8412edb-6386-4192-a22f-43557a51ea5f'. See inner errors for details."}
     
    Inner Errors:
    {"code": "ResourceKindRequireAcceptTerms", "message": "This subscription cannot create TextAnalytics until you agree to Responsible AI terms for this resource. You can agree to Responsible AI terms by creating a resource through the Azure Portal then trying again. For more detail go to https://go.microsoft.com/fwlink/?linkid=2164190"}
    ```

## Verbinden Sie sich mit Ihrer Datenbank mit psql in der Azure Cloud Shell

In dieser Aufgabe stellen Sie eine Verbindung zur `rentals`-Datenbank auf Ihrem Azure Database for PostgreSQL-Server her, indem Sie das [psql Befehlszeilendienstprogramm](https://www.postgresql.org/docs/current/app-psql.html) aus der [Azure Cloud Shell](https://learn.microsoft.com/azure/cloud-shell/overview) verwenden.

1. Navigieren Sie im [Azure-Portal](https://portal.azure.com/) zu Ihrer neu erstellten Azure Database for PostgreSQL – Flexibler Serverinstanz.

2. Wählen Sie im Ressourcenmenü unter **Einstellungen** die Option **Datenbanken** und dann **Verbinden** für die Datenbank `rentals`.

    ![Screenshot der Seite Azure-Datenbank für PostgreSQL-Datenbanken. Datenbanken und Verbinden für die Vermietungsdatenbank sind durch rote Boxen hervorgehoben.](media/17-postgresql-rentals-database-connect.png)

3. Geben Sie bei der Eingabeaufforderung „Kennwort für den Benutzer pgAdmin“ in der Cloud Shell das zufällig generierte Kennwort für die Anmeldung **pgAdmin** ein.

    Sobald Sie angemeldet sind, wird die Eingabeaufforderung `psql` für die Datenbank `rentals` angezeigt.

## Füllen der Datenbank mit Beispieldaten

Bevor Sie die Stimmung der Bewertungen von Mietobjekten mit der `azure_ai`-Erweiterung analysieren können, müssen Sie Beispieldaten zu Ihrer Datenbank hinzufügen. Fügen Sie der `rentals`-Datenbank eine Tabelle hinzu und füllen Sie sie mit Kundenrezensionen, damit Sie Daten haben, mit denen Sie eine Stimmungsanalyse durchführen können.

1. Führen Sie den folgenden Befehl aus, um eine Tabelle mit dem Namen `reviews` zu erstellen, in der die von Kundinnen und Kunden abgegebenen Immobilienbewertungen gespeichert werden:

    ```sql
    DROP TABLE IF EXISTS reviews;

    CREATE TABLE reviews (
        id int,
        listing_id int, 
        date date,
        comments text
    );
    ```

2. Als Nächstes verwenden Sie den Befehl `COPY`, um die Tabelle mit Daten aus einer CSV-Datei zu füllen. Führen Sie den folgenden Befehl aus, um Kundenrezensionen in die Tabelle `reviews` zu laden:

    ```sql
    \COPY reviews FROM 'mslearn-postgresql/Allfiles/Labs/Shared/reviews.csv' CSV HEADER
    ```

    Die Befehlsausgabe sollte `COPY 354` lauten, was bedeutet, dass 354 Zeilen aus der CSV-Datei in die Tabelle geschrieben wurden.

## Installieren und konfigurieren Sie die `azure_ai`-Erweiterung

Bevor Sie die `azure_ai`-Erweiterung verwenden, müssen Sie sie in Ihrer Datenbank installieren und für die Verbindung mit Ihren Azure KI Services-Ressourcen konfigurieren. Mit der `azure_ai`-Erweiterung können Sie die Azure OpenAI und Azure KI Language-Services in Ihre Datenbank integrieren. Um die Erweiterung in Ihrer Datenbank zu aktivieren, gehen Sie folgendermaßen vor:

1. Führen Sie den folgenden Befehl an der `psql`-Eingabeaufforderung aus, um zu überprüfen, ob die `azure_ai`- und die `vector`-Erweiterungen durch das Bicep-Bereitstellungsskript, das Sie beim Einrichten Ihrer Umgebung ausgeführt haben, erfolgreich zur _Positivliste_ Ihres Servers hinzugefügt wurden:

    ```sql
    SHOW azure.extensions;
    ```

    Der Befehl zeigt die Liste der Erweiterungen in der _Positivliste_ des Servers an. Wenn alles korrekt installiert wurde, muss Ihre Ausgabe `azure_ai` und `vector` enthalten, etwa so:

    ```sql
     azure.extensions 
    ------------------
     azure_ai,vector
    ```

    Bevor eine Erweiterung in einer Azure Database for PostgreSQL – Flexibler Server-Datenbank installiert und verwendet werden kann, muss sie zur _Positivliste_ des Servers hinzugefügt werden, wie in [Wie man PostgreSQL-Erweiterungen verwendet](https://learn.microsoft.com/azure/postgresql/flexible-server/concepts-extensions#how-to-use-postgresql-extensions) beschrieben.

2. Jetzt können Sie die `azure_ai`-Erweiterung mit dem Befehl [CREATE EXTENSION](https://www.postgresql.org/docs/current/sql-createextension.html) installieren.

    ```sql
    CREATE EXTENSION IF NOT EXISTS azure_ai;
    ```

    `CREATE EXTENSION` lädt eine neue Erweiterung in die Datenbank, indem die zugehörige Skriptdatei ausgeführt wird. Dieses Skript erstellt normalerweise neue SQL-Objekte wie Funktionen, Datentypen und Schemata. Es wird ein Fehler ausgegeben, wenn eine Erweiterung mit demselben Namen bereits existiert. Durch Hinzufügen von `IF NOT EXISTS` kann der Befehl ausgeführt werden, ohne dass eine Fehlermeldung ausgegeben wird, wenn er bereits installiert ist.

## Verbinden Sie Ihr Azure KI Services-Konto

Die im `azure_cognitive` Schema der `azure_ai` Erweiterung enthaltenen Azure KI Services-Integrationen bieten eine umfangreiche Sammlung von KI-Sprachfeatures, die direkt über die Datenbank zugänglich sind. Die Funktionen zur Stimmungsanalyse werden durch den [Azure KI Language-Service](https://learn.microsoft.com/azure/ai-services/language-service/overview) aktiviert.

1. Damit Sie Ihre Azure KI Language-Services mit der Erweiterung `azure_ai` erfolgreich anrufen können, müssen Sie der Erweiterung den Endpunkt und den Schlüssel mitteilen. Navigieren Sie in derselben Browser-Registerkarte, in der die Cloud Shell geöffnet ist, zu Ihrer Sprachdienst-Ressource im [Azure-Portal](https://portal.azure.com/) und wählen Sie im linken Navigationsmenü unter **Ressourcenverwaltung** den Punkt **Schlüssel und Endpunkt** aus.

    ![Es wird ein Screenshot der Seite Schlüssel und Endpunkte des Azure-Sprachdienstes angezeigt. Die Schaltflächen KEY 1 und Endpunkt kopieren sind durch rote Felder hervorgehoben.](media/16-azure-language-service-keys-endpoints.png)

    > [!Note]
    >
    > Wenn Sie bei der Installation der Erweiterung `azure_ai` die `NOTICE: extension "azure_ai" already exists, skipping CREATE EXTENSION` -Meldung erhalten haben und die Erweiterung bereits mit Ihrem Sprachdienst-Endpunkt und -Schlüssel konfiguriert ist, können Sie die `azure_ai.get_setting()`-Funktion verwenden, um zu überprüfen, ob diese Einstellungen korrekt sind und dann Schritt 2 überspringen, wenn sie korrekt sind.

2. Kopieren Sie die Werte für den Endpunkt und den Zugriffsschlüssel. Ersetzen Sie dann in den folgenden Befehlen die `{endpoint}` und `{api-key}` Token durch die Werte, die Sie aus dem Azure-Portal kopiert haben. Führen Sie die Befehle von der Eingabeaufforderung `psql` in der Cloud Shell aus, um Ihre Werte der `azure_ai.settings`-Tabelle hinzuzufügen.

    ```sql
    SELECT azure_ai.set_setting('azure_cognitive.endpoint', '{endpoint}');
    ```

    ```sql
    SELECT azure_ai.set_setting('azure_cognitive.subscription_key', '{api-key}');
    ```

## Überprüfen der Funktionen der Erweiterung zur Analyse von Gefühlen

In dieser Aufgabe verwenden Sie die Funktion `azure_cognitive.analyze_sentiment()`, um Bewertungen von Mietangeboten auszuwerten.

1. Da Sie im weiteren Verlauf dieser Übung ausschließlich in der Cloud Shell arbeiten, kann es hilfreich sein, den Bereich in Ihrem Browserfenster zu erweitern, indem Sie die Schaltfläche **Maximieren** oben rechts im Bereich der Cloud Shell wählen.

    ![Screenshot des Azure Cloud Shell-Fensters mit der Schaltfläche „Maximieren“, die durch eine rote Box hervorgehoben ist.](media/16-azure-cloud-shell-pane-maximize.png)

2. Bei der Arbeit mit `psql` in der Cloud Shell kann es hilfreich sein, die erweiterte Anzeige für Abfrageergebnisse zu aktivieren, da dies die Lesbarkeit der Ausgabe für nachfolgende Befehle verbessert. Führen Sie den folgenden Befehl aus, damit die erweiterte Anzeige automatisch angewendet wird.

    ```sql
    \x auto
    ```

3. Die Funktionen der `azure_ai`-Erweiterung für die Stimmungsanalyse befinden sich im `azure_cognitive`-Schema. Verwenden der `analyze_sentiment()`-Funktion. Verwenden Sie den [`\df` Meta-Befehl](https://www.postgresql.org/docs/current/app-psql.html#APP-PSQL-META-COMMAND-DF-LC), um die Funktion durch Ausführen zu untersuchen:

    ```sql
    \df azure_cognitive.analyze_sentiment
    ```

    Die Ausgabe des Meta-Befehls zeigt das Schema, den Namen, den Ergebnisdatentyp und die Argumente der Funktion an. Diese Informationen helfen Ihnen zu verstehen, wie Sie mit der Funktion in Ihren Abfragen interagieren können.

    Die Ausgabe zeigt drei Überlastungen der Funktion `analyze_sentiment()`, sodass Sie deren Unterschiede überprüfen können. Die Eigenschaft `Argument data types` in der Ausgabe zeigt die Liste der Argumente, die die drei Funktionsüberladungen erwarten:

    | Argument | Typ | Standard | Beschreibung |
    | -------- | ---- | ------- | ----------- |
    | Text | `text` oder `text[]` || Der Text bzw. die Texte, für die die Stimmung analysiert werden soll. |
    | language_text | `text` oder `text[]` || Sprachcode (oder Array von Sprachcodes), der die Sprache des Textes angibt, der auf die Stimmung analysiert werden soll. Überprüfen Sie die [Liste der unterstützten Sprachen](https://learn.microsoft.com/azure/ai-services/language-service/sentiment-opinion-mining/language-support), um die erforderlichen Sprachcodes abzurufen. |
    | batch_size | `integer` | 10 | Nur für die beiden Überladungen, die eine Eingabe von `text[]` erwarten. Gibt die Anzahl der gleichzeitig zu verarbeitenden Datensätze an. |
    | disable_service_logs | `boolean` | false | Flag, das angibt, ob Dienstprotokolle deaktiviert werden sollen. |
    | timeout_ms | `integer` | NULL | Timeout in Millisekunden, nach dem der Vorgang beendet wird. |
    | throw_on_error | `boolean` | true | Flag, das angibt, ob die Funktion beim Fehler eine Ausnahme auslösen soll, was zu einem Rollback der Umbruchtransaktionen führt. |
    | max_attempts | `integer` | 1 | Anzahl der Wiederholungen des Aufrufs an Azure KI Services im Falle eines Fehlers. |
    | retry_delay_ms | `integer` | 1.000 | Die Zeit (in Millisekunden), die gewartet werden muss, bevor versucht wird, den Azure KI Services-Endpunkt erneut aufzurufen. |

4. Es ist auch wichtig, die Struktur des Datentyps zu verstehen, den eine Funktion zurückgibt, damit Sie die Ausgabe in Ihren Abfragen richtig behandeln können. Führen Sie den folgenden Befehl aus, um den Typ `sentiment_analysis_result` zu überprüfen:

    ```sql
    \dT+ azure_cognitive.sentiment_analysis_result
    ```

5. Die Ausgabe des obigen Befehls zeigt, dass der Typ `sentiment_analysis_result` ein `tuple` ist. Sie können die Struktur dieses `tuple` näher untersuchen, indem Sie den folgenden Befehl ausführen, um die im Typ `sentiment_analysis_result` enthaltenen Spalten zu betrachten:

    ```sql
    \d+ azure_cognitive.sentiment_analysis_result
    ```

    Die Ausgabe dieses Befehls sollte etwa so aussehen wie die folgende:

    ```sql
                     Composite type "azure_cognitive.sentiment_analysis_result"
         Column     |     Type         | Collation | Nullable | Default | Storage  | Description 
    ----------------+------------------+-----------+----------+---------+----------+-------------
     sentiment      | text             |           |          |         | extended | 
     positive_score | double precision |           |          |         | plain    | 
     neutral_score  | double precision |           |          |         | plain    | 
     negative_score | double precision |           |          |         | plain    |
    ```

    Das `azure_cognitive.sentiment_analysis_result` ist ein zusammengesetzter Typ, der die Stimmungsvorhersagen des Eingabetextes enthält. Er enthält die Stimmung, die positiv, negativ, neutral oder gemischt sein kann, und die Bewertungen für positive, neutrale und negative Aspekte im Text. Die Ergebnisse werden als reale Zahlen zwischen 0 und 1 dargestellt. Beispielsweise ist die Stimmung in (neutral, 0,26, 0,64, 0,09) neutral, mit einem positiven Ergebnis von 0,26, neutral von 0,64 und negativ bei 0,09.

## Analysieren der Stimmung der Bewertungen

1. Nachdem Sie nun die Funktion `analyze_sentiment()` und das `sentiment_analysis_result`, das sie zurückgibt, kennengelernt haben, lassen Sie uns die Funktion in die Praxis umsetzen. Führen Sie die folgende einfache Abfrage aus, die eine Stimmungsanalyse für eine Handvoll Kommentare in der Tabelle `reviews` durchführt:

    ```sql
    SELECT
        id,
        azure_cognitive.analyze_sentiment(comments, 'en') AS sentiment
    FROM reviews
    WHERE id <= 10
    ORDER BY id;
    ```

    Beachten Sie bei den beiden analysierten Datensätzen die Werte `sentiment` in der Ausgabe, `(mixed,0.71,0.09,0.2)` und `(positive,0.99,0.01,0)`. Diese stellen die `sentiment_analysis_result` dar, die von der Funktion `analyze_sentiment()` in der obigen Abfrage zurückgegeben werden. Die Analyse wurde über das Feld `comments` in der Tabelle `reviews` durchgeführt.

    > [!Note]
    >
    > Wenn Sie die Funktion `analyze_sentiment()` inline verwenden, können Sie die Stimmung des Textes innerhalb Ihrer Abfragen schnell analysieren. Während dies für eine kleine Anzahl von Datensätzen gut funktioniert, ist es möglicherweise nicht ideal, um die Stimmung einer großen Anzahl von Datensätzen zu analysieren oder alle Datensätze in einer Tabelle zu aktualisieren, die Zehntausende von Bewertungen oder mehr enthalten kann.

2. Ein weiterer Ansatz, der bei längeren Rezensionen nützlich sein kann, ist die Analyse der Stimmung jedes einzelnen Satzes darin. Verwenden Sie dazu die Überladung der Funktion `analyze_sentiment()`, die ein Array mit Text akzeptiert.

    ```sql
    SELECT
        azure_cognitive.analyze_sentiment(ARRAY_REMOVE(STRING_TO_ARRAY(comments, '.'), ''), 'en') AS sentence_sentiments
    FROM reviews
    WHERE id = 1;
    ```

    In der obigen Abfrage haben Sie die Funktion `STRING_TO_ARRAY` von PostgreSQL verwendet. Außerdem wurde die Funktion `ARRAY_REMOVE` verwendet, um alle Array-Elemente zu entfernen, die leere Strings sind, da diese bei der Funktion `analyze_sentiment()` Fehler verursachen.

    Die Ausgabe der Abfrage ermöglicht Ihnen ein besseres Verständnis der `mixed` Stimmung, die der gesamten Rezension zugeordnet ist. Die Sätze sind eine Mischung aus positiven, neutralen und negativen Gefühlen.

3. Die beiden vorangegangenen Abfragen haben die `sentiment_analysis_result` direkt aus der Abfrage zurückgegeben. Sie werden jedoch wahrscheinlich die zugrunde liegenden Werte innerhalb der `sentiment_analysis_result``tuple` abrufen wollen. Führen Sie die folgende Abfrage aus, die nach überwiegend positiven Bewertungen sucht und die Stimmungskomponenten in einzelne Felder extrahiert:

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

    Die obige Abfrage verwendet einen gemeinsamen Tabellenausdruck oder CTE, um die Stimmungsbewertungen für alle Datensätze in der Tabelle `reviews` zu erhalten. Es wählt dann die `sentiment`-Spalten des zusammengesetzten Typs aus dem `sentiment_analysis_result`, die von der CTE zurückgegeben werden, um die einzelnen Werte aus `tuple.` zu extrahieren.

## Speichern der Stimmung in der Bewertungstabelle

Für das Empfehlungssystem für Mietobjekte, das Sie für Margie's Travel entwickeln, möchten Sie Stimmungsbewertungen in der Datenbank speichern, damit Sie nicht jedes Mal, wenn Stimmungsbewertungen angefordert werden, Anrufe tätigen und Kosten verursachen müssen. Die spontane Durchführung von Stimmungsanalysen kann bei einer kleinen Anzahl von Datensätzen oder bei der Analyse von Daten in nahezu Echtzeit sehr nützlich sein. Dennoch ist es für Ihre gespeicherten Bewertungen sinnvoll, die Stimmungsdaten zur Verwendung in Ihrer Anwendung in die Datenbank aufzunehmen. Dazu müssen Sie die Tabelle `reviews` ändern, um Spalten für die Speicherung der Stimmungsbewertung und der positiven, neutralen und negativen Bewertungen hinzuzufügen.

1. Führen Sie die folgende Abfrage aus, um die Tabelle `reviews` zu aktualisieren, sodass sie die Details der Gefühle speichern kann:

    ```sql
    ALTER TABLE reviews
    ADD COLUMN sentiment varchar(10),
    ADD COLUMN positive_score numeric,
    ADD COLUMN neutral_score numeric,
    ADD COLUMN negative_score numeric;
    ```

2. Als nächstes möchten Sie die vorhandenen Datensätze in der Tabelle `reviews` mit ihrem Stimmungswert und den zugehörigen Bewertungen aktualisieren.

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

    Die Ausführung dieser Abfrage nimmt viel Zeit in Anspruch, da die Kommentare für jede Rezension in der Tabelle einzeln an den Endpunkt des Sprachdienstes zur Analyse gesendet werden. Das Versenden von Datensätzen in Stapeln ist effizienter, wenn es sich um viele Datensätze handelt.

3. Führen Sie die folgende Abfrage aus, um dieselbe Aktualisierungsaktion durchzuführen, aber senden Sie diesmal Kommentare aus der Tabelle `reviews` in Stapeln von 10 (dies ist die maximal zulässige Stapelgröße) und bewerten Sie den Unterschied in der Leistung.

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

    Diese Abfrage ist zwar etwas komplexer, da sie zwei CTEs verwendet, aber sie ist viel leistungsfähiger. In dieser Abfrage analysiert die erste CTE die Stimmung von Stapeln von Überprüfungskommentaren und die zweite extrahiert die resultierende Tabelle `sentiment_analysis_results` in eine neue Tabelle, die ein `id` enthält, basierend auf der ordinalen Position und dem ` `Sentiment_analysis_result` für jede Zeile. Das zweite CTE kann dann in der Aktualisierungsanweisung verwendet werden, um die Werte in die Datenbank zu schreiben.

4. Führen Sie als Nächstes eine Abfrage aus, um die Aktualisierungsergebnisse zu beobachten und suchen Sie nach Bewertungen mit einer **negativen** Stimmung, beginnend mit der negativsten zuerst.

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

Sobald Sie diese Übung abgeschlossen haben, löschen Sie die von Ihnen erstellten Azure-Ressourcen. Sie zahlen für die konfigurierte Kapazität, nicht dafür, wie viel die Datenbank genutzt wird. Folgen Sie diesen Anweisungen, um Ihre Ressourcengruppe und alle Ressourcen, die Sie für dieses Lab erstellt haben, zu löschen.

1. Öffnen Sie einen Webbrowser und navigieren Sie zum [Azure-Portal](https://portal.azure.com/). Wählen Sie auf der Startseite die Option **Ressourcengruppen** unter „Azure-Services“ aus.

    ![Screenshot der Ressourcengruppen, die im Azure-Portal unter Azure-Service durch ein rotes Feld hervorgehoben werden.](media/16-azure-portal-home-azure-services-resource-groups.png)

2. Geben Sie in das Suchfeld Filter für ein beliebiges Feld den Namen der Ressourcengruppe ein, die Sie für dieses Lab erstellt haben und wählen Sie dann Ihre Ressourcengruppe aus der Liste aus.

3. Wählen Sie auf der Seite **Übersicht** der Ressourcengruppe die Option **Ressourcengruppe löschen** aus.

    ![Screenshot des Übersichtsblatts der Ressourcengruppe, wobei die Schaltfläche „Ressourcengruppe löschen“ durch eine rote Box hervorgehoben ist.](media/16-resource-group-delete.png)

4. Geben Sie im Bestätigungsdialog zur Bestätigung den Namen der Ressourcengruppe ein, die Sie löschen möchten und wählen Sie dann **Löschen**.
