---
lab:
  title: Extrahieren Sie Erkenntnisse mit der Azure KI Language
  module: Extract insights using the Azure AI Language service with Azure Database for PostgreSQL
---

# Extrahieren Sie Erkenntnisse mit dem Azure KI Language-Service mit Azure Database for PostgreSQL

Erinnern Sie sich daran, dass das Unternehmen Markttrends analysieren möchte, wie z. B. die beliebtesten Phrasen oder Orte. Das Team beabsichtigt außerdem, den Schutz von personenbezogenen Informationen (PII) zu verbessern. Die aktuellen Daten werden in einem Azure Database for PostgreSQL – Flexibler Server gespeichert. Das Projektbudget ist klein, daher ist es wichtig, die Vorlaufkosten und die laufenden Kosten für die Pflege von Schlüsselwörtern und Tags zu minimieren. Die Entwicklerinnen und Entwickler sind sich der vielen Formen bewusst, die PII annehmen können und ziehen eine kostengünstige, überprüfte Lösung einem internen Abgleich mit regulären Ausdrücken vor.

Sie integrieren die Datenbank mithilfe der `azure_ai`-Erweiterung in Azure KI Language-Services. Die Erweiterung bietet benutzerdefinierte SQL-Funktions-APIs für mehrere Azure Cognitive Service-APIs, darunter:

- oder Entitätserkennung
- Entitätserkennung
- PII-Anerkennung

Auf diese Weise kann das Data-Science-Team schnell Daten über die Beliebtheit von Angeboten vergleichen, um Markttrends zu ermitteln. Außerdem erhalten Anwendungsentwickler damit einen PII-sicheren Text, den sie in Situationen präsentieren können, in denen kein Zugriff erforderlich ist. Die Speicherung identifizierter Entitäten ermöglicht eine menschliche Überprüfung im Falle einer Anfrage oder einer falsch positiven PII-Erkennung (wenn Sie etwas für PII halten, das es nicht ist).

Am Ende haben Sie vier neue Spalten in der `listings`-Tabelle mit extrahierten Erkenntnissen:

- `key_phrases`
- `recognized_entities`
- `pii_safe_description`
- `pii_entities`

## Vor der Installation

