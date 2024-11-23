---
lab:
  title: Durchführen einer extraktive und abstrakte Zusammenfassungen
  module: Summarize data using Azure AI Services and Azure Database for PostgreSQL
---

# Durchführen einer extraktive und abstrakte Zusammenfassungen

Die von Margie's Travel verwaltete App für Mietobjekte bietet Immobilienverwaltern die Möglichkeit, Mietangebote zu beschreiben. Viele der Beschreibungen im System sind lang und enthalten viele Details über das Mietobjekt, die Nachbarschaft und lokale Attraktionen, Geschäfte und andere Annehmlichkeiten. Eine Funktion, die bei der Implementierung neuer KI-gestützter Funktionen für die App angefragt wurde, ist die Verwendung generativer KI zur Erstellung prägnanter Zusammenfassungen dieser Beschreibungen, die es Ihren Nutzern erleichtern, Immobilien schnell zu überprüfen. In dieser Übung verwenden Sie die Erweiterung `azure_ai` in einem Azure Database for PostgreSQL – Flexibler Server, um abstrakte und extraktive Zusammenfassungen von Mietobjektbeschreibungen durchzuführen und die resultierenden Zusammenfassungen zu vergleichen.

## Vor der Installation

Sie benötigen ein [Azure-Abonnement](https://azure.microsoft.com/free) mit administrativen Rechten und müssen für den Azure OpenAI-Zugang in diesem Abonnement zugelassen sein. Wenn Sie Zugriff auf Azure OpenAI benötigen, bewerben Sie sich auf der Seite [Eingeschränkter Zugriff auf Azure OpenAI](https://learn.microsoft.com/legal/cognitive-services/openai/limited-access).

### Bereitstellen von Ressourcen in Ihrem Azure-Abonnement

Dieser Schritt führt Sie durch die Verwendung von Azure CLI-Befehlen aus der Azure Cloud Shell, um eine Ressourcengruppe zu erstellen und ein Bicep-Skript auszuführen, um die für diese Übung erforderlichen Azure-Services in Ihrem Azure-Abonnement bereitzustellen.

1. Öffnen Sie einen Webbrowser, und navigieren Sie zum [Azure-Portal](https://portal.azure.com/).

2. Wählen Sie das Symbol **Cloud Shell** in der Symbolleiste des Azure-Portals aus, um einen neuen [Cloud Shell](https://learn.microsoft.com/azure/cloud-shell/overview)-Bereich am unteren Rand Ihres Browserfensters zu öffnen.

    ![Screenshot der Azure-Symbolleiste mit dem Cloud Shell-Symbol, das durch eine rote Box hervorgehoben ist.](media/15-portal-toolbar-cloud-shell.png)

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

Beim Ausführen des Bicep-Bereitstellungsskripts können einige Fehler auftreten.

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

In dieser Aufgabe verbinden Sie sich mit der `rentals`-Datenbank auf Ihrem Azure Database for PostgreSQL – Flexibler Server mit dem [psql Befehlszeilen-Dienstprogramm](https://www.postgresql.org/docs/current/app-psql.html) von der [Azure Cloud Shell](https://learn.microsoft.com/azure/cloud-shell/overview) aus.

1. Navigieren Sie im [Azure-Portal](https://portal.azure.com/) zu Ihrer neu erstellten Azure Database for PostgreSQL – Flexibler Serverinstanz.

2. Wählen Sie im Ressourcenmenü unter **Einstellungen** die Option **Datenbanken** und dann **Verbinden** für die Datenbank `rentals`.

    ![Screenshot der Seite Azure-Datenbank für PostgreSQL-Datenbanken. Datenbanken und Verbinden für die Vermietungsdatenbank sind durch rote Boxen hervorgehoben.](media/15-postgresql-rentals-database-connect.png)

3. Geben Sie bei der Eingabeaufforderung „Kennwort für den Benutzer pgAdmin“ in der Cloud Shell das zufällig generierte Kennwort für die Anmeldung **pgAdmin** ein.

    Sobald Sie angemeldet sind, wird die Eingabeaufforderung `psql` für die Datenbank `rentals` angezeigt.

4. Im weiteren Verlauf dieser Übung arbeiten Sie weiterhin in der Cloud Shell. Daher kann es hilfreich sein, den Bereich in Ihrem Browserfenster zu erweitern, indem Sie die Schaltfläche **Maximieren** oben rechts im Bereich wählen.

    ![Screenshot des Azure Cloud Shell-Fensters mit der Schaltfläche „Maximieren“, die durch eine rote Box hervorgehoben ist.](media/15-azure-cloud-shell-pane-maximize.png)

## Füllen der Datenbank mit Beispieldaten

Bevor Sie sich mit der `azure_ai`-Erweiterung befassen, fügen Sie der `rentals`-Datenbank einige Tabellen hinzu und füllen sie mit Beispieldaten, damit Sie Informationen haben, mit denen Sie arbeiten können, wenn Sie die Funktionen der Erweiterung prüfen.

1. Führen Sie die folgenden Befehle aus, um die Tabellen `listings` und `reviews` zu erstellen, in denen die Daten für die Auflistung von Mietobjekten und die Kundenrezensionen gespeichert werden:

    ```sql
    DROP TABLE IF EXISTS listings;

    CREATE TABLE listings (
        id int,
        name varchar(100),
        description text,
        property_type varchar(25),
        room_type varchar(30),
        price numeric,
        weekly_price numeric
    );
    ```

    ```sql
    DROP TABLE IF EXISTS reviews;

    CREATE TABLE reviews (
        id int,
        listing_id int, 
        date date,
        comments text
    );
    ```

2. Als Nächstes verwenden Sie den Befehl `COPY`, um Daten aus CSV-Dateien in jede Tabelle zu laden, die Sie oben erstellt haben. Beginnen Sie, indem Sie den folgenden Befehl ausführen, um die Tabelle `listings` aufzufüllen:

    ```sql
    \COPY listings FROM 'mslearn-postgresql/Allfiles/Labs/Shared/listings.csv' CSV HEADER
    ```

    Die Befehlsausgabe sollte `COPY 50` lauten, was bedeutet, dass 50 Zeilen aus der CSV-Datei in die Tabelle geschrieben wurden.

3. Führen Sie abschließend den folgenden Befehl aus, um Kundenrezensionen in die Tabelle `reviews` zu laden:

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

Die im `azure_cognitive` Schema der `azure_ai` Erweiterung enthaltenen Azure KI Services-Integrationen bieten eine umfangreiche Sammlung von KI-Sprachfeatures, die direkt über die Datenbank zugänglich sind. Die Funktionen zur Textzusammenfassung werden über den [Azure KI Language-Service](https://learn.microsoft.com/azure/ai-services/language-service/overview) aktiviert.

1. Damit Sie Ihre Azure KI Language-Services mit der Erweiterung `azure_ai` erfolgreich anrufen können, müssen Sie der Erweiterung den Endpunkt und den Schlüssel mitteilen. Navigieren Sie in derselben Browser-Registerkarte, in der die Cloud Shell geöffnet ist, zu Ihrer Sprachdienst-Ressource im [Azure-Portal](https://portal.azure.com/) und wählen Sie im linken Navigationsmenü unter **Ressourcenverwaltung** den Punkt **Schlüssel und Endpunkt** aus.

    ![Es wird ein Screenshot der Seite Schlüssel und Endpunkte des Azure-Sprachdienstes angezeigt. Die Schaltflächen KEY 1 und Endpunkt kopieren sind durch rote Felder hervorgehoben.](media/15-azure-language-service-keys-endpoints.png)

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

## Überprüfen der Zusammenfassungsfunktionen der Erweiterung

In dieser Aufgabe überprüfen Sie die beiden Verdichtungsfunktionen im `azure_cognitive`-Schema.

1. Da Sie im weiteren Verlauf dieser Übung ausschließlich in der Cloud Shell arbeiten, kann es hilfreich sein, den Bereich in Ihrem Browserfenster zu erweitern, indem Sie die Schaltfläche **Maximieren** oben rechts im Bereich der Cloud Shell wählen.

    ![Screenshot des Azure Cloud Shell-Fensters mit der Schaltfläche „Maximieren“, die durch eine rote Box hervorgehoben ist.](media/15-azure-cloud-shell-pane-maximize.png)

2. Bei der Arbeit mit `psql` in der Cloud Shell kann es hilfreich sein, die erweiterte Anzeige für Abfrageergebnisse zu aktivieren, da dies die Lesbarkeit der Ausgabe für nachfolgende Befehle verbessert. Führen Sie den folgenden Befehl aus, damit die erweiterte Anzeige automatisch angewendet wird.

    ```sql
    \x auto
    ```

3. Die Textverdichtungsfunktionen der Erweiterung `azure_ai` befinden sich im Schema `azure_cognitive`. Für die extraktive Verdichtung verwenden Sie die Funktion `summarize_extractive()`. Verwenden Sie den [`\df` Meta-Befehl](https://www.postgresql.org/docs/current/app-psql.html#APP-PSQL-META-COMMAND-DF-LC), um die Funktion durch Ausführen zu untersuchen:

    ```sql
    \df azure_cognitive.summarize_extractive
    ```

    Die Ausgabe des Meta-Befehls zeigt das Schema, den Namen, den Ergebnisdatentyp und die Argumente der Funktion an. Diese Informationen helfen Ihnen zu verstehen, wie Sie mit der Funktion in Ihren Abfragen interagieren können.

    Die Ausgabe zeigt drei Überlastungen der Funktion `summarize_extractive()`, sodass Sie deren Unterschiede überprüfen können. Die Eigenschaft `Argument data types` in der Ausgabe zeigt die Liste der Argumente, die die drei Funktionsüberladungen erwarten:

    | Argument | Typ | Standard | Beschreibung |
    | -------- | ---- | ------- | ----------- |
    | Text | `text` oder `text[]` || Die Texte, für die Zusammenfassungen generiert werden sollen. |
    | language_text | `text` oder `text[]` || Sprachcode (oder Array von Sprachcodes), der die Sprache des zusammenzufassenden Texts darstellt. Überprüfen Sie die [Liste der unterstützten Sprachen](https://learn.microsoft.com/azure/ai-services/language-service/summarization/language-support), um die erforderlichen Sprachcodes abzurufen. |
    | sentence_count | `integer` | 3 | Die Anzahl der zu generierenden Sammelsätze. |
    | sort_by | `text` | 'offset' | Die Sortierreihenfolge für die generierten Sammelsätze. Zulässige Werte sind „offset“ und „rank“, wobei der Offset die Startposition jedes extrahierten Satzes innerhalb des ursprünglichen Inhalts darstellt und als KI-generierter Indikator für die Relevanz eines Satzes für die Hauptidee des Inhalts einen Rang zuweist. |
    | batch_size | `integer` | 25 | Nur für die beiden Überladungen, die eine Eingabe von `text[]` erwarten. Gibt die Anzahl der gleichzeitig zu verarbeitenden Datensätze an. |
    | disable_service_logs | `boolean` | false | Flag, das angibt, ob Dienstprotokolle deaktiviert werden sollen. |
    | timeout_ms | `integer` | NULL | Timeout in Millisekunden, nach dem der Vorgang beendet wird. |
    | throw_on_error | `boolean` | true | Flag, das angibt, ob die Funktion beim Fehler eine Ausnahme auslösen soll, was zu einem Rollback der Umbruchtransaktionen führt. |
    | max_attempts | `integer` | 1 | Anzahl der Wiederholungen des Aufrufs an Azure KI Services im Falle eines Fehlers. |
    | retry_delay_ms | `integer` | 1.000 | Die Zeit (in Millisekunden), die gewartet werden muss, bevor versucht wird, den Azure KI Services-Endpunkt erneut aufzurufen. |

4. Wiederholen Sie den obigen Schritt, aber führen Sie dieses Mal den [`\df` Meta-Befehl](https://www.postgresql.org/docs/current/app-psql.html#APP-PSQL-META-COMMAND-DF-LC) für die Funktion `azure_cognitive.summarize_abstractive()` aus und überprüfen Sie die Ausgabe.

    Die beiden Funktionen haben ähnliche Signaturen, obwohl `summarize_abstractive()` keinen `sort_by`-Parameter hat und ein Array von `text` gegenüber dem Array von `azure_cognitive.sentence` zusammengesetzten Typen zurückgibt, das von der Funktion `summarize_extractive()` zurückgegeben wird. Diese Diskrepanz hat mit der Art und Weise zu tun, wie die beiden unterschiedlichen Methoden Zusammenfassungen erstellen. Die extraktive Zusammenfassung identifiziert die kritischsten Sätze in dem Text, den sie zusammenfasst, ordnet sie ein und gibt diese dann als Zusammenfassung wieder. Die abstrakte Zusammenfassung hingegen verwendet generative KI, um neue, originelle Sätze zu erstellen, die die wichtigsten Punkte des Textes zusammenfassen.

5. Es ist auch wichtig, die Struktur des Datentyps zu verstehen, den eine Funktion zurückgibt, damit Sie die Ausgabe in Ihren Abfragen richtig behandeln können. Um den von der Funktion `summarize_extractive()` zurückgegebenen Typ `azure_cognitive.sentence` zu prüfen, führen Sie Folgendes aus:

    ```sql
    \dT+ azure_cognitive.sentence
    ```

6. Die Ausgabe des obigen Befehls zeigt, dass der Typ `sentence` ein `tuple` ist. Um die Struktur dieses `tuple` zu untersuchen und die im `sentence` zusammengesetzten Typ enthaltenen Spalten zu überprüfen, führen Sie aus:

    ```sql
    \d+ azure_cognitive.sentence
    ```

    Die Ausgabe dieses Befehls sollte etwa so aussehen wie die folgende:

    ```sql
                            Composite type "azure_cognitive.sentence"
        Column  |     Type         | Collation | Nullable | Default | Storage  | Description 
    ------------+------------------+-----------+----------+---------+----------+-------------
     text       | text             |           |           |        | extended | 
     rank_score | double precision |           |           |        | plain    |
    ```

    Der `azure_cognitive.sentence` ist ein zusammengesetzter Typ, der den Text eines extraktiven Satzes und einen Rangwert für jeden Satz enthält, der angibt, wie relevant der Satz für das Hauptthema des Textes ist. Die Dokumentenzusammenfassung ordnet die extrahierten Sätze ein und Sie können festlegen, ob sie in der Reihenfolge ihres Erscheinens oder nach ihrem Rang zurückgegeben werden.

## Erstellen einer Zusammenfassungen für Immobilienbeschreibungen

In dieser Aufgabe verwenden Sie die Funktionen `summarize_extractive()` und `summarize_abstractive()`, um prägnante Zusammenfassungen in zwei Sätzen für Eigenschaftsbeschreibungen zu erstellen.

1. Nachdem Sie nun die Funktion `summarize_extractive()` und die von ihr zurückgegebene `sentiment_analysis_result` kennengelernt haben, lassen Sie uns die Funktion in die Tat umsetzen. Führen Sie die folgende einfache Abfrage aus, die eine Stimmungsanalyse für eine Handvoll Kommentare in der Tabelle `reviews` durchführt:

    ```sql
    SELECT
        id,
        name,
        description,
        azure_cognitive.summarize_extractive(description, 'en', 2) AS extractive_summary
    FROM listings
    WHERE id IN (1, 2);
    ```

    Vergleichen Sie die beiden Sätze im Feld `extractive_summary` in der Ausgabe mit dem Original `description` und stellen Sie fest, dass die Sätze nicht original sind, sondern aus `description` extrahiert wurden. Die numerischen Werte, die hinter jedem Satz aufgeführt sind, sind die vom Sprachendienst zugewiesene Punktzahl.

2. Als Nächstes führen Sie eine abstrakte Zusammenfassung der identischen Datensätze durch:

    ```sql
    SELECT
        id,
        name,
        description,
        azure_cognitive.summarize_abstractive(description, 'en', 2) AS abstractive_summary
    FROM listings
    WHERE id IN (1, 2);
    ```

    Die abstrakten Zusammenfassungsfunktionen der Erweiterung liefern eine einzigartige, natürlichsprachliche Zusammenfassung, die die Gesamtabsicht des Originaltextes wiedergibt.

    Wenn Sie eine ähnliche Fehlermeldung wie die folgende erhalten, haben Sie bei der Erstellung Ihrer Azure-Umgebung eine Region gewählt, die keine abstrakte Zusammenfassung unterstützt:

    ```bash
    ERROR: azure_cognitive.summarize_abstractive: InvalidRequest: Invalid Request.

    InvalidParameterValue: Job task: 'AbstractiveSummarization-task' failed with validation errors: ['Invalid Request.']

    InvalidRequest: Job task: 'AbstractiveSummarization-task' failed with validation error: Document abstractive summarization is not supported in the region Central US. The supported regions are North Europe, East US, West US, UK South, Southeast Asia.
    ```

    Um diesen Schritt und die übrigen Aufgaben mit der abstrakten Zusammenfassung durchführen zu können, müssen Sie einen neuen Azure KI Language-Service in einer der in der Fehlermeldung angegebenen unterstützten Regionen erstellen. Dieser Dienst kann in derselben Ressourcengruppe bereitgestellt werden, die Sie auch für andere Lab-Ressourcen verwendet haben. Alternativ können Sie die verbleibenden Aufgaben auch durch eine extraktive Zusammenfassung ersetzen, haben dann aber nicht den Vorteil, dass Sie die Ergebnisse der beiden verschiedenen Zusammenfassungsmethoden vergleichen können.

3. Führen Sie eine letzte Abfrage aus, um einen direkten Vergleich der beiden Zusammenfassungsmethoden durchzuführen:

    ```sql
    SELECT
        id,
        azure_cognitive.summarize_extractive(description, 'en', 2) AS extractive_summary,
        azure_cognitive.summarize_abstractive(description, 'en', 2) AS abstractive_summary
    FROM listings
    WHERE id IN (1, 2);
    ```

    Wenn Sie die erstellten Zusammenfassungen nebeneinander stellen, können Sie die Qualität der mit den einzelnen Methoden erstellten Zusammenfassungen leicht vergleichen. Für die Anwendung Margie's Travel ist die abstrakte Zusammenfassung die bessere Option. Sie liefert prägnante Zusammenfassungen, die qualitativ hochwertige Informationen auf natürliche und lesbare Weise vermitteln. Die extraktiven Zusammenfassungen enthalten zwar einige Details, sind aber unzusammenhängender und bieten weniger Wert als der ursprüngliche Inhalt, der durch die abstrakte Zusammenfassung erstellt wurde.

## Zusammenfassung der Ladenbeschreibung in der Datenbank

1. Führen Sie die folgende Abfrage aus, um die Tabelle `listings` zu ändern und eine neue Spalte `summary` hinzuzufügen:

    ```sql
    ALTER TABLE listings
    ADD COLUMN summary text;
    ```

2. Um mit generativer KI Zusammenfassungen für alle vorhandenen Eigenschaften in der Datenbank zu erstellen, ist es am effizientesten, die Beschreibungen in Stapeln zu senden, damit der Sprachdienst mehrere Datensätze gleichzeitig verarbeiten kann.

    ```sql
    WITH batch_cte AS (
        SELECT azure_cognitive.summarize_abstractive(ARRAY(SELECT description FROM listings ORDER BY id), 'en', batch_size => 25) AS summary
    ),
    summary_cte AS (
        SELECT
            ROW_NUMBER() OVER () AS id,
            ARRAY_TO_STRING(summary, ',') AS summary
        FROM batch_cte
    )
    UPDATE listings AS l
    SET summary = s.summary
    FROM summary_cte AS s
    WHERE l.id = s.id;
    ```

    Die Aktualisierungsanweisung verwendet zwei Common Table Expressions (CTEs), um die Daten zu bearbeiten, bevor die Tabelle `listings` mit den Zusammenfassungen aktualisiert wird. Der erste CTE (`batch_cte`) sendet alle `description`-Werte aus der Tabelle `listings` an den Sprachdienst, um abstrakte Zusammenfassungen zu erstellen. Dies geschieht in Stapeln von jeweils 25 Datensätzen. Der zweite CTE (`summary_cte`) verwendet die Ordnungsposition der von der Funktion `summarize_abstractive()` zurückgegebenen Zusammenfassungen, um jeder Zusammenfassung ein `id` zuzuweisen, das dem Datensatz entspricht, aus dem das `description` in der Tabelle `listings` stammt. Es verwendet auch die Funktion `ARRAY_TO_STRING`, um die generierten Zusammenfassungen aus dem Rückgabewert des Textarrays (`text[]`) zu ziehen und in eine einfache Zeichenkette zu konvertieren. Schließlich schreibt die Anweisung `UPDATE` die Zusammenfassung in die Tabelle `listings` für die zugehörige Auflistung.

3. Als letzten Schritt führen Sie eine Abfrage aus, um die in die Tabelle `listings` geschriebenen Zusammenfassungen anzuzeigen:

    ```sql
    SELECT
        id,
        name,
        description,
        summary
    FROM listings
    LIMIT 5;
    ```

## Generieren einer KI-Zusammenfassung von Bewertungen für ein Angebot

Bei der Margie's Travel App hilft die Anzeige einer Zusammenfassung aller Bewertungen für ein Objekt den Nutzern, sich schnell einen Überblick über die Gesamtheit der Bewertungen zu verschaffen.

1. Führen Sie die folgende Abfrage aus, die alle Bewertungen für einen Eintrag in einer einzigen Zeichenfolge zusammenfasst und dann eine abstrakte Zusammenfassung über diese Zeichenfolge erstellt:

    ```sql
    SELECT unnest(azure_cognitive.summarize_abstractive(reviews_combined, 'en')) AS review_summary
    FROM (
        -- Combine all reviews for a listing
        SELECT string_agg(comments, ' ') AS reviews_combined
        FROM reviews
        WHERE listing_id = 1
    );
    ```

## Bereinigung

Sobald Sie diese Übung abgeschlossen haben, löschen Sie die von Ihnen erstellten Azure-Ressourcen. Sie zahlen für die konfigurierte Kapazität, nicht dafür, wie viel die Datenbank genutzt wird. Folgen Sie diesen Anweisungen, um Ihre Ressourcengruppe und alle Ressourcen, die Sie für dieses Lab erstellt haben, zu löschen.

1. Öffnen Sie einen Webbrowser und navigieren Sie zum [Azure-Portal](https://portal.azure.com/). Wählen Sie auf der Startseite die Option **Ressourcengruppen** unter „Azure-Services“ aus.

    ![Screenshot der Ressourcengruppen, die im Azure-Portal unter Azure-Service durch ein rotes Feld hervorgehoben werden.](media/15-azure-portal-home-azure-services-resource-groups.png)

2. Geben Sie in das Suchfeld Filter für ein beliebiges Feld den Namen der Ressourcengruppe ein, die Sie für dieses Lab erstellt haben und wählen Sie dann Ihre Ressourcengruppe aus der Liste aus.

3. Wählen Sie auf der Seite **Übersicht** der Ressourcengruppe die Option **Ressourcengruppe löschen** aus.

    ![Screenshot des Übersichtsblatts der Ressourcengruppe, wobei die Schaltfläche „Ressourcengruppe löschen“ durch eine rote Box hervorgehoben ist.](media/15-resource-group-delete.png)

4. Geben Sie im Bestätigungsdialog zur Bestätigung den Namen der Ressourcengruppe ein, die Sie löschen möchten und wählen Sie dann **Löschen**.
