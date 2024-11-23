---
lab:
  title: Übersetzen von Text mit Azure KI Übersetzer
  module: Translate Text using the Azure AI Translator and Azure Database for PostgreSQL
---

# Übersetzen von Text mit Azure KI Übersetzer

Als leitender Entwickler von Margie's Travel wurden Sie gebeten, bei einer Internationalisierungsmaßnahme mitzuwirken. Heute sind alle Mietangebote für den Kurzzeitvermietungsservice des Unternehmens auf Englisch. Sie möchten diese Angebote ohne großen Entwicklungsaufwand in eine Vielzahl von Sprachen übersetzen. Alle Ihre Daten werden in einem Azure Database for PostgreSQL – Flexibler Server gehostet und Sie möchten Azure KI Services für die Übersetzung verwenden.

In dieser Übung übersetzen Sie mithilfe des Azure KI Übersetzer-Services über eine Azure Database for PostgreSQL – Flexibler Server-Datenbank englischsprachigen Text in verschiedene Sprachen.

## Vor der Installation

Sie benötigen ein [Azure-Abonnement](https://azure.microsoft.com/free) mit Administratorrechten.

### Bereitstellen von Ressourcen in Ihrem Azure-Abonnement

Dieser Schritt führt Sie durch die Verwendung von Azure CLI-Befehlen aus der Azure Cloud Shell, um eine Ressourcengruppe zu erstellen und ein Bicep-Skript auszuführen, um die Azure-Services, die für die Durchführung dieser Übung erforderlich sind, in Ihrem Azure-Abonnement bereitzustellen.

1. Öffnen Sie einen Webbrowser, und navigieren Sie zum [Azure-Portal](https://portal.azure.com/).

2. Wählen Sie das Symbol **Cloud Shell** in der Symbolleiste des Azure-Portals aus, um einen neuen [Cloud Shell](https://learn.microsoft.com/azure/cloud-shell/overview)-Bereich am unteren Rand Ihres Browserfensters zu öffnen.

    ![Screenshot der Azure-Symbolleiste mit dem Cloud Shell-Symbol, das durch eine rote Box hervorgehoben ist.](media/11-portal-toolbar-cloud-shell.png)

    Wählen Sie bei Aufforderung die erforderlichen Optionen aus, um eine *Bash*-Shell zu öffnen. Wenn Sie zuvor eine *PowerShell*-Konsole verwendet haben, wechseln Sie zu einer *Bash*-Shell.

3. Geben Sie an der Cloud Shell-Eingabeaufforderung Folgendes ein, um das GitHub-Repository mit den Übungsressourcen zu klonen:

    ```bash
    git clone https://github.com/MicrosoftLearning/mslearn-postgresql.git
    ```

    Wenn Sie dieses GitHub-Repositorium bereits in einem früheren Modul geklont haben, steht es Ihnen weiterhin zur Verfügung und Sie erhalten möglicherweise die folgende Fehlermeldung:

    ```bash
    fatal: destination path 'mslearn-postgresql' already exists and is not an empty directory.
    ```

    Wenn Sie diese Meldung erhalten, können Sie bedenkenlos mit dem nächsten Schritt fortfahren.

4. Als Nächstes führen Sie drei Befehle aus, um Variablen zu definieren und so die redundante Eingabe zu reduzieren, wenn Sie Azure-CLI-Befehle zum Erstellen von Azure-Ressourcen verwenden. Die Variablen stehen für den Namen, den Sie Ihrer Ressourcengruppe zuweisen (`RG_NAME`), für die Azure-Region (`REGION`), in der die Ressourcen bereitgestellt werden und für ein zufällig generiertes Kennwort für den PostgreSQL-Administrator-Login (`ADMIN_PASSWORD`).

    Im ersten Befehl ist die Region, die der entsprechenden Variablen zugewiesen ist, `eastus`, aber Sie können sie auch durch einen Ort Ihrer Wahl ersetzen. Wenn Sie jedoch die Standardeinstellung ersetzen, müssen Sie eine andere [Azure-Region auswählen, die die abstrakte Zusammenfassung unterstützt](https://learn.microsoft.com/azure/ai-services/language-service/summarization/region-support), um sicherzustellen, dass Sie alle Aufgaben in den Modulen in diesem Lernpfad erledigen können.

    ```bash
    REGION=eastus
    ```

    Mit dem folgenden Befehl weisen Sie den Namen für die Ressourcengruppe zu, die alle in dieser Übung verwendeten Ressourcen enthalten wird. Der Name der Ressourcengruppe, der der entsprechenden Variablen zugewiesen ist, lautet `rg-learn-postgresql-ai-$REGION`, wobei `$REGION` der Ort ist, den Sie oben angegeben haben. Sie können den Namen jedoch auch in einen anderen Namen für die Ressourcengruppe ändern, der Ihren Wünschen entspricht.

    ```bash
    RG_NAME=rg-learn-postgresql-ai-$REGION
    ```

    Der letzte Befehl generiert nach dem Zufallsprinzip ein Kennwort für das PostgreSQL-Admin-Login. Stellen Sie sicher, dass Sie es an einen sicheren Ort kopieren, um es später für die Verbindung zu Ihrem flexiblen PostgreSQL-Server zu verwenden.

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
    az deployment group create --resource-group $RG_NAME --template-file "mslearn-postgresql/Allfiles/Labs/Shared/deploy-translate.bicep" --parameters restore=false adminLogin=pgAdmin adminLoginPassword=$ADMIN_PASSWORD
    ```

    Das Bicep-Bereitstellungsskript stellt die für diese Übung erforderlichen Azure-Service in Ihrer Ressourcengruppe bereit. Zu den bereitgestellten Ressourcen gehören ein Azure Database for PostgreSQL – Flexibler Server und ein Azure KI Übersetzer-Service. Das Bicep-Skript führt auch einige Konfigurationsschritte aus, z. B. das Hinzufügen der Erweiterungen `azure_ai` und `vector` zur _Positivliste_ des PostgreSQL-Servers (über den Serverparameter azure.extensions) und das Erstellen einer Datenbank namens `rentals` auf dem Server. **Beachten Sie, dass sich die Bicep-Datei von den anderen Modulen in diesem Lernpfad unterscheidet.**

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

1. Navigieren Sie im [Azure-Portal](https://portal.azure.com/) zu Ihrem neu erstellten Azure Database for PostgreSQL – Flexibler Server.

2. Wählen Sie im Ressourcenmenü unter **Einstellungen** die Option **Datenbanken** und dann **Verbinden** für die Datenbank `rentals`.

    ![Screenshot der Seite Azure-Datenbank für PostgreSQL-Datenbanken. Datenbanken und Verbinden für die Vermietungsdatenbank sind durch rote Boxen hervorgehoben.](media/17-postgresql-rentals-database-connect.png)

3. Geben Sie bei der Eingabeaufforderung „Kennwort für den Benutzer pgAdmin“ in der Cloud Shell das zufällig generierte Kennwort für die Anmeldung **pgAdmin** ein.

    Sobald Sie angemeldet sind, wird die Eingabeaufforderung `psql` für die Datenbank `rentals` angezeigt.

4. Im weiteren Verlauf dieser Übung arbeiten Sie weiterhin in der Cloud Shell. Daher kann es hilfreich sein, den Bereich in Ihrem Browserfenster zu erweitern, indem Sie die Schaltfläche **Maximieren** oben rechts im Bereich wählen.

    ![Screenshot des Azure Cloud Shell-Fensters mit der Schaltfläche „Maximieren“, die durch eine rote Box hervorgehoben ist.](media/17-azure-cloud-shell-pane-maximize.png)

## Auffüllen der Datenbank mit Listendaten

Sie müssen über englischsprachige Listendaten verfügen, um sie übersetzen zu können. Wenn Sie die Tabelle `listings` in der Datenbank `rentals` nicht in einem früheren Modul erstellt haben, folgen Sie diesen Anweisungen, um sie zu erstellen.

1. Führen Sie die folgenden Befehle aus, um die Tabelle `listings` zu erstellen, in der die Daten der Mietobjekte gespeichert werden:

    ```sql
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

2. Als Nächstes verwenden Sie den Befehl `COPY`, um Daten aus CSV-Dateien in jede Tabelle zu laden, die Sie oben erstellt haben. Beginnen Sie, indem Sie den folgenden Befehl ausführen, um die Tabelle `listings` aufzufüllen:

    ```sql
    \COPY listings FROM 'mslearn-postgresql/Allfiles/Labs/Shared/listings.csv' CSV HEADER
    ```

    Die Befehlsausgabe sollte `COPY 50` lauten, was bedeutet, dass 50 Zeilen aus der CSV-Datei in die Tabelle geschrieben wurden.

## Erstellen zusätzlicher Tabellen für die Übersetzung

Sie haben die `listings`-Daten, benötigen aber zwei zusätzliche Tabellen, um die Übersetzung durchzuführen.

1. Führen Sie die folgenden Befehle aus, um die Tabellen `languages` und `listing_translations` zu erstellen.

    ```sql
    CREATE TABLE languages (
        code VARCHAR(7) NOT NULL PRIMARY KEY
    );
    ```

    ```sql
    CREATE TABLE listing_translations(
        id INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
        listing_id INT,
        language_code VARCHAR(7),
        description TEXT
    );
    ```

2. Fügen Sie dann eine Zeile pro Sprache für die Übersetzung ein. In diesem Fall werden Sie Zeilen für fünf Sprachen erstellen: Deutsch, Chinesisch (vereinfacht), Hindi, Ungarisch und Suaheli.

    ```sql
    INSERT INTO languages(code)
    VALUES
        ('de'),
        ('zh-Hans'),
        ('hi'),
        ('hu'),
        ('sw');
    ```

    Die Befehlsausgabe sollte `INSERT 0 5` lauten, was bedeutet, dass Sie fünf neue Zeilen in die Tabelle eingefügt haben.

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

    Bevor eine Erweiterung in Azure Database for PostgreSQL – Flexibler Server installiert und verwendet werden kann, muss sie zur _Positivliste_ des Servers hinzugefügt werden, wie in [Wie man PostgreSQL-Erweiterungen verwendet](https://learn.microsoft.com/azure/postgresql/flexible-server/concepts-extensions#how-to-use-postgresql-extensions) beschrieben.

2. Jetzt können Sie die `azure_ai`-Erweiterung mit dem Befehl [CREATE EXTENSION](https://www.postgresql.org/docs/current/sql-createextension.html) installieren.

    ```sql
    CREATE EXTENSION IF NOT EXISTS azure_ai;
    ```

    `CREATE EXTENSION` lädt eine neue Erweiterung in die Datenbank, indem die zugehörige Skriptdatei ausgeführt wird. Dieses Skript erstellt normalerweise neue SQL-Objekte wie Funktionen, Datentypen und Schemata. Es wird ein Fehler ausgegeben, wenn eine Erweiterung mit demselben Namen bereits existiert. Durch Hinzufügen von `IF NOT EXISTS` kann der Befehl ausgeführt werden, ohne dass eine Fehlermeldung ausgegeben wird, wenn er bereits installiert ist.

3. Sie müssen dann die Funktion `azure_ai.set_setting()` verwenden, um die Verbindung zu Ihrem Azure KI Übersetzer-Service zu konfigurieren. Minimieren Sie auf der gleichen Browser-Registerkarte, auf der Ihre Cloud Shell geöffnet ist, oder stellen Sie das Cloud Shell-Fenster wieder her und navigieren Sie dann zu Ihrer Azure KI Übersetzer-Ressource im [Azure Portal](https://portal.azure.com/). Sobald Sie sich auf der Ressourcenseite von Azure KI Übersetzer befinden, wählen Sie im Ressourcenmenü unter dem Abschnitt **Ressourcenverwaltung** die Option **Schlüssel und Endpunkt** und kopieren dann einen der verfügbaren Schlüssel, Ihre Region und Ihren Dokumentübersetzungs-Endpunkt.

    ![Sie sehen einen Screenshot der Seite Schlüssel und Endpunkte des Azure KI Übersetzer-Services. Die Schaltflächen zum Kopieren der Endpunkte KEY 1, Region und Dokumentübersetzung sind durch rote Boxen hervorgehoben.](media/18-azure-ai-translator-keys-and-endpoint.png)

    Sie können `KEY 1` oder `KEY 2` verwenden. Wenn Sie jederzeit zwei Schlüssel zur Verfügung haben, können Sie die Schlüssel auf sichere Weise rotieren und neu generieren, ohne Dienstunterbrechungen zu verursachen.

4. Konfigurieren Sie die `azure_cognitive`-Einstellungen so, dass sie auf Ihren KI-Übersetzer-Endpunkt, Ihren Abonnementschlüssel und Ihre Region verweisen. Der Wert für `azure_cognitive.endpoint` ist die URL für die Dokumentübersetzung Ihres Dienstes. Der Wert für `azure_cognitive.subscription_key` ist Schlüssel 1 oder Schlüssel 2. Der Wert für `azure_cognitive.region` wird die Region der Azure KI Übersetzer-Instanz sein.

    ```sql
    SELECT azure_ai.set_setting('azure_cognitive.endpoint','https://<YOUR_ENDPOINT>.cognitiveservices.azure.com/');
    SELECT azure_ai.set_setting('azure_cognitive.subscription_key', '<YOUR_KEY>');
    SELECT azure_ai.set_setting('azure_cognitive.region', '<YOUR_REGION>');
    ```

## Erstellen einer gespeicherten Prozedur zum Übersetzen von Listingdaten

Um die Tabelle mit den Sprachübersetzungen aufzufüllen, erstellen Sie eine gespeicherte Prozedur, mit der Sie Daten in Stapeln laden können.

1. Führen Sie den folgenden Befehl an der Eingabeaufforderung `psql` aus, um eine neue gespeicherte Prozedur namens `translate_listing_descriptions` zu erstellen.

    ```sql
    CREATE OR REPLACE PROCEDURE translate_listing_descriptions(max_num_listings INT DEFAULT 10)
    LANGUAGE plpgsql
    AS $$
    BEGIN
        WITH batch_to_load(id, description) AS
        (
            SELECT id, description
            FROM listings l
            WHERE NOT EXISTS (SELECT * FROM listing_translations ll WHERE ll.listing_id = l.id)
            LIMIT max_num_listings
        )
        INSERT INTO listing_translations(listing_id, language_code, description)
        SELECT b.id, l.code, (unnest(tr.translations)).TEXT
        FROM batch_to_load b
            CROSS JOIN languages l
            CROSS JOIN LATERAL azure_cognitive.translate(b.description, l.code) tr;
    END;
    $$;
    ```

    Diese gespeicherte Prozedur lädt einen Stapel von 5 Datensätzen, übersetzt die Beschreibung in jede von Ihnen ausgewählte Sprache und fügt die übersetzten Beschreibungen in die Tabelle `listing_translations` ein.

2. Führen Sie die gespeicherte Prozedur mit dem folgenden SQL-Befehl aus:

    ```sql
    CALL translate_listing_descriptions(10);
    ```

    Dieser Aufruf benötigt etwa eine Sekunde pro Mietangebot, um es in fünf Sprachen zu übersetzen. Jeder Durchlauf sollte also etwa 10 Sekunden dauern. Die Befehlsausgabe sollte `CALL` lauten, was bedeutet, dass der Aufruf der gespeicherten Prozedur erfolgreich war.

3. Rufen Sie die gespeicherte Prozedur vier weitere Male auf, sodass Sie diese Prozedur fünfmal aufgerufen haben. Das erzeugt Übersetzungen für jeden Eintrag in der Tabelle.

4. Führen Sie das folgende Skript aus, um die Anzahl der aufgeführten Übersetzungen zu ermitteln.

    ```sql
    SELECT COUNT(*) FROM listing_translations;
    ```

    Der Aufruf sollte einen Wert von 250 zurückgeben, was bedeutet, dass jeder Eintrag in fünf Sprachen übersetzt wurde. Sie können die Daten weiter analysieren, indem Sie die Tabelle `listing_translations` abfragen.

## Erstellen einer Prozedur zum Hinzufügen eines neuen Eintrags mit Übersetzungen

Sie haben eine gespeicherte Prozedur, um bestehende Angebote zu übersetzen, aber Ihre Internationalisierungspläne erfordern auch die Übersetzung neuer Angebote, sobald diese eingegeben werden. Zu diesem Zweck erstellen Sie eine weitere gespeicherte Prozedur.

1. Führen Sie den folgenden Befehl an der Eingabeaufforderung `psql` aus, um eine neue gespeicherte Prozedur namens `add_listing` zu erstellen.

    ```sql
    CREATE OR REPLACE PROCEDURE add_listing(id INT, name VARCHAR(255), description TEXT)
    LANGUAGE plpgsql
    AS $$
    DECLARE
    listing_id INT;
    BEGIN
        INSERT INTO listings(id, name, description)
        VALUES(id, name, description);

        INSERT INTO listing_translations(listing_id, language_code, description)
        SELECT id, l.code, (unnest(tr.translations)).TEXT
        FROM languages l
            CROSS JOIN LATERAL azure_cognitive.translate(description, l.code) tr;
    END;
    $$;
    ```

    Diese gespeicherte Prozedur wird eine Zeile in die Tabelle `listings` einfügen. Dann übersetzt es die Beschreibung für jede Sprache in der `languages`-Tabelle und fügt diese Übersetzungen in die `listing_translations`-Tabelle ein.

2. Führen Sie die gespeicherte Prozedur mit dem folgenden SQL-Befehl aus:

    ```sql
    CALL add_listing(51, 'A Beautiful Home', 'This is a beautiful home in a great location.');
    ```

    Die Befehlsausgabe sollte `CALL` lauten, was bedeutet, dass der Aufruf der gespeicherten Prozedur erfolgreich war.

3. Führen Sie das folgende Skript aus, um die Übersetzungen für Ihren neuen Eintrag zu erhalten.

    ```sql
    SELECT l.id, l.name, l.description, lt.language_code, lt.description AS translated_description
    FROM listing_translations lt
        INNER JOIN listings l ON lt.listing_id = l.id
    WHERE l.name = 'A Beautiful Home';
    ```

    Der Aufruf sollte fünf Zeilen mit Werten ähnlich der folgenden Tabelle zurückgeben.

    ```sql
     id  | listing_id | language_code |                    description                     
    -----+------------+---------------+------------------------------------------------------
     126 |          2 | de            | Dies ist ein schönes Haus in einer großartigen Lage.
     127 |          2 | zh-Hans       | 这是一个美丽的家，地理位置优越。
     128 |          2 | hi            | यह एक महान स्थान में एक सुंदर घर है।
     129 |          2 | hu            | Ez egy gyönyörű otthon egy nagyszerű helyen.
     130 |          2 | sw            | Hii ni nyumba nzuri katika eneo kubwa.
    ```

## Bereinigung

Sobald Sie diese Übung abgeschlossen haben, löschen Sie die von Ihnen erstellten Azure-Ressourcen. Sie zahlen für die konfigurierte Kapazität, nicht dafür, wie viel die Datenbank genutzt wird. Folgen Sie diesen Anweisungen, um Ihre Ressourcengruppe und alle Ressourcen, die Sie für dieses Lab erstellt haben, zu löschen.

1. Öffnen Sie einen Webbrowser und navigieren Sie zum [Azure-Portal](https://portal.azure.com/). Wählen Sie auf der Startseite die Option **Ressourcengruppen** unter „Azure-Services“ aus.

    ![Screenshot der Ressourcengruppen, die im Azure-Portal unter Azure-Service durch ein rotes Feld hervorgehoben werden.](media/17-azure-portal-home-azure-services-resource-groups.png)

2. Geben Sie in das Suchfeld Filter für ein beliebiges Feld den Namen der Ressourcengruppe ein, die Sie für dieses Lab erstellt haben und wählen Sie dann die Ressourcengruppe aus der Liste aus.

3. Wählen Sie auf der Seite **Übersicht** der Ressourcengruppe die Option **Ressourcengruppe löschen** aus.

    ![Screenshot des Übersichtsblatts der Ressourcengruppe, wobei die Schaltfläche „Ressourcengruppe löschen“ durch eine rote Box hervorgehoben ist.](media/17-resource-group-delete.png)

4. Geben Sie im Bestätigungsdialog zur Bestätigung den Namen der Ressourcengruppe ein, die Sie löschen möchten und wählen Sie dann **Löschen**.