Sie benötigen ein [Azure-Abonnement](https://azure.microsoft.com/free) mit administrativen Rechten und müssen für den Azure OpenAI-Zugang in diesem Abonnement zugelassen sein. Wenn Sie Zugriff auf Azure OpenAI benötigen, bewerben Sie sich auf der Seite [Eingeschränkter Zugriff auf Azure OpenAI](https://learn.microsoft.com/legal/cognitive-services/openai/limited-access).

### Bereitstellen von Ressourcen in Ihrem Azure-Abonnement

Dieser Schritt führt Sie durch die Verwendung von Azure CLI-Befehlen aus der Azure Cloud Shell, um eine Ressourcengruppe zu erstellen und ein Bicep-Skript auszuführen, um die für diese Übung erforderlichen Azure-Services in Ihrem Azure-Abonnement bereitzustellen.

1. Öffnen Sie einen Webbrowser, und navigieren Sie zum [Azure-Portal](https://portal.azure.com/).

2. Wählen Sie das Symbol **Cloud Shell** in der Symbolleiste des Azure-Portals aus, um einen neuen [Cloud Shell](https://learn.microsoft.com/azure/cloud-shell/overview)-Bereich am unteren Rand Ihres Browserfensters zu öffnen.

    ![Screenshot der Azure-Symbolleiste mit dem Cloud Shell-Symbol, das durch eine rote Box hervorgehoben ist.](media/17-portal-toolbar-cloud-shell.png)

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

    Das Bicep-Bereitstellungsskript stellt die für diese Übung erforderlichen Azure-Service in Ihrer Ressourcengruppe bereit. Zu den bereitgestellten Ressourcen gehören eine Azure Database for PostgreSQL – Flexibler Server, Azure OpenAI und ein Azure KI Language-Service. Das Bicep-Skript führt auch einige Konfigurationsschritte aus, wie z. B. das Hinzufügen der `azure_ai`- und `vector`-Erweiterungen zur _Positivliste_ des PostgreSQL-Servers (über den `azure.extensions`-Server-Parameter), das Erstellen einer Datenbank mit dem Namen `rentals` auf dem Server und das Hinzufügen einer Bereitstellung mit dem Namen `embedding` unter Verwendung des `text-embedding-ada-002`-Modells zu Ihrem Azure OpenAI Service. Beachten Sie, dass die Bicep-Datei von allen Modulen in diesem Lernpfad gemeinsam genutzt wird, sodass Sie möglicherweise nur einige der bereitgestellten Ressourcen in einigen Übungen verwenden.

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

In dieser Aufgabe stellen Sie eine Verbindung zur `rentals`-Datenbank auf Ihrem Azure Database for PostgreSQL-Server her, indem Sie das [psql Befehlszeilendienstprogramm](https://www.postgresql.org/docs/current/app-psql.html) aus der [Azure Cloud Shell](https://learn.microsoft.com/azure/cloud-shell/overview) verwenden.

1. Navigieren Sie im [Azure-Portal](https://portal.azure.com/) zu Ihrer neu erstellten Azure-Datenbank für PostgreSQL – Flexibler Server.

2. Wählen Sie im Ressourcenmenü unter **Einstellungen** die Option **Datenbanken** und dann **Verbinden** für die Datenbank `rentals`.

    ![Screenshot der Seite Azure-Datenbank für PostgreSQL-Datenbanken. Datenbanken und Verbinden für die Vermietungsdatenbank sind durch rote Boxen hervorgehoben.](media/17-postgresql-rentals-database-connect.png)

3. Geben Sie bei der Eingabeaufforderung „Kennwort für den Benutzer pgAdmin“ in der Cloud Shell das zufällig generierte Kennwort für die Anmeldung **pgAdmin** ein.

    Sobald Sie angemeldet sind, wird die Eingabeaufforderung `psql` für die Datenbank `rentals` angezeigt.

4. Im weiteren Verlauf dieser Übung arbeiten Sie weiterhin in der Cloud Shell. Daher kann es hilfreich sein, den Bereich in Ihrem Browserfenster zu erweitern, indem Sie die Schaltfläche **Maximieren** oben rechts im Bereich wählen.

    ![Screenshot des Azure Cloud Shell-Fensters mit der Schaltfläche „Maximieren“, die durch eine rote Box hervorgehoben ist.](media/17-azure-cloud-shell-pane-maximize.png)

## Setup: Erweiterungen konfigurieren

Um Vektoren zu speichern und abzufragen und um Einbettungen zu erzeugen, müssen Sie zwei Erweiterungen für Azure Database for PostgreSQL – Flexibler Server zulassen und aktivieren: `vector` und `azure_ai`.

1. Um beide Erweiterungen zuzulassen, fügen Sie `vector` und `azure_ai` zum Serverparameter `azure.extensions` hinzu, wie in [Wie man PostgreSQL-Erweiterungen verwendet](https://learn.microsoft.com/en-us/azure/postgresql/flexible-server/concepts-extensions#how-to-use-postgresql-extensions) beschrieben.

2. Führen Sie den folgenden SQL-Befehl aus, um die Erweiterung `vector` zu aktivieren. Detaillierte Anweisungen finden Sie unter [Aktivierung und Verwendung von `pgvector` auf Azure Database for PostgreSQL – Flexibler Server](https://learn.microsoft.com/en-us/azure/postgresql/flexible-server/how-to-use-pgvector#enable-extension).

    ```sql
    CREATE EXTENSION vector;
    ```

3. Um die Erweiterung `azure_ai` zu aktivieren, führen Sie den folgenden SQL-Befehl aus. Sie benötigen den Endpunkt und den API-Schlüssel für die ***Azure OpenAI*** Ressource. Detaillierte Anweisungen finden Sie unter [Aktivieren Sie die `azure_ai`-Erweiterung](https://learn.microsoft.com/en-us/azure/postgresql/flexible-server/generative-ai-azure-overview#enable-the-azure_ai-extension).

    ```sql
    CREATE EXTENSION azure_ai;
    ```

    ```sql
    SELECT azure_ai.set_setting('azure_openai.endpoint', 'https://<endpoint>.openai.azure.com');
    ```

    ```sql
    SELECT azure_ai.set_setting('azure_openai.subscription_key', '<API Key>');
    ```

4. Um Ihre ***Azure KI Language***-Services mithilfe der `azure_ai`-Erweiterung erfolgreich aufrufen zu können, müssen Sie der Erweiterung ihren Endpunkt und Schlüssel zur Verfügung stellen. Navigieren Sie in derselben Browser-Registerkarte, in der die Cloud Shell geöffnet ist, zu Ihrer Sprachdienst-Ressource im [Azure-Portal](https://portal.azure.com/) und wählen Sie im linken Navigationsmenü unter **Ressourcenverwaltung** den Punkt **Schlüssel und Endpunkt** aus.

5. Kopieren Sie die Werte für den Endpunkt und den Zugriffsschlüssel. Ersetzen Sie dann in den folgenden Befehlen die `{endpoint}` und `{api-key}` Token durch die Werte, die Sie aus dem Azure-Portal kopiert haben. Führen Sie die Befehle von der Eingabeaufforderung `psql` in der Cloud Shell aus, um Ihre Werte der `azure_ai.settings`-Tabelle hinzuzufügen.

    ```sql
    SELECT azure_ai.set_setting('azure_cognitive.endpoint', '{endpoint}');
    ```

    ```sql
    SELECT azure_ai.set_setting('azure_cognitive.subscription_key', '{api-key}');
    ```

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

Um Ihre Probendaten zurückzusetzen, können Sie `DROP TABLE listings` ausführen und diese Schritte wiederholen.

## Schlüsselbegriffe extrahieren

1. Die Schlüsselphrasen werden als `text[]` extrahiert, wie die Funktion `pg_typeof` zeigt:

    ```sql
    SELECT pg_typeof(azure_cognitive.extract_key_phrases('The food was delicious and the staff were wonderful.', 'en-us'));
    ```

    Erstellen Sie eine Spalte, die die Schlüsselergebnisse enthält.

    ```sql
    ALTER TABLE listings ADD COLUMN key_phrases text[];
    ```

1. Füllen Sie die Spalte in Stapeln aus. Je nach Quote möchten Sie vielleicht den Wert `LIMIT` anpassen. *Fühlen Sie sich frei, den Befehl so oft auszuführen, wie Sie möchten*; Sie brauchen für diese Übung nicht alle Zeilen auszufüllen.

    ```sql
    UPDATE listings
    SET key_phrases = azure_cognitive.extract_key_phrases(description)
    FROM (SELECT id FROM listings WHERE key_phrases IS NULL ORDER BY id LIMIT 100) subset
    WHERE listings.id = subset.id;
    ```

1. Abfrage der Einträge nach Schlüsselwörtern:

    ```sql
    SELECT id, name FROM listings WHERE 'closet' = ANY(key_phrases);
    ```

    Je nachdem, welche Einträge mit Schlüsselwörtern gefüllt sind, erhalten Sie die folgenden Ergebnisse:

    ```sql
       id    |                name                
    ---------+-------------------------------------
      931154 | Met Tower in Belltown! MT2
      931758 | Hottest Downtown Address, Pool! MT2
     1084046 | Near Pike Place & Space Needle! MT2
     1084084 | The Best of the Best, Seattle! MT2
    ```

## Erkennung benannter Entitäten

1. Die Entitäten werden als `azure_cognitive.entity[]` extrahiert, wie die Funktion `pg_typeof` zeigt:

    ```sql
    SELECT pg_typeof(azure_cognitive.recognize_entities('For more information, see Cognitive Services Compliance and Privacy notes.', 'en-us'));
    ```

    Erstellen Sie eine Spalte, die die Schlüsselergebnisse enthält.

    ```sql
    ALTER TABLE listings ADD COLUMN entities azure_cognitive.entity[];
    ```

2. Füllen Sie die Spalte in Stapeln aus. Dieser Vorgang kann mehrere Minuten dauern. Vielleicht möchten Sie den Wert `LIMIT` je nach Quote anpassen oder mit Teilergebnissen schneller zurückkehren. *Fühlen Sie sich frei, den Befehl so oft auszuführen, wie Sie möchten*; Sie brauchen für diese Übung nicht alle Zeilen auszufüllen.

    ```sql
    UPDATE listings
    SET entities = azure_cognitive.recognize_entities(description, 'en-us')
    FROM (SELECT id FROM listings WHERE entities IS NULL ORDER BY id LIMIT 500) subset
    WHERE listings.id = subset.id;
    ```

3. Sie können jetzt alle Inserate abfragen, um Immobilien mit Kellern zu finden:

    ```sql
    SELECT id, name
    FROM listings, unnest(entities) e
    WHERE e.text LIKE '%basement%'
    LIMIT 10;
    ```

    Das Ergebnis sieht ungefähr so aus:

    ```sql
       id    |                name                
    ---------+-------------------------------------
      430610 | 3br/3ba. modern, roof deck, garage
      430610 | 3br/3ba. modern, roof deck, garage
     1214306 | Private Bed/bath in Home: green (A)
       74328 | Spacious Designer Condo
      938785 | Best Ocean Views By Pike Place! PA1
       23430 | 1 Bedroom Modern Water View Condo
      828298 | 2 Bedroom Sparkling City Oasis
      338043 | large modern unit & fab location
      872152 | Luxurious Local Lifestyle 2Bd/2+Bth
      116221 | Modern, Light-Filled Fremont Flat
    ```

## PII-Anerkennung

1. Die Entitäten werden als `azure_cognitive.pii_entity_recognition_result` extrahiert, wie die Funktion `pg_typeof` zeigt:

    ```sql
    SELECT pg_typeof(azure_cognitive.recognize_pii_entities('For more information, see Cognitive Services Compliance and Privacy notes.', 'en-us'));
    ```

    Dieser Wert ist ein zusammengesetzter Typ, der geschwärzten Text und ein Array von PII-Entitäten enthält, wie verifiziert von:

    ```sql
    \d azure_cognitive.pii_entity_recognition_result
    ```

    Was gedruckt wird:

    ```sql
         Composite type "azure_cognitive.pii_entity_recognition_result"
         Column    |           Type           | Collation | Nullable | Default 
    ---------------+--------------------------+-----------+----------+---------
     redacted_text | text                     |           |          | 
     entities      | azure_cognitive.entity[] |           |          | 
    ```

    Erstellen Sie eine Spalte, die den geschwärzten Text enthält und eine weitere für die erkannten Entitäten:

    ```sql
    ALTER TABLE listings ADD COLUMN description_pii_safe text;
    ALTER TABLE listings ADD COLUMN pii_entities azure_cognitive.entity[];
    ```

2. Füllen Sie die Spalte in Stapeln aus. Dieser Vorgang kann mehrere Minuten dauern. Vielleicht möchten Sie den Wert `LIMIT` je nach Quote anpassen oder mit Teilergebnissen schneller zurückkehren. *Fühlen Sie sich frei, den Befehl so oft auszuführen, wie Sie möchten*; Sie brauchen für diese Übung nicht alle Zeilen auszufüllen.

    ```sql
    UPDATE listings
    SET
     description_pii_safe = pii.redacted_text,
     pii_entities = pii.entities
    FROM (SELECT id, description FROM listings WHERE description_pii_safe IS NULL OR pii_entities IS NULL ORDER BY id LIMIT 100) subset,
    LATERAL azure_cognitive.recognize_pii_entities(subset.description, 'en-us') as pii
    WHERE listings.id = subset.id;
    ```

3. Sie können jetzt Angebotsbeschreibungen anzeigen, bei denen alle potenziellen PII geschwärzt sind:

    ```sql
    SELECT description_pii_safe
    FROM listings
    WHERE description_pii_safe IS NOT NULL
    LIMIT 1;
    ```

    Zeigt Folgendes an:

    ```sql
    A lovely stone-tiled room with kitchenette. New full mattress futon bed. Fridge, microwave, kettle for coffee and tea. Separate entrance into book-lined mudroom. Large bathroom with Jacuzzi (shared occasionally with ***** to do laundry). Stone-tiled, radiant heated floor, 300 sq ft room with 3 large windows. The bed is queen-sized futon and has a full-sized mattress with topper. Bedside tables and reading lights on both sides. Also large leather couch with cushions. Kitchenette is off the side wing of the main room and has a microwave, and fridge, and an electric kettle for making coffee or tea. Kitchen table with two chairs to use for meals or as desk. Extra high-speed WiFi is also provided. Access to English Garden. The Ballard Neighborhood is a great place to visit: *10 minute walk to downtown Ballard with fabulous bars and restaurants, great ****** farmers market, nice three-screen cinema, and much more. *5 minute walk to the Ballard Locks, where ships enter and exit Puget Sound
    ```

4. Sie können auch die in der PII erkannten Entitäten identifizieren, indem Sie zum Beispiel die gleiche Liste wie oben verwenden:

    ```sql
    SELECT entities
    FROM listings
    WHERE entities IS NOT NULL
    LIMIT 1;
    ```

    Zeigt Folgendes an:

    ```sql
                            pii_entities                        
    -------------------------------------------------------------
    {"(hosts,PersonType,\"\",0.93)","(Sunday,DateTime,Date,1)"}
    ```

## Arbeit überprüfen

Stellen Sie sicher, dass die extrahierten Schlüsselbegriffe, erkannten Entitäten und PII ausgefüllt wurden:

1. Überprüfen der Schlüsselbegriffe:

    ```sql
    SELECT COUNT(*) FROM listings WHERE key_phrases IS NOT NULL;
    ```

    Je nachdem, wie viele Batches Sie durchgeführt haben, sollten Sie etwa Folgendes sehen:

    ```sql
    count 
    -------
     100
    ```

2. Überprüfen der anerkannten Entitäten:

    ```sql
    SELECT COUNT(*) FROM listings WHERE entities IS NOT NULL;
    ```

    Die Ausgabe sollte in etwa wie folgt aussehen:

    ```sql
    count 
    -------
     500
    ```

3. Überprüfen der geschwärzten PII:

    ```sql
    SELECT COUNT(*) FROM listings WHERE description_pii_safe IS NOT NULL;
    ```

    Wenn Sie eine einzelne Charge von 100 geladen haben, sollten Sie fogendes sehen:

    ```sql
    count 
    -------
     100
    ```

    Sie können überprüfen, bei wie vielen Einträgen PII entdeckt wurden:

    ```sql
    SELECT COUNT(*) FROM listings WHERE description != description_pii_safe;
    ```

    Die Ausgabe sollte in etwa wie folgt aussehen:

    ```sql
    count 
    -------
        87
    ```

4. Überprüfen Sie die gefundenen PII-Entitäten: Gemäß den obigen Angaben sollten wir 13 ohne ein leeres PII-Array haben.

    ```sql
    SELECT COUNT(*) FROM listings WHERE pii_entities IS NULL AND description_pii_safe IS NOT NULL;
    ```

    Ergebnis:

    ```sql
    count 
    -------
        13
    ```

## Bereinigung

Sobald Sie diese Übung abgeschlossen haben, löschen Sie die von Ihnen erstellten Azure-Ressourcen. Sie zahlen für die konfigurierte Kapazität, nicht dafür, wie viel die Datenbank genutzt wird. Folgen Sie diesen Anweisungen, um Ihre Ressourcengruppe und alle Ressourcen, die Sie für dieses Lab erstellt haben, zu löschen.

1. Öffnen Sie einen Webbrowser und navigieren Sie zum [Azure-Portal](https://portal.azure.com/). Wählen Sie auf der Startseite die Option **Ressourcengruppen** unter „Azure-Services“ aus.

    ![Screenshot der Ressourcengruppen, die im Azure-Portal unter Azure-Service durch ein rotes Feld hervorgehoben werden.](media/17-azure-portal-home-azure-services-resource-groups.png)

2. Geben Sie in das Suchfeld Filter für ein beliebiges Feld den Namen der Ressourcengruppe ein, die Sie für dieses Lab erstellt haben und wählen Sie dann Ihre Ressourcengruppe aus der Liste aus.

3. Wählen Sie auf der Seite **Übersicht** der Ressourcengruppe die Option **Ressourcengruppe löschen** aus.

    ![Screenshot des Übersichtsblatts der Ressourcengruppe, wobei die Schaltfläche „Ressourcengruppe löschen“ durch eine rote Box hervorgehoben ist.](media/17-resource-group-delete.png)

4. Geben Sie im Bestätigungsdialog zur Bestätigung den Namen der Ressourcengruppe ein, die Sie löschen möchten und wählen Sie dann **Löschen**.
