---
lab:
  title: Erkunden der Azure KI-Erweiterung
  module: Explore Generative AI with Azure Database for PostgreSQL
---

# Erkunden der Azure KI-Erweiterung

Als leitende Entwicklerin bzw. leitender Entwickler bei Margie's Travel haben Sie die Aufgabe, eine KI-gestützte Anwendung zu entwickeln, die Ihren Kundinnnen bzw. Kunden intelligente Empfehlungen für Mietobjekte gibt. Sie möchten mehr über die `azure_ai`-Erweiterung für Azure Database for PostgreSQL erfahren und wie sie Ihnen helfen kann, die Leistung der generativen KI (GenAI) in Ihre App zu integrieren. In dieser Übung erkunden Sie die `azure_ai`-Erweiterung und ihre Funktionalität, indem Sie sie in einer Azure Database for PostgreSQL – Flexibler Server-Datenbank installieren und ihre Möglichkeiten zur Integration von Azure KI- und ML-Services untersuchen.

## Vor der Installation

Sie benötigen ein [Azure-Abonnement](https://azure.microsoft.com/free) mit administrativen Rechten und müssen für den Azure OpenAI-Zugang in diesem Abonnement zugelassen sein. Wenn Sie Zugriff auf Azure OpenAI benötigen, bewerben Sie sich auf der Seite [Eingeschränkter Zugriff auf Azure OpenAI](https://learn.microsoft.com/legal/cognitive-services/openai/limited-access).

### Bereitstellen von Ressourcen in Ihrem Azure-Abonnement

Dieser Schritt führt Sie durch die Verwendung von Azure CLI-Befehlen aus der Azure Cloud Shell, um eine Ressourcengruppe zu erstellen und ein Bicep-Skript auszuführen, um die für diese Übung erforderlichen Azure-Services in Ihrem Azure-Abonnement bereitzustellen.

1. Öffnen Sie einen Webbrowser, und navigieren Sie zum [Azure-Portal](https://portal.azure.com/).

2. Wählen Sie das Symbol **Cloud Shell** in der Symbolleiste des Azure-Portals aus, um einen neuen [Cloud Shell](https://learn.microsoft.com/azure/cloud-shell/overview)-Bereich am unteren Rand Ihres Browserfensters zu öffnen.

    ![Screenshot der Azure-Symbolleiste mit dem Cloud Shell-Symbol, das durch eine rote Box hervorgehoben ist.](media/12-portal-toolbar-cloud-shell.png)

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

    Der letzte Befehl generiert nach dem Zufallsprinzip ein Kennwort für das PostgreSQL-Admin-Login. Kopieren Sie sie an einen sicheren Ort, um sie später bei der Verbindung mit Ihrem PostgreSQL – Flexibler Server zu verwenden.

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

1. Navigieren Sie im [Azure-Portal](https://portal.azure.com/) zu Ihrem neu erstellten Azure Database for PostgreSQL – Flexibler Server.

2. Wählen Sie im Ressourcenmenü unter **Einstellungen** die Option **Datenbanken** und dann **Verbinden** für die Datenbank `rentals`.

    ![Screenshot der Seite Azure-Datenbank für PostgreSQL-Datenbanken. Datenbanken und Verbinden für die Vermietungsdatenbank sind durch rote Boxen hervorgehoben.](media/12-postgresql-rentals-database-connect.png)

3. Geben Sie bei der Eingabeaufforderung „Kennwort für den Benutzer pgAdmin“ in der Cloud Shell das zufällig generierte Kennwort für die Anmeldung **pgAdmin** ein.

    Sobald Sie angemeldet sind, wird die Eingabeaufforderung `psql` für die Datenbank `rentals` angezeigt.

4. Im weiteren Verlauf dieser Übung arbeiten Sie weiterhin in der Cloud Shell. Daher kann es hilfreich sein, den Bereich in Ihrem Browserfenster zu erweitern, indem Sie die Schaltfläche **Maximieren** oben rechts im Bereich wählen.

    ![Screenshot des Azure Cloud Shell-Fensters mit der Schaltfläche „Maximieren“, die durch eine rote Box hervorgehoben ist.](media/12-azure-cloud-shell-pane-maximize.png)

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

## Überprüfen der in der Erweiterung `azure_ai` enthaltenen Objekte

Ein Blick auf die Objekte innerhalb der `azure_ai`-Erweiterung kann Ihnen helfen, ihre Möglichkeiten besser zu verstehen. In dieser Aufgabe untersuchen Sie die verschiedenen Schemata, benutzerdefinierten Funktionen (UDFs) und zusammengesetzten Typen, die der Datenbank durch die Erweiterung hinzugefügt wurden.

1. Bei der Arbeit mit `psql` in der Cloud Shell kann es hilfreich sein, die erweiterte Anzeige für Abfrageergebnisse zu aktivieren, da dies die Lesbarkeit der Ausgabe für nachfolgende Befehle verbessert. Führen Sie den folgenden Befehl aus, damit die erweiterte Anzeige automatisch angewendet wird.

    ```sql
    \x auto
    ```

2. Der [`\dx` Meta-Befehl](https://www.postgresql.org/docs/current/app-psql.html#APP-PSQL-META-COMMAND-DX-LC) wird verwendet, um Objekte aufzulisten, die in einer Erweiterung enthalten sind. Führen Sie den folgenden Befehl von der Eingabeaufforderung `psql` aus, um die Objekte in der Erweiterung `azure_ai` anzuzeigen. Möglicherweise müssen Sie die Leertaste drücken, um die vollständige Liste der Objekte anzuzeigen.

    ```psql
    \dx+ azure_ai
    ```

    Die Ausgabe des Meta-Befehls zeigt, dass die Erweiterung `azure_ai` vier Schemas, mehrere benutzerdefinierte Funktionen (UDFs), mehrere zusammengesetzte Typen in der Datenbank und die Tabelle `azure_ai.settings` erstellt. Abgesehen von den Schemas wird allen Objektnamen das Schema vorangestellt, zu dem sie gehören. Schemas werden verwendet, um verwandte Funktionen und Typen, die die Erweiterung hinzufügt, in Buckets zu gruppieren. In der folgenden Tabelle finden Sie eine Liste der Schemata, die durch die Erweiterung hinzugefügt wurden und eine kurze Beschreibung der einzelnen Schemata:

    | Schema      | Beschreibung                                              |
    | ----------------- | ------------------------------------------------------------------------------------------------------ |
    | `azure_ai`    | Das Hauptschema, in dem sich die Konfigurationstabelle und die UDFs für die Interaktion mit der Erweiterung befinden. |
    | `azure_openai`  | Enthält die UDFs, die das Aufrufen eines Azure OpenAI-Endpunkts ermöglichen.                    |
    | `azure_cognitive` | Bietet UDFs und zusammengesetzte Typen für die Integration der Datenbank mit Azure KI Services.     |
    | `azure_ml`    | Enthält die UDFs für die Integration von Azure Machine Learning (ML) Services.                |

### Erkunden des Azure KI-Schemas

Das `azure_ai`-Schema bietet den Rahmen für die direkte Interaktion mit Azure KI und ML Services aus Ihrer Datenbank. Es enthält Funktionen zum Einrichten von Verbindungen zu diesen Diensten und zum Abrufen dieser Verbindungen aus der Tabelle `settings`, die ebenfalls im selben Schema gehostet wird. Die Tabelle `settings` bietet sicheren Speicherplatz in der Datenbank für Endpunkte und Schlüssel, die mit Ihren Azure KI- und ML-Services verbunden sind.

1. Um die in einem Schema definierten Funktionen zu überprüfen, können Sie den [`\df`Meta-Befehl](https://www.postgresql.org/docs/current/app-psql.html#APP-PSQL-META-COMMAND-DF-LC) verwenden und dabei das Schema angeben, dessen Funktionen angezeigt werden sollen. Führen Sie das Folgende aus, um die Funktionen im Schema `azure_ai` anzuzeigen:

    ```sql
    \df azure_ai.*
    ```

    Die Ausgabe des Befehls sollte eine Tabelle ähnlich der folgenden sein:

    ```sql
                  List of functions
     Schema |  Name  | Result data type | Argument data types | Type 
    ----------+-------------+------------------+----------------------+------
     azure_ai | get_setting | text      | key text      | func
     azure_ai | set_setting | void      | key text, value text | func
     azure_ai | version  | text      |           | func
    ```

    Mit der Funktion `set_setting()` können Sie den Endpunkt und den Schlüssel Ihrer Azure KI- und ML-Services festlegen, damit sich die Erweiterung mit ihnen verbinden kann. Es akzeptiert einen **Schlüssel** und den **Wert**, der ihm zugewiesen werden soll. Die `azure_ai.get_setting()` Funktion bietet eine Möglichkeit zum Abrufen der Werte, die Sie mit der `set_setting()` Funktion festgelegt haben. Sie akzeptiert die **Taste** der Einstellung, die Sie ansehen möchten und gibt den ihr zugewiesenen Wert zurück. Für beide Methoden muss der Schlüssel eine der folgenden Sein:

    | Schlüssel | Beschreibung |
    | --- | ----------- |
    | `azure_openai.endpoint` | Ein unterstützter OpenAI-Endpunkt (z. B. <https://example.openai.azure.com>). |
    | `azure_openai.subscription_key` | Ein Abonnementschlüssel für eine Azure OpenAI-Ressource. |
    | `azure_cognitive.endpoint` | Ein unterstützter Azure KI Services Endpunkt (z. B. <https://example.cognitiveservices.azure.com>). |
    | `azure_cognitive.subscription_key` | Ein Abonnementschlüssel für eine Azure KI Services-Ressource. |
    | `azure_ml.scoring_endpoint` | Ein unterstützter Azure ML Scoring Endpunkt (z. B. <https://example.eastus2.inference.ml.azure.com/score>) |
    | `azure_ml.endpoint_key` | Ein Endpunktschlüssel für eine Azure ML-Bereitstellung. |

    > Wichtig
    >
    > Da die Verbindungsinformationen für Azure KI Services, einschließlich der API-Schlüssel, in einer Konfigurationstabelle in der Datenbank gespeichert werden, definiert die `azure_ai`-Erweiterung eine Rolle namens `azure_ai_settings_manager`, um sicherzustellen, dass diese Informationen geschützt und nur für Benutzende zugänglich sind, denen diese Rolle zugewiesen wurde. Diese Rolle ermöglicht das Lesen und Schreiben von Einstellungen im Zusammenhang mit der Erweiterung. Nur Mitglieder der Rolle `azure_ai_settings_manager` können die Funktionen `azure_ai.get_setting()` und `azure_ai.set_setting()` aufrufen. In einem Azure Database for PostgreSQL – Flexibler Server wird allen Admin-Benutzenden (denen die Rolle `azure_pg_admin` zugewiesen ist) auch die Rolle `azure_ai_settings_manager` zugewiesen.

2. Um zu demonstrieren, wie Sie die Funktionen `azure_ai.set_setting()` und `azure_ai.get_setting()` verwenden, konfigurieren Sie die Verbindung zu Ihrem Azure OpenAI-Konto. Minimieren Sie auf der gleichen Browser-Registerkarte, auf der Ihre Cloud Shell geöffnet ist, das Cloud Shell-Fenster oder stellen Sie es wieder her und navigieren Sie dann zu Ihrer Azure OpenAI-Ressource im [Azure-Portal](https://portal.azure.com/). Sobald Sie sich auf der Seite der Azure OpenAI-Ressource befinden, wählen Sie im Ressourcenmenü unter dem Abschnitt **Ressourcenverwaltung** die Option **Schlüssel und Endpunkt** aus und kopieren Sie dann Ihren Endpunkt und einen der verfügbaren Schlüssel.

    ![Es wird ein Screenshot der Seite Schlüssel und Endpunkte des Azure OpenAI-Services angezeigt, wobei die Schaltflächen KEY 1 und Endpunkt kopieren durch rote Boxen hervorgehoben sind.](media/12-azure-openai-keys-and-endpoints.png)

    Sie können `KEY 1` oder `KEY 2` verwenden. Wenn Sie jederzeit zwei Schlüssel zur Verfügung haben, können Sie die Schlüssel auf sichere Weise rotieren und neu generieren, ohne Dienstunterbrechungen zu verursachen.

3. Sobald Sie Ihren Endpunkt und Schlüssel haben, maximieren Sie das Cloud Shell-Fenster erneut und fügen Sie die Werte mit den folgenden Befehlen zur Konfigurationstabelle hinzu. Stellen Sie sicher, dass Sie die `{endpoint}` und `{api-key}` Token durch die Werte ersetzen, die Sie aus dem Azure-Portal kopiert haben.

    ```sql
    SELECT azure_ai.set_setting('azure_openai.endpoint', '{endpoint}');
    ```

    ```sql
    SELECT azure_ai.set_setting('azure_openai.subscription_key', '{api-key}');
    ```

4. Sie können die in die Tabelle `azure_ai.settings` geschriebenen Einstellungen mit der Funktion `azure_ai.get_setting()` in den folgenden Abfragen überprüfen:

    ```sql
    SELECT azure_ai.get_setting('azure_openai.endpoint');
    SELECT azure_ai.get_setting('azure_openai.subscription_key');
    ```

    Die Erweiterung `azure_ai` ist jetzt mit Ihrem Azure OpenAI-Konto verbunden.

### Überprüfen des Azure OpenAI Schemas

Das Schema `azure_openai` bietet die Möglichkeit, die Erstellung von Vektoreinbettungen von Textwerten in Ihre Datenbank mit Azure OpenAI zu integrieren. Mithilfe dieses Schemas können Sie [Einbettungen mit Azure OpenAI](https://learn.microsoft.com/azure/ai-services/openai/how-to/embeddings) direkt aus der Datenbank generieren, um Vektordarstellungen von Eingabetexten zu erstellen, die dann in Vektorähnlichkeitssuchen verwendet und von Modellen für maschinelles Lernen genutzt werden können. Das Schema enthält eine einzige Funktion, `create_embeddings()`, mit zwei Überladungen. Eine Überladung akzeptiert eine einzelne Eingabezeichenkette, die andere erwartet ein Array von Eingabezeichenketten.

1. Wie oben beschrieben, können Sie den [`\df` Meta-Befehl](https://www.postgresql.org/docs/current/app-psql.html#APP-PSQL-META-COMMAND-DF-LC) verwenden, um die Details der Funktionen im `azure_openai`-Schema anzuzeigen:

    ```sql
    \df azure_openai.*
    ```

    Die Ausgabe zeigt die beiden Überladungen der Funktion `azure_openai.create_embeddings()`, sodass Sie die Unterschiede zwischen den beiden Versionen der Funktion und den Typen, die sie zurückgeben, überprüfen können. Die Eigenschaft `Argument data types` in der Ausgabe zeigt die Liste der Argumente, die die beiden Funktionsüberladungen erwarten:

    | Argument    | Typ       | Standard | Beschreibung                                                          |
    | --------------- | ------------------ | ------- | ------------------------------------------------------------------------------------------------------------------------------ |
    | deployment_name | `text`      |    | Name der Bereitstellungsstelle in Azure OpenAI Studio, die das `text-embedding-ada-002`-Modell enthält.               |
    | input     | `text` oder `text[]` |    | Eingabetext (oder Array von Text), für den Einbettungen erstellt werden.                                |
    | batch_size   | `integer`     | 100  | Nur für die Überlast, die eine Eingabe von `text[]` erwartet. Gibt die Anzahl der gleichzeitig zu verarbeitenden Datensätze an.          |
    | timeout_ms   | `integer`     | 3600000 | Timeout in Millisekunden, nach dem der Vorgang beendet wird.                                 |
    | throw_on_error | `boolean`     | true  | Flag, das angibt, ob die Funktion beim Fehler eine Ausnahme auslösen soll, was zu einem Rollback der Umbruchtransaktionen führt. |
    | max_attempts  | `integer`     | 1   | Anzahl der Wiederholungen des Aufrufs an den Azure OpenAI-Dienst im Falle eines Fehlers.                     |
    | retry_delay_ms | `integer`     | 1.000  | Die Zeit (in Millisekunden), die gewartet werden muss, bevor versucht wird, den Azure OpenAI-Dienstendpunkt erneut aufzurufen.        |

2. Um ein vereinfachtes Beispiel für die Verwendung der Funktion zu geben, führen Sie die folgende Abfrage aus, die eine Vektoreinbettung für das Feld `description` in der Tabelle `listings` erstellt. Der Parameter `deployment_name` in der Funktion wird auf `embedding` gesetzt. Dies ist der Name der Bereitstellung des Modells `text-embedding-ada-002` in Ihrem Azure OpenAI Service (es wurde mit diesem Namen vom Bicep-Bereitstellungsskript erstellt):

    ```sql
    SELECT
      id,
      name,
      azure_openai.create_embeddings('embedding', description) AS vector
    FROM listings
    LIMIT 1;
    ```

    Die Ausgabe sieht in etwa wie folgt aus:

    ```sql
     id |      name       |              vector
    ----+-------------------------------+------------------------------------------------------------
      1 | Stylish One-Bedroom Apartment | {0.020068742,0.00022734122,0.0018286322,-0.0064167166,...}
    ```

    Der Kürze halber werden die Vektoreinbettungen in der obigen Ausgabe abgekürzt.

    [Embeddings](https://learn.microsoft.com/azure/postgresql/flexible-server/generative-ai-overview#embeddings) sind ein Konzept des maschinellen Lernens und der Verarbeitung natürlicher Sprache (NLP), bei dem Objekte wie Wörter, Dokumente oder Entitäten als [Vektoren](https://learn.microsoft.com/azure/postgresql/flexible-server/generative-ai-overview#vectors) in einem mehrdimensionalen Raum dargestellt werden. Mithilfe von Einbettungen können Modelle für maschinelles Lernen bewerten, wie eng zwei Informationen miteinander verbunden sind. Diese Technik identifiziert auf effiziente Weise Beziehungen und Ähnlichkeiten zwischen Daten und ermöglicht es Algorithmen, Muster zu erkennen und genaue Vorhersagen zu treffen.

    Mit der `azure_ai` Erweiterung können Sie Einbettungen für Eingabetext generieren. Damit die generierten Vektoren zusammen mit den übrigen Daten in der Datenbank gespeichert werden können, müssen Sie die `vector` Erweiterung installieren, indem Sie den Anweisungen in der Dokumentation [ Vektorunterstützung in Ihrer Datenbank aktivieren ](https://learn.microsoft.com/azure/postgresql/flexible-server/how-to-use-pgvector#enable-extension) folgen. Das liegt jedoch außerhalb des Rahmens dieser Übung.

### Untersuchen des Schemas azure_cognitive

Das `azure_cognitive`-Schema bietet den Rahmen für die direkte Interaktion mit Azure KI Services aus Ihrer Datenbank. Die Azure KI Services-Integrationen im Schema bieten eine Vielzahl von KI-Sprachfunktionen, auf die Sie direkt von der Datenbank aus zugreifen können. Zu den Funktionen gehören Stimmungsanalyse, Spracherkennung, Extraktion von Schlüsselwörtern, Erkennung von Entitäten, Textzusammenfassung und Übersetzung. Diese Funktionen werden über den [Azure KI Language-Service](https://learn.microsoft.com/azure/ai-services/language-service/overview) aktiviert.

1. Um alle in einem Schema definierten Funktionen zu überprüfen, können Sie den [`\df` Meta-Befehl](https://www.postgresql.org/docs/current/app-psql.html#APP-PSQL-META-COMMAND-DF-LC) verwenden, wie Sie es bereits getan haben. Um die Funktionen im Schema `azure_cognitive` anzuzeigen, führen Sie aus:

    ```sql
    \df azure_cognitive.*
    ```

2. In diesem Schema sind zahlreiche Funktionen definiert, sodass die Ausgabe des [`\df`Meta-Befehls](https://www.postgresql.org/docs/current/app-psql.html#APP-PSQL-META-COMMAND-DF-LC) schwer zu lesen sein kann. Daher ist es am besten, sie in kleinere Teile zu zerlegen. Führen Sie das Folgende aus, um nur die Funktion `analyze_sentiment()` zu betrachten:

    ```sql
    \df azure_cognitive.analyze_sentiment
    ```

    In der Ausgabe sehen Sie, dass die Funktion drei Überladungen hat, von denen eine eine einzelne Eingabezeichenkette akzeptiert und die beiden anderen Arrays mit Text erwarten. Die Ausgabe zeigt das Schema der Funktion, den Namen, den Ergebnisdatentyp und die Argumentdatentypen. Diese Informationen helfen Ihnen zu verstehen, wie Sie die Funktion verwenden können.

3. Wiederholen Sie den obigen Befehl und ersetzen Sie dabei den Funktionsnamen `analyze_sentiment` durch jeden der folgenden Funktionsnamen, um alle verfügbaren Funktionen im Schema zu prüfen:

   - `detect_language`
   - `extract_key_phrases`
   - `linked_entities`
   - `recognize_entities`
   - `recognize_pii_entities`
   - `summarize_abstractive`
   - `summarize_extractive`
   - `translate`

    Untersuchen Sie für jede Funktion die verschiedenen Formen der Funktion und ihre erwarteten Eingaben und resultierenden Datentypen.

4. Neben den Funktionen enthält das Schema `azure_cognitive` auch mehrere zusammengesetzte Typen, die als Rückgabedatentypen der verschiedenen Funktionen verwendet werden. Es ist unbedingt erforderlich, die Struktur des Datentyps zu verstehen, den eine Funktion zurückgibt, damit Sie die Ausgabe in Ihren Abfragen richtig behandeln können. Führen Sie zum Beispiel den folgenden Befehl aus, um den Typ `sentiment_analysis_result` zu überprüfen:

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
       Column  |   Type   | Collation | Nullable | Default | Storage | Description 
    ----------------+------------------+-----------+----------+---------+----------+-------------
     sentiment   | text      |     |     |    | extended | 
     positive_score | double precision |     |     |    | plain  | 
     neutral_score | double precision |     |     |    | plain  | 
     negative_score | double precision |     |     |    | plain  |
    ```

    Das `azure_cognitive.sentiment_analysis_result` ist ein zusammengesetzter Typ, der die Stimmungsvorhersagen des Eingabetextes enthält. Er enthält die Stimmung, die positiv, negativ, neutral oder gemischt sein kann, und die Bewertungen für positive, neutrale und negative Aspekte im Text. Die Ergebnisse werden als reale Zahlen zwischen 0 und 1 dargestellt. Beispielsweise ist die Stimmung in (neutral, 0,26, 0,64, 0,09) neutral, mit einem positiven Ergebnis von 0,26, neutral von 0,64 und negativ bei 0,09.

6. Wie bei den `azure_openai`-Funktionen müssen Sie, um mit der `azure_ai`-Erweiterung erfolgreich Azure KI Services aufrufen zu können, den Endpunkt und einen Schlüssel für Ihren Azure KI Language-Service angeben. Minimieren Sie das Fenster der Cloud Shell oder stellen Sie es wieder her, indem Sie dieselbe Browser-Registerkarte verwenden, auf der die Cloud Shell geöffnet ist, und navigieren Sie dann zu Ihrer Sprachdienst-Ressource im [Azure-Portal](https://portal.azure.com/). Wählen Sie im Ressourcenmenü unter dem Abschnitt **Ressourcenverwaltung** die Option **Schlüssel und Endpunkt**.

    ![Es wird ein Screenshot der Seite Schlüssel und Endpunkte des Azure-Sprachdienstes angezeigt. Die Schaltflächen KEY 1 und Endpunkt kopieren sind durch rote Felder hervorgehoben.](media/12-azure-language-service-keys-and-endpoints.png)

7. Kopieren Sie die Werte für den Endpunkt und den Zugriffsschlüssel und ersetzen Sie die `{endpoint}`- und `{api-key}`-Token durch die Werte, die Sie aus dem Azure-Portal kopiert haben. Maximieren Sie die Cloud Shell erneut und führen Sie die Befehle von der Eingabeaufforderung `psql` in der Cloud Shell aus, um Ihre Werte zur Konfigurationstabelle hinzuzufügen.

    ```sql
    SELECT azure_ai.set_setting('azure_cognitive.endpoint', '{endpoint}');
    ```

    ```sql
    SELECT azure_ai.set_setting('azure_cognitive.subscription_key', '{api-key}');
    ```

8. Führen Sie nun die folgende Abfrage aus, um die Stimmung bei einigen Bewertungen zu analysieren:

    ```sql
    SELECT
      id,
      comments,
      azure_cognitive.analyze_sentiment(comments, 'en') AS sentiment
    FROM reviews
    WHERE id IN (1, 3);
    ```

    Beachten Sie die `sentiment`-Werte in der Ausgabe, `(mixed,0.71,0.09,0.2)` und `(positive,0.99,0.01,0)`. Diese stellen die `sentiment_analysis_result` dar, die von der Funktion `analyze_sentiment()` in der obigen Abfrage zurückgegeben werden. Die Analyse wurde über das Feld `comments` in der Tabelle `reviews` durchgeführt.

## Überprüfen des Azure ML-Schemas

Mit dem `azure_ml`-Schema können sich Funktionen direkt von Ihrer Datenbank aus mit Azure ML-Services verbinden.

1. Um die in einem Schema definierten Funktionen zu überprüfen, können Sie den [`\df`Meta-Befehl](https://www.postgresql.org/docs/current/app-psql.html#APP-PSQL-META-COMMAND-DF-LC) verwenden. Um die Funktionen im Schema `azure_ml` anzuzeigen, führen Sie aus:

    ```sql
    \df azure_ml.*
    ```

    In der Ausgabe sehen Sie, dass in diesem Schema zwei Funktionen definiert sind, `azure_ml.inference()` und `azure_ml.invoke()`, deren Details unten angezeigt werden:

    ```sql
                  List of functions
    -----------------------------------------------------------------------------------------------------------
    Schema       | azure_ml
    Name        | inference
    Result data type  | jsonb
    Argument data types | input_data jsonb, deployment_name text DEFAULT NULL::text, timeout_ms integer DEFAULT NULL::integer, throw_on_error boolean DEFAULT true, max_attempts integer DEFAULT 1, retry_delay_ms integer DEFAULT 1000
    Type        | func
    ```

    Die Funktion `inference()` verwendet ein trainiertes maschinelles Lernmodell, um auf der Grundlage neuer, ungesehener Daten Ausgaben vorherzusagen oder zu erzeugen.

    Durch die Angabe eines Endpunkts und Schlüssels können Sie sich mit einem bereitgestellten Azure ML-Endpunkt verbinden, so wie Sie sich mit Ihren Azure OpenAI- und Azure KI Services-Endpunkten verbunden haben. Für die Interaktion mit Azure ML ist ein trainiertes und bereitgestelltes Modell erforderlich. Daher fällt diese Übung aus dem Rahmen und Sie werden diese Verbindung nicht herstellen, um sie hier auszuprobieren.

## Bereinigung

Sobald Sie diese Übung abgeschlossen haben, löschen Sie die von Ihnen erstellten Azure-Ressourcen. Sie zahlen für die konfigurierte Kapazität, nicht dafür, wie viel die Datenbank genutzt wird. Folgen Sie diesen Anweisungen, um Ihre Ressourcengruppe und alle Ressourcen, die Sie für dieses Lab erstellt haben, zu löschen.

1. Öffnen Sie einen Webbrowser und navigieren Sie zum [Azure-Portal](https://portal.azure.com/). Wählen Sie auf der Startseite die Option **Ressourcengruppen** unter „Azure-Services“ aus.

    ![Screenshot der Ressourcengruppen, die im Azure-Portal unter Azure-Service durch ein rotes Feld hervorgehoben werden.](media/12-azure-portal-home-azure-services-resource-groups.png)

2. Geben Sie in das Suchfeld Filter für ein beliebiges Feld den Namen der Ressourcengruppe ein, die Sie für dieses Lab erstellt haben und wählen Sie dann Ihre Ressourcengruppe aus der Liste aus.

3. Wählen Sie auf der Seite **Übersicht** der Ressourcengruppe die Option **Ressourcengruppe löschen** aus.

    ![Screenshot des Übersichtsblatts der Ressourcengruppe, wobei die Schaltfläche „Ressourcengruppe löschen“ durch eine rote Box hervorgehoben ist.](media/12-resource-group-delete.png)

4. Geben Sie im Bestätigungsdialog zur Bestätigung den Namen der Ressourcengruppe ein, die Sie löschen möchten und wählen Sie dann **Löschen**.
