---
lab:
  title: Generieren von Vektoreinbettungen mit Azure OpenAI
  module: Enable Semantic Search with Azure Database for PostgreSQL
---

# Generieren von Vektoreinbettungen mit Azure OpenAI

Zum Ausführen von semantischen Suchvorgängen müssen Sie zuerst Einbettungsvektoren aus einem Modell generieren, sie in einer Vektordatenbank speichern und dann die Einbettungen abfragen. Sie erstellen eine Datenbank, füllen sie mit Beispieldaten auf und führen semantische Suchvorgänge für diese Auflistungen aus.

Am Ende dieser Übung verfügen Sie über eine flexible Azure Database for PostgreSQL-Serverinstanz mit den aktivierten Erweiterungen `vector` und `azure_ai`. Sie erstellen Einbettungen für die `listings`-Tabelle des [Seattle Airbnb Open Data](https://www.kaggle.com/datasets/airbnb/seattle?select=listings.csv)-Datasets. Sie führen auch semantische Suchvorgänge für diese Auflistungen aus, indem Sie den Einbettungsvektor einer Abfrage generieren und eine Vektorkosinus-Entfernungssuche durchführen.

## Vor der Installation

Hierfür benötigen Sie ein [Azure-Abonnement](https://azure.microsoft.com/free) mit Administratorrechten.

### Bereitstellen von Ressourcen in Ihrem Azure-Abonnement

Dieser Schritt führt Sie durch die Verwendung von Azure CLI-Befehlen aus der Azure Cloud Shell, um eine Ressourcengruppe zu erstellen und ein Bicep-Skript auszuführen, um die Azure-Dienste bereitzustellen, die zum Abschließen dieser Übung in Ihrem Azure-Abonnement erforderlich sind.

> **Hinweis**: Wenn Sie mehrere Module in diesem Lernpfad absolvieren, können Sie die Azure-Umgebung gemeinsam nutzen. In diesem Fall müssen Sie diesen Schritt der Ressourcenbereitstellung nur einmal ausführen.

1. Öffnen Sie einen Webbrowser, und navigieren Sie zum [Azure-Portal](https://portal.azure.com/).

2. Wählen Sie das Symbol **Cloud Shell** in der Symbolleiste des Azure-Portals aus, um einen neuen [Cloud Shell](https://learn.microsoft.com/azure/cloud-shell/overview)-Bereich am unteren Rand Ihres Browserfensters zu öffnen.

    ![Screenshot der Azure-Symbolleiste mit dem Cloud Shell-Symbol, das durch einen roten Rahmen hervorgehoben ist.](media/13-portal-toolbar-cloud-shell.png)

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

    Das Bicep-Bereitstellungsskript stellt die Azure-Dienste bereit, die zum Abschließen dieser Übung in Ihrer Ressourcengruppe erforderlich sind. Die bereitgestellten Ressourcen umfassen eine flexible Azure-Datenbank für PostgreSQL-Server, Azure OpenAI und ein Azure KI Language-Dienst. Das Bicep-Skript führt auch einige Konfigurationsschritte aus, wie z. B. das Hinzufügen der `azure_ai`- und `vector`-Erweiterungen zur _Positivliste_ des PostgreSQL-Servers (über den `azure.extensions` Server-Parameter), das Erstellen einer Datenbank mit dem Namen `rentals` auf dem Server und das Hinzufügen einer Bereitstellung mit dem Namen `embedding` unter Verwendung des `text-embedding-ada-002`-Modells zu Ihrem Azure OpenAI Service. Beachten Sie, dass die Bicep-Datei von allen Modulen in diesem Lernpfad gemeinsam genutzt wird, sodass Sie möglicherweise nur einige der bereitgestellten Ressourcen in einigen Übungen verwenden.

    Die Bereitstellung dauert in der Regel mehrere Minuten. Sie können es von der Cloud Shell aus überwachen oder zur Seite **Bereitstellungen** für die oben erstellte Ressourcengruppe navigieren und dort den Bereitstellungsfortschritt beobachten.

8. Schließen Sie den Cloud Shell-Bereich, sobald die Ressourcenbereitstellung abgeschlossen ist.
 
### Problembehandlung bei der Bereitstellung

Beim Ausführen des Bicep-Bereitstellungsskripts treten möglicherweise einige Fehler auf.

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

1. Navigieren Sie im [Azure-Portal](https://portal.azure.com/) zu Ihrer neu erstellten Azure Database for PostgreSQL – Flexibler Server.

2. Wählen Sie im Ressourcenmenü unter **Einstellungen** die Option **Datenbanken** und wählen Sie **Verbinden** für die Datenbank `rentals`.

    ![Screenshot der Azure Database for PostgreSQL-Datenbankseite. Datenbanken und Verbinden für die Mietdatenbank werden in roten Feldern hervorgehoben.](media/13-postgresql-rentals-database-connect.png)

3. Geben Sie beim Prompt „Kennwort für den Benutzer pgAdmin“ in der Cloud Shell das zufällig generierte Kennwort für die **pgAdmin**-Anmeldung ein.

    Sobald Sie angemeldet sind, wird der `psql`-Prompt für die `rentals`-Datenbank angezeigt.

4. Im weiteren Verlauf dieser Übung arbeiten Sie weiterhin in der Cloud Shell. Daher kann es hilfreich sein, den Bereich in Ihrem Browserfenster zu erweitern, indem Sie die Schaltfläche **Maximieren** oben rechts im Bereich wählen.

    ![Screenshot des Azure Cloud Shell-Fensters mit der Schaltfläche „Maximieren“, die durch eine rote Box hervorgehoben ist.](media/13-azure-cloud-shell-pane-maximize.png)

## Setup: Konfigurieren von Erweiterungen

Um Vektoren zu speichern und abzufragen und um Einbettungen zu erzeugen, müssen Sie zwei Erweiterungen für einen flexiblen Azure Database for PostgreSQL-Server zulassen und aktivieren: `vector` und `azure_ai`.

1. Um beide Erweiterungen zuzulassen, fügen Sie `vector` und `azure_ai` zum Server-Parameter `azure.extensions` hinzu, wie in den Anweisungen unter [Wie man PostgreSQL-Erweiterungen verwendet](https://learn.microsoft.com/en-us/azure/postgresql/flexible-server/concepts-extensions#how-to-use-postgresql-extensions) beschrieben.

2. Führen Sie den folgenden SQL-Befehl aus, um die Erweiterung `vector` zu aktivieren. Ausführliche Anweisungen finden Sie unter [Aktivierung und Verwendung von `pgvector` auf Azure Database for PostgreSQL – Flexibler Server](https://learn.microsoft.com/en-us/azure/postgresql/flexible-server/how-to-use-pgvector#enable-extension).

    ```sql
    CREATE EXTENSION vector;
    ```

3. Um die Erweiterung `azure_ai` zu aktivieren, führen Sie den folgenden SQL-Befehl aus. Sie benötigen den Endpunkt und den API-Schlüssel für die Azure OpenAI-Ressource. Ausführliche Anweisungen finden Sie unter [Aktivieren der `azure_ai`-Erweiterung](https://learn.microsoft.com/en-us/azure/postgresql/flexible-server/generative-ai-azure-overview#enable-the-azure_ai-extension).

    ```sql
    CREATE EXTENSION azure_ai;
    SELECT azure_ai.set_setting('azure_openai.endpoint', 'https://<endpoint>.openai.azure.com');
    SELECT azure_ai.set_setting('azure_openai.subscription_key', '<API Key>');
    ```

## Füllen der Datenbank mit Beispieldaten

Bevor Sie sich mit der `azure_ai`-Erweiterung befassen, fügen Sie der `rentals`-Datenbank einige Tabellen hinzu und füllen sie mit Beispieldaten, damit Sie bei der Überprüfung der Funktionalität der Erweiterung mit Informationen arbeiten können.

1. Führen Sie die folgenden Befehle aus, um die Tabellen `listings` und `reviews` zu erstellen, in denen die Daten für die Auflistung der Mietobjekte und die Bewertungen der Kundschaft gespeichert werden:

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

3. Führen Sie schließlich den folgenden Befehl aus, um die Bewertungen der Kundschaft in die Tabelle `reviews` zu laden:

    ```sql
    \COPY reviews FROM 'mslearn-postgresql/Allfiles/Labs/Shared/reviews.csv' CSV HEADER
    ```

    Die Befehlsausgabe sollte `COPY 354` lauten, was bedeutet, dass 354 Zeilen aus der CSV-Datei in die Tabelle geschrieben wurden.

Um Ihre Probendaten zurückzusetzen, können Sie `DROP TABLE listings` ausführen und diese Schritte wiederholen.

## Erstellen und Speichern von Einbettungsvektoren

Da wir nun einige Beispieldaten haben, ist es an der Zeit, die Einbettungsvektoren zu generieren und zu speichern. Die `azure_ai`-Erweiterung erleichtert das Aufrufen der Azure OpenAI-Einbettungs-API.

1. Fügen Sie die Einbettungsvektorspalte hinzu.

    Das Modell `text-embedding-ada-002` ist so konfiguriert, dass es 1.536 Dimensionen zurückgibt, verwenden Sie also diese Größe für die Vektorspalte.

    ```sql
    ALTER TABLE listings ADD COLUMN listing_vector vector(1536);
    ```

1. Erzeugen Sie einen Einbettungsvektor für die Beschreibung jedes Eintrags, indem Sie Azure OpenAI über die benutzerdefinierte Funktion „create_embeddings“ aufrufen, die von der Erweiterung „azure_ai“ implementiert wird:

    ```sql
    UPDATE listings
    SET listing_vector = azure_openai.create_embeddings('embedding', description, max_attempts => 5, retry_delay_ms => 500)
    WHERE listing_vector IS NULL;
    ```

    Beachten Sie, dass dies je nach verfügbarem Kontingent mehrere Minuten dauern kann.

1. Sehen Sie sich einen Beispielvektor an, indem Sie diese Abfrage ausführen:

    ```sql
    SELECT listing_vector FROM listings LIMIT 1;
    ```

    Sie erhalten ein ähnliches Ergebnis, aber mit 1.536 Vektorspalten:

    ```sql
    postgres=> SELECT listing_vector FROM listings LIMIT 1;
    -[ RECORD 1 ]--+------ ...
    listing_vector | [-0.0018742813,-0.04530062,0.055145424, ... ]
    ```

## Ausführen einer semantischen Suchabfrage

Nachdem Sie nun Daten aufgelistet haben, die mit Einbettungsvektoren erweitert wurden, ist es an der Zeit, eine semantische Suchabfrage auszuführen. Dazu ermitteln Sie den Einbettungsvektor für die Abfragezeichenfolge und führen dann eine Kosinussuche durch, um die Einträge zu finden, deren Beschreibungen der Anfrage semantisch am ähnlichsten sind.

1. Generieren Sie die Einbettung für die Abfragezeichenfolge.

    ```sql
    SELECT azure_openai.create_embeddings('embedding', 'bright natural light');
    ```

    Das Ergebnis könnte in etwa wie folgt aussehen:

    ```sql
    -[ RECORD 1 ]-----+-- ...
    create_embeddings | {-0.0020871465,-0.002830255,0.030923981, ...}
    ```

1. Verwenden Sie die Einbettung in einer Kosinussuche (`<=>` steht für den Vorgang der Kosinusdistanz), um die 10 der Abfrage am ähnlichsten erscheinenden Einträge zu finden.

    ```sql
    SELECT id, name FROM listings ORDER BY listing_vector <=> azure_openai.create_embeddings('embedding', 'bright natural light')::vector LIMIT 10;
    ```

    Sie erhalten dann ein ähnliches Ergebnis wie dieses. Die Ergebnisse können variieren, da Einbettungsvektoren nicht garantiert deterministisch sind:

    ```sql
        id    |                name                
    ----------+-------------------------------------
     6796336  | A duplex near U district!
     7635966  | Modern Capitol Hill Apartment
     7011200  | Bright 1 bd w deck. Great location
     8099917  | The Ravenna Apartment
     10211928 | Charming Ravenna Bungalow
     692671   | Sun Drenched Ballard Apartment
     7574864  | Modern Greenlake Getaway
     7807658  | Top Floor Corner Apt-Downtown View
     10265391 | Art filled, quiet, walkable Seattle
     5578943  | Madrona Studio w/Private Entrance
    ```

1. Sie können die `description`-Spalte auch projizieren, um den Text der übereinstimmenden Zeilen zu lesen, deren Beschreibungen semantisch ähnlich waren. Diese Abfrage gibt beispielsweise die beste Übereinstimmung zurück:

    ```sql
    SELECT id, description FROM listings ORDER BY listing_vector <=> azure_openai.create_embeddings('embedding', 'bright natural light')::vector LIMIT 1;
    ```

    Damit wird etwa Folgendes ausgegeben: 

    ```sql
       id    | description
    ---------+------------
     6796336 | This is a great place to live for summer because you get a lot of sunlight at the living room. A huge living room space with comfy couch and one ceiling window and glass windows around the living room.
    ```

Um die semantische Suche intuitiv zu verstehen, beachten Sie, dass die Beschreibung die Begriffe „hell“ oder „natürlich“ nicht enthält. Aber es hebt „Sommer“ und „Sonnenlicht“, „Fenster“ und ein „Deckenfenster“ hervor.

## Arbeit überprüfen

Nachdem Sie die obigen Schritte durchgeführt haben, enthält die `listings`-Tabelle Beispieldaten von [Seattle Airbnb Open Data](https://www.kaggle.com/datasets/airbnb/seattle/data?select=listings.csv) auf Kaggle. Die Auflistungen wurden durch Einbettungsvektoren erweitert, um semantische Suchvorgänge auszuführen.

1. Bestätigen Sie, dass die Listings-Tabelle vier Spalten hat: `id`, `name`, `description` und `listing_vector`.

    ```sql
    \d listings
    ```

    Es sollte etwa so aussehen:

    ```sql
                            Table "public.listings"
          Column    |         Type           | Collation | Nullable | Default 
    ----------------+------------------------+-----------+----------+---------
      id            | integer                |           | not null | 
      name          | character varying(255) |           | not null | 
      description   | text                   |           | not null | 
     listing_vector | vector(1536)           |           |          | 
     Indexes:
        "listings_pkey" PRIMARY KEY, btree (id)
    ```

1. Vergewissern Sie sich, dass mindestens eine Zeile eine „listing_vector-Spalte aufgefüllt hat.

    ```sql
    SELECT COUNT(*) > 0 FROM listings WHERE listing_vector IS NOT NULL;
    ```

    Das Ergebnis muss ein `t`, welches „true“ (wahr) bedeutet, anzeigen. Ein Hinweis darauf, dass mindestens eine Zeile mit Einbettungen der entsprechenden Beschreibungsspalte vorhanden ist:

    ```sql
    ?column? 
    ----------
    t
    (1 row)
    ```

    Bestätigen Sie, dass der Einbettungsvektor 1.536 Dimensionen hat:

    ```sql
    SELECT vector_dims(listing_vector) FROM listings WHERE listing_vector IS NOT NULL LIMIT 1;
    ```

    Ergebnis:

    ```sql
    vector_dims 
    -------------
            1536
    (1 row)
    ```

1. Vergewissern Sie sich, dass semantische Suchvorgänge Ergebnisse zurückgeben.

    Verwenden Sie die Einbettung in eine Kosinussuche und rufen Sie die 10 häufigsten Auflistungen der Abfrage ab.

    ```sql
    SELECT id, name FROM listings ORDER BY listing_vector <=> azure_openai.create_embeddings('embedding', 'bright natural light')::vector LIMIT 10;
    ```

    Sie erhalten ein Ergebnis wie folgt, je nachdem, welche Zeilen Einbettungsvektoren zugewiesen wurden:

    ```sql
     id |                name                
    --------+-------------------------------------
     315120 | Large, comfy, light, garden studio
     429453 | Sunny Bedroom #2 w/View: Wallingfrd
     17951  | West Seattle, The Starlight Studio
     48848  | green suite seattle - dog friendly
     116221 | Modern, Light-Filled Fremont Flat
     206781 | Bright & Spacious Studio
     356566 | Sunny Bedroom w/View: Wallingford
     9419   | Golden Sun vintage warm/sunny
     136480 | Bright Cheery Room in Seattle House
     180939 | Central District Green GardenStudio
    ```

## Bereinigung

Nachdem Sie diese Übung abgeschlossen haben, löschen Sie die Azure-Ressourcen, die Sie erstellt haben. Die Abrechnung basiert auf der konfigurierten Kapazität und nicht auf der Verwendung der Datenbank. Befolgen Sie diese Anweisungen, um Ihre Ressourcengruppe und alle Ressourcen zu löschen, die Sie für diese Übung erstellt haben.

1. Öffnen Sie einen Webbrowser und navigieren Sie zum [Azure-Portal](https://portal.azure.com/). Wählen Sie auf der Startseite **Ressourcengruppen** unter Azure-Dienste aus.

    ![Screenshot von Ressourcengruppen, die durch ein rotes Feld unter Azure-Diensten im Azure-Portal hervorgehoben sind.](media/13-azure-portal-home-azure-services-resource-groups.png)

2. Geben Sie in das Suchfeld für ein beliebiges Feld den Namen der Ressourcengruppe ein, die Sie für dieses Lab erstellt haben, und wählen Sie dann Ihre Ressourcengruppe aus der Liste aus.

3. Wählen Sie auf der Seite **Übersicht** der Ressourcengruppe die Option **Ressourcengruppe löschen** aus.

    ![Screenshot des Übersichtsblatts der Ressourcengruppe mit der Schaltfläche „Ressourcengruppe löschen“, die durch ein rotes Kästchen hervorgehoben ist.](media/13-resource-group-delete.png)

4. Geben Sie im Bestätigungsdialogfeld den Namen der Ressourcengruppe ein, die Sie löschen möchten, und wählen Sie dann **Löschen** aus.
