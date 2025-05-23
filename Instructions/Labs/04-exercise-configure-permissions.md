---
lab:
  title: Konfigurieren von Berechtigungen in Azure Database for PostgreSQL
  module: Secure Azure Database for PostgreSQL
---

# Konfigurieren von Berechtigungen in Azure Database for PostgreSQL

In den Übungen dieses Labs weisen Sie rollenbasierte Zugriffssteuerungsrollen (RBAC) zu, um den Zugriff auf Azure Database for PostgreSQL-Ressourcen zu steuern, und „PostgreSQL GRANTS“, um den Zugriff auf Datenbankvorgänge zu steuern.

## Vor der Installation

Für diese Übung benötigen Sie ein eigenes Azure-Abonnement. Wenn Sie nicht über ein Azure-Abonnement verfügen, können Sie ein [kostenloses Azure-Testkonto](https://azure.microsoft.com/free) erstellen.

Zusätzlich muss Folgendes auf Ihrem Computer installiert sein:

- Visual Studio Code.
- Postgres Visual Studio Code-Erweiterung von Microsoft
- Azure-Befehlszeilenschnittstelle.
- Git.

## Erstellen der Übungsumgebung

In diesen und späteren Übungen verwenden Sie ein Bicep-Skript, um die Azure Database for PostgreSQL – Flexibler Server und andere Ressourcen in Ihrem Azure-Abonnement bereitzustellen. Die Bicep-Skripts befinden sich im `/Allfiles/Labs/Shared`-Ordner des zuvor geklonten Github Repositorys.

### Laden Sie die Visual Studio Code- und die PostgreSQL-Erweiterung herunter, und installieren Sie sie.

Wenn Sie Visual Studio Code nicht installiert haben:

1. Navigieren Sie in einem Browser zu [Herunterladen von Visual Studio Code](https://code.visualstudio.com/download) , und wählen Sie die entsprechende Version für Ihr Betriebssystem aus.

1. Folgen Sie den Anweisungen zur Installation für Ihr Betriebssystem.

1. Öffnen Sie Visual Studio Code.

1. Klicken Sie im linken Menü auf **Erweiterungen**, um den Bereich „Erweiterungen“ anzuzeigen.

1. Geben Sie **Subnetze** in die Suchleiste ein. Das Symbol der PostgreSQL-Erweiterung für Visual Studio Code wird angezeigt. Stellen Sie sicher, dass Sie die Microsoft-Option auswählen.

1. Wählen Sie **Installieren** aus. Die Erweiterung wird installiert.

### Herunterladen und Installieren von Azure CLI und Git

Wenn Sie Azure CLI oder Git nicht installiert haben:

1. Navigieren Sie in einem Browser zu [Azure CLI installieren](https://learn.microsoft.com/cli/azure/install-azure-cli) und befolgen Sie die Anweisungen für Ihr Betriebssystem.

1. Navigieren Sie in einem Browser zu [Git herunterladen und installieren](https://git-scm.com/downloads) und folgen Sie den Anweisungen für Ihr Betriebssystem.

### Herunterladen der Übungsdateien

Wenn Sie bereits das GitHub-Repository geklont haben, das die Übungsdateien enthält, *überspringen Sie den Download der Übungsdateien*.

Um die Übungsdateien herunterzuladen, klonen Sie das Github Repository mit den Übungsdateien auf Ihrem lokalen Computer. Das Repository enthält alle Skripts und Ressourcen, die Sie zum Abschließen dieser Übung benötigen.

1. Öffnen Sie Visual Studio Code, wenn es noch nicht geöffnet ist.

1. Wählen Sie **Alle Befehle anzeigen** (STRG+UMSCHALT+P) aus, um die Befehlspalette zu öffnen.

1. Suchen Sie in der Befehlspalette den Befehl **Git: Klonen**, und wählen Sie ihn aus.

1. Geben Sie in der Befehlspalette Folgendes ein, um das Github Repository mit Übungsressourcen zu klonen, und drücken Sie die **EINGABETASTE**:

    ```bash
    https://github.com/MicrosoftLearning/mslearn-postgresql.git
    ```

1. Folgen Sie den Prompts, um einen Ordner auszuwählen, in den das Repository geklont werden soll. Das Repository wird in einen Ordner namens `mslearn-postgresql` an dem von Ihnen ausgewählten Speicherort geklont.

1. Wenn Sie gefragt werden, ob Sie das geklonte Repository öffnen möchten, wählen Sie **Öffnen** aus. Das Repository wird in Visual Studio Code geöffnet.

### Bereitstellen von Ressourcen in Ihrem Azure-Abonnement

Wenn Ihre Azure-Ressourcen bereits installiert sind, *überspringen Sie die Bereitstellung von Ressourcen*.

Dieser Schritt führt Sie durch die Verwendung von Azure CLI-Befehlen aus Visual Studio Code, um eine Ressourcengruppe zu erstellen und ein Bicep-Skript auszuführen, um die für diese Übung erforderlichen Azure-Services in Ihrem Azure-Abonnement bereitzustellen.

> &#128221; Wenn Sie mehrere Module in diesem Lernpfad absolvieren, können Sie die Azure-Umgebung für alle gemeinsam nutzen. In diesem Fall müssen Sie diesen Schritt der Ressourcenzuteilung nur einmal ausführen.

1. Öffnen Sie Visual Studio Code, falls es noch nicht geöffnet ist, und öffnen Sie den Ordner, in den Sie das Github Repository geklont haben.

1. Erweitern Sie den Ordner **mslearn-postgresql** im Explorer-Bereich.

1. Erweitern Sie den Ordner **Allfiles/Labs/Shared** .

1. Klicken Sie mit der rechten Maustaste auf den Ordner **Allfiles/Labs/Shared** und wählen Sie **In integriertem Terminal öffnen** aus. Diese Auswahl öffnet ein Terminalfenster im Visual Studio Code-Fenster.

1. Das Terminal kann standardmäßig ein **PowerShell**-Fenster öffnen. Für diesen Abschnitt des Labs verwenden Sie die **Bash Shell**. Neben dem **+**-Symbol gibt es einen Dropdownpfeil. Wählen Sie ihn aus, und wählen Sie **Git Bash** oder **Bash** aus der Liste der verfügbaren Profile aus. Diese Auswahl öffnet ein neues Terminalfenster mit der **Bash Shell**.

    > &#128221; Sie können das **PowerShell**-Terminalfenster schließen, wenn Sie möchten, aber es ist nicht erforderlich. Sie können mehrere Terminalfenster gleichzeitig geöffnet haben.

1. Führen Sie im Terminalfenster den folgenden Befehl aus, um sich bei Ihrem Azure-Konto anzumelden:

    ```bash
    az login
    ```

    Dieser Befehl öffnet ein neues Browserfenster, in dem Sie aufgefordert werden, sich bei Ihrem Azure-Konto anzumelden. Kehren Sie nach der Anmeldung zum Terminalfenster zurück.

1. Als Nächstes führen Sie drei Befehle aus, um Variablen zu definieren und so die redundante Eingabe zu reduzieren, wenn Sie Azure-CLI-Befehle zum Erstellen von Azure-Ressourcen verwenden. Die Variablen stehen für den Namen, den Sie Ihrer Ressourcengruppe zuweisen (`RG_NAME`), für die Azure-Region (`REGION`), in der die Ressourcen bereitgestellt werden und für ein zufällig generiertes Kennwort für den PostgreSQL-Administrator-Login (`ADMIN_PASSWORD`).

    Im ersten Befehl ist die Region, die der entsprechenden Variablen zugewiesen ist, `eastus`, aber Sie können sie auch durch einen Ort Ihrer Wahl ersetzen.

    ```bash
    REGION=eastus
    ```

    Der folgende Befehl weist den Namen zu, der für die Ressourcengruppe verwendet werden soll, die alle in dieser Übung verwendeten Ressourcen enthält. Der Name der Ressourcengruppe, die der entsprechenden Variablen zugewiesen ist, lautet `rg-learn-work-with-postgresql-$REGION`, wobei `$REGION` der zuvor angegebene Speicherort ist. *Sie können den Namen jedoch auch in einen anderen Namen für die Ressourcengruppe ändern, der Ihren Wünschen entspricht oder den Sie bereits verwenden*.

    ```bash
    RG_NAME=rg-learn-work-with-postgresql-$REGION
    ```

    Der letzte Befehl generiert nach dem Zufallsprinzip ein Kennwort für die PostgreSQL-Admin-Anmeldedaten. Kopieren Sie sie an einen sicheren Ort, damit Sie sie später verwenden können, um eine Verbindung zu Ihrem flexiblen PostgreSQL-Server herzustellen.

    ```bash
    #!/bin/bash
    
    # Define array of allowed characters explicitly
    chars=( {a..z} {A..Z} {0..9} '!' '@' '#' '$' '%' '^' '&' '*' '(' ')' '_' '+' )
    
    a=()
    for ((i = 0; i < 100; i++)); do
        rand_char=${chars[$RANDOM % ${#chars[@]}]}
        a+=("$rand_char")
    done
    
    # Join first 18 characters without delimiter
    ADMIN_PASSWORD=$(IFS=; echo "${a[*]:0:18}")
    
    echo "Your randomly generated PostgreSQL admin user's password is:"
    echo "$ADMIN_PASSWORD"
    echo "Please copy it to a safe place, as you will need it later to connect to your PostgreSQL flexible server."
    ```

1. Wenn Sie Zugriff auf mehr als ein Azure-Abonnement haben und Ihr Standardabonnement *nicht* das Abonnement ist, in dem Sie die Ressourcengruppe und andere Ressourcen für diese Übung erstellen möchten, führen Sie diesen Befehl aus, um das entsprechende Abonnement festzulegen. Ersetzen Sie dabei das Token `<subscriptionName|subscriptionId>` durch den Namen oder die ID des Abonnements, das Sie verwenden möchten:

    ```azurecli
    az account set --subscription 16b3c013-d300-468d-ac64-7eda0820b6d3
    ```

1. (Überspringen Sie diesen Schritt, wenn Sie bereits eine Ressourcengruppe verwenden.) Führen Sie den folgenden Azure CLI-Befehl aus, um Ihre Ressourcengruppe zu erstellen:

    ```azurecli
    az group create --name $RG_NAME --location $REGION
    ```

1. Verwenden Sie schließlich die Azure CLI, um ein Bicep-Bereitstellungsskript auszuführen, um Azure-Ressourcen in Ihrer Ressourcengruppe bereitzustellen:

    ```azurecli
    az deployment group create --resource-group $RG_NAME --template-file "Allfiles/Labs/Shared/deploy-postgresql-server.bicep" --parameters adminLogin=pgAdmin adminLoginPassword=$ADMIN_PASSWORD
    ```

    Das Bicep-Bereitstellungsskript stellt die für diese Übung erforderlichen Azure-Service in Ihrer Ressourcengruppe bereit. Die eingesetzten Ressourcen sind eine Azure-Datenbank für PostgreSQL – Flexibler Server. Das Bicep-Skript erstellt auch eine Datenbank, die in der Befehlszeile als Parameter konfiguriert werden kann.

    Die Bereitstellung dauert in der Regel mehrere Minuten. Sie können den Vorgang über das Bash-Terminal überwachen oder zur Seite **Bereitstellungen** für die zuvor erstellte Ressourcengruppe navigieren und dort den Fortschritt der Bereitstellung beobachten.

1. Da das Skript einen zufälligen Namen für den PostgreSQL-Server erstellt, können Sie den Namen des Servers finden, indem Sie den folgenden Befehl ausführen:

    ```azurecli
    az postgres flexible-server list --query "[].{Name:name, ResourceGroup:resourceGroup, Location:location}" --output table
    ```

    Notieren Sie sich den Namen des Servers, da Sie ihn benötigen, um später in dieser Übung eine Verbindung mit dem Server herzustellen.

    > &#128221; Sie finden auch den Namen des Servers im Azure-Portal. Navigieren Sie im Azure-Portal zu **Ressourcengruppen**, und wählen Sie die zuvor erstellte Ressourcengruppe aus. Der PostgreSQL-Server ist in der Ressourcengruppe aufgeführt.

### Problembehandlung bei der Bereitstellung

Beim Ausführen des Bicep-Bereitstellungsskripts können einige Fehler auftreten. Die häufigsten Meldungen und die Schritte zu ihrer Behebung sind:

- Wenn Sie zuvor das Bicep-Bereitstellungsskript für diesen Lernpfad ausgeführt und anschließend die Ressourcen gelöscht haben, wird möglicherweise die folgende Fehlermeldung angezeigt, wenn Sie versuchen, das Skript innerhalb von 48 Stunden nach dem Löschen der Ressourcen erneut auszuführen:

    ```bash
    {"code": "InvalidTemplateDeployment", "message": "The template deployment 'deploy' is not valid according to the validation procedure. The tracking id is '4e87a33d-a0ac-4aec-88d8-177b04c1d752'. See inner errors for details."}
    
    Inner Errors:
    {"code": "FlagMustBeSetForRestore", "message": "An existing resource with ID '/subscriptions/{subscriptionId}/resourceGroups/rg-learn-postgresql-ai-eastus/providers/Microsoft.CognitiveServices/accounts/{accountName}' has been soft-deleted. To restore the resource, you must specify 'restore' to be 'true' in the property. If you don't want to restore existing resource, please purge it first."}
    ```

    Wenn Sie diese Meldung erhalten, ändern Sie den vorherigen Befehl `azure deployment group create` so, dass der Parameter `restore` auf `true` festgelegt wird, und führen Sie ihn erneut aus.

- Wenn die ausgewählte Region für die Bereitstellung bestimmter Ressourcen eingeschränkt ist, müssen Sie die Variable `REGION` auf einen anderen Speicherort setzen und die Befehle erneut ausführen, um die Ressourcengruppe zu erstellen und das Bicep-Bereitstellungsskript auszuführen.

    ```bash
    {"status":"Failed","error":{"code":"DeploymentFailed","target":"/subscriptions/{subscriptionId}/resourceGroups/{resourceGrouName}/providers/Microsoft.Resources/deployments/{deploymentName}","message":"At least one resource deployment operation failed. Please list deployment operations for details. Please see https://aka.ms/arm-deployment-operations for usage details.","details":[{"code":"ResourceDeploymentFailure","target":"/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/Microsoft.DBforPostgreSQL/flexibleServers/{serverName}","message":"The resource write operation failed to complete successfully, because it reached terminal provisioning state 'Failed'.","details":[{"code":"RegionIsOfferRestricted","message":"Subscriptions are restricted from provisioning in this region. Please choose a different region. For exceptions to this rule please open a support request with Issue type of 'Service and subscription limits'. See https://review.learn.microsoft.com/en-us/azure/postgresql/flexible-server/how-to-request-quota-increase for more details."}]}]}}
    ```

- Wenn das Lab KI-Ressourcen benötigt, wird möglicherweise die folgende Fehlermeldung angezeigt. Dieser Fehler tritt auf, wenn das Skript keine KI-Ressource erstellen kann, da die zuständige KI-Vereinbarung akzeptiert werden muss. Wenn dies der Fall ist, verwenden Sie die Azure-Portal Benutzeroberfläche, um eine Azure KI Services-Ressource zu erstellen, und führen Sie dann das Bereitstellungsskript erneut aus.

    ```bash
    {"code": "InvalidTemplateDeployment", "message": "The template deployment 'deploy' is not valid according to the validation procedure. The tracking id is 'f8412edb-6386-4192-a22f-43557a51ea5f'. See inner errors for details."}
     
    Inner Errors:
    {"code": "ResourceKindRequireAcceptTerms", "message": "This subscription cannot create TextAnalytics until you agree to Responsible AI terms for this resource. You can agree to Responsible AI terms by creating a resource through the Azure Portal then trying again. For more detail go to https://go.microsoft.com/fwlink/?linkid=2164190"}
    ```

## Verbinden mit der PostgreSQL-Erweiterung in Visual Studio Code

In diesem Abschnitt stellen Sie eine Verbindung mit dem PostgreSQL-Server mithilfe der PostgreSQL-Erweiterung in Visual Studio Code her. Sie verwenden die PostgreSQL-Erweiterung, um SQL-Skripts auf dem PostgreSQL-Server auszuführen.

1. Öffnen Sie Visual Studio Code, falls es noch nicht geöffnet ist, und öffnen Sie den Ordner, in den Sie das Github Repository geklont haben.

1. Wählen Sie im linken Menü das **PostgreSQL**-Symbol aus.

    > &#128221; Wenn das PostgreSQL-Symbol nicht angezeigt wird, wählen Sie das Symbol **Erweiterungen** aus, und suchen Sie nach **PostgreSQL**. Wählen Sie die Erweiterung **PostgreSQL** von Microsoft und dann **Installieren** aus.

1. Wenn Sie bereits eine Verbindung mit Ihrem PostgreSQL-Server erstellt haben, fahren Sie mit dem nächsten Schritt fort. So erstellen Sie eine neue Verbindung:

    1. Wählen Sie in der Erweiterung **PostgreSQL** die Option **+ Verbindung hinzufügen**, um eine neue Verbindung hinzuzufügen.

    1. Geben Sie im Dialogfeld **NEUE VERBINDUNG** die folgenden Informationen ein:

        - **Servername**: `<your-server-name>`.postgres.database.azure.com
        - **Authentifizierungstyp**: Kennwort
        - **Benutzername**: pgAdmin
        - **Kennwort**: Das zufällige Kennwort, das Sie zuvor generiert haben.
        - Aktivieren Sie das Kontrollkästchen **Kennwort speichern**.
        - **Verbindungsname: **: `<your-server-name>`

    1. Überprüfen Sie die Verbindung, indem Sie **Verbindung testen** auswählen. Wenn die Verbindung erfolgreich hergestellt wurde, wählen Sie **Speichern und verbinden**, um die Verbindung zu speichern. Andernfalls überprüfen Sie die Verbindungsinformationen und versuchen Sie es erneut.

1. Falls noch keine Verbindung besteht, wählen Sie für Ihren PostgreSQL-Server die Option **Verbinden**. Sie sind nun mit dem Azure Database for PostgreSQL-Server verbunden.

1. Erweitern Sie den Serverknoten und die zugehörigen Datenbanken. Die vorhandenen Datenbanken werden aufgelistet.

1. Falls Sie die zoodb-Datenbank noch nicht erstellt haben, wählen Sie **Datei**, **Datei öffnen** und navigieren Sie zu dem Ordner, in dem Sie die Skripte gespeichert haben. Wählen Sie **../Allfiles/Labs/02/Lab2_ZooDb.sql** und **Öffnen**.

1. Vergewissern Sie sich unten rechts in Visual Studio Code, dass die Verbindung in Grün angezeigt wird. Ist dies nicht der Fall, sollte **PGSQL Disconnected** angezeigt werden. Wählen Sie den Text **PGSQL Disconnected** aus und wählen Sie anschließend Ihre PostgreSQL-Serververbindung aus der Liste in der Befehlspalette aus. Wenn Sie nach einem Kennwort gefragt werden, geben Sie das zuvor erstellte Kennwort ein.

1. Es ist an der Zeit, die Datenbank zu erstellen.

    1. Markieren Sie die Anweisungen **DROP** und **CREATE** und führen Sie sie aus.

    1. Wenn Sie nur die Anweisung **SELECT current_database()** markieren und ausführen, werden Sie feststellen, dass die Datenbank derzeit auf `postgres` festgelegt ist. Sie müssen sie in `zoodb` ändern.

    1. Wählen Sie die Ellipse in der Menüleiste mit dem Symbol *Ausführen* und wählen Sie **PostgreSQL-Datenbank ändern**. Wählen Sie in der Liste der Datenbanken die Option `zoodb` aus.

    > &#128221; Sie können die Datenbank auch im Abfragebereich ändern. Sie können den Servernamen und den Datenbanknamen unter der Abfrageregisterkarte selbst notieren. Durch Auswahl des Datenbanknamens wird eine Liste der Datenbanken angezeigt. Wählen Sie die Datenbank `zoodb` aus der Liste aus.

    1. Führen Sie die Anweisung **SELECT current_database()** erneut aus, um zu bestätigen, dass die Datenbank nun auf `zoodb` festgelegt ist.

    1. Markieren Sie die Abschnitte **Tabellen erstellen**, **Fremdschlüssel erstellen** und **Tabellen befüllen** und führen Sie sie aus.

    1. Markieren Sie die 3 **SELECT**-Anweisungen am Ende des Skripts und führen Sie sie aus, um zu überprüfen, ob die Tabellen erstellt und aufgefüllt wurden.

## Erstellen eines neuen Benutzerkontos in Microsoft Entra ID

> &#128221; In den meisten Produktions- oder Entwicklungsumgebungen verfügen Sie möglicherweise nicht über die erforderlichen Abonnementkontenberechtigungen, um Konten in Ihrem Microsoft Entra ID-Dienst zu erstellen. In diesem Fall sollten Sie, sofern dies von Ihrer Organisation erlaubt ist, Ihren Microsoft Entra ID-Administrator bitten, ein Testkonto für Sie zu erstellen. *Wenn Sie das Testkonto von Microsoft Entra nicht erhalten können, überspringen Sie diesen Abschnitt und fahren Sie mit dem Abschnitt **Zugriff gewähren auf Azure Database for PostgreSQL** fort*.

1. Melden Sie sich im [Azure-Portal](https://portal.azure.com) mit einem Besitzerkonto an und navigieren Sie zu Microsoft Entra ID.

1. Wählen Sie unter **Verwalten** die Option **Benutzer** aus.

1. Wählen Sie oben links **Neuer Benutzer** und dann **Neuen Benutzer erstellen** aus.

1. Geben Sie auf der Seite **Neuer Benutzer** die folgenden Details ein, und wählen Sie dann **Erstellen** aus:
    - **Benutzerprinzipalname:** Wählen Sie einen Prinzipalnamen
    - **Anzeigename:** Wählen Sie einen Anzeigenamen
    - **Kennwort:** Deaktivieren Sie **Kennwort automatisch generieren** und geben Sie dann ein sicheres Kennwort ein. Notieren Sie sich den Namen des Hauptbenutzers und das Kennwort.
    - Klicken Sie auf **Überprüfen + erstellen**.

    > &#128161; Wenn das Benutzerkonto erstellt wird, notieren Sie sich den vollständigen **Benutzerprinzipalnamen**, damit Sie ihn später zur Anmeldung verwenden können.

### Zuweisen der Rolle „Leser“

1. Wählen Sie im Azure-Portal **Alle Ressourcen** und dann Ihre Azure Database for PostgreSQL-Ressource aus.

1. Wählen Sie **Zugriffssteuerung (IAM)** und dann **Rollenzuweisungen** aus. Das neue Konto erscheint nicht in der Liste.

1. Wählen Sie **+ Hinzufügen** und dann **Rollenzuweisung hinzufügen** aus.

1. Wählen Sie die Rolle **Leser** und dann **Weiter** aus.

1. Wählen Sie **+ Mitglieder auswählen**, fügen Sie das neue Konto, das Sie im vorherigen Schritt hinzugefügt haben, zur Mitgliederliste hinzu und wählen Sie dann **Weiter**.

1. Wählen Sie **Überprüfen und zuweisen** aus.

### Testen der Rolle „Leser“

1. Wählen Sie oben rechts im Azure-Portal Ihr Benutzerkonto und dann **Abmelden** aus.

1. Melden Sie sich als neuer Benutzende mit dem Benutzernamen und dem Kennwort an, die Sie sich notiert haben. Ersetzen Sie das Standardkennwort, wenn Sie dazu aufgefordert werden, und merken Sie sich das neue Kennwort.

1. Wählen Sie **Später fragen**, wenn Sie zur Multi-Faktor-Authentifizierung aufgefordert werden.

1. Wählen Sie auf der Homepage des Portals **Alle Ressourcen** und dann Ihre Azure Database for PostgreSQL-Ressource aus.

1. Wählen Sie **Stop** (Beenden) aus. Daraufhin wird ein Fehler angezeigt, da die Rolle „Leser“ lediglich zulässt, die Ressource anzuzeigen, nicht aber sie zu ändern.

### Zuweisen der Rolle „Mitwirkender“

1. Wählen Sie oben rechts im Azure-Portal das Benutzerkonto des neuen Kontos aus und wählen Sie dann **Abmelden**.

1. Melden Sie sich mit Ihrem ursprünglichen Besitzerkonto an.

1. Navigieren Sie zu Ihrer Azure Database for PostgreSQL-Ressource, und wählen Sie dann **Zugriffssteuerung (IAM)** aus.

1. Wählen Sie **+ Hinzufügen** und dann **Rollenzuweisung hinzufügen** aus.

1. Wählen Sie **Privilegierte Administratorrollen**

1. Wählen Sie die Rolle **Mitwirkender** und dann **Weiter** aus.

1. Fügen Sie das neue Konto, das Sie zuvor hinzugefügt haben, zur Mitgliederliste hinzu und wählen Sie dann **Weiter**.

1. Wählen Sie **Überprüfen und zuweisen** aus.

1. Wählen Sie **Rollenzuweisungen** aus. Das neue Konto verfügt nun über Zuweisungen für die Rollen „Leser“ und „Mitwirkender“.

## Testen der Rolle „Mitwirkender“

1. Wählen Sie oben rechts im Azure-Portal Ihr Benutzerkonto und dann **Abmelden** aus.

1. Melden Sie sich mit dem neuen Konto an, mit dem Benutzernamen und dem Kennwort, die Sie notiert haben.

1. Wählen Sie auf der Homepage des Portals **Alle Ressourcen** und dann Ihre Azure Database for PostgreSQL-Ressource aus.

1. Wählen Sie **Beenden** und dann **Ja** aus. Dieses Mal wird der Server ohne Fehler gestoppt, da dem neuen Konto die erforderliche Rolle zugewiesen wurde.

1. Wählen Sie **Start**, um sicherzustellen, dass der PostgreSQL-Server für die nächsten Schritte bereit ist.

1. Wählen Sie oben rechts im Azure-Portal das Benutzerkonto des neuen Kontos aus und wählen Sie dann **Abmelden**.

1. Melden Sie sich mit Ihrem ursprünglichen Besitzerkonto an.

## Gewähren von Zugriff auf Azure Database for PostgreSQL

In diesem Abschnitt erstellen Sie eine neue Rolle in der PostgreSQL-Datenbank und weisen ihr Berechtigungen für den Zugriff auf die Datenbank zu. Außerdem testen Sie die neue Rolle, um sicherzustellen, dass sie über die richtigen Berechtigungen verfügt.

1. Öffnen Sie Visual Studio Code, wenn es noch nicht geöffnet ist.

1. Öffnen Sie die Befehlspalette (Strg + Umschalt + P) und wählen Sie **PGSQL: New Query**.

1. Wählen Sie Ihre PostgreSQL-Serververbindung aus der Liste in der Befehlspalette aus. Wenn Sie nach einem Kennwort gefragt werden, geben Sie das zuvor erstellte Kennwort ein.

1. Im Fenster **Neue Abfrage** ändern Sie die Datenbank in **zoodb**. Um die Datenbank zu ändern, wählen Sie die Ellipse in der Menüleiste mit dem Symbol *Ausführen* und wählen Sie **PostgreSQL-Datenbank ändern**. Wählen Sie in der Liste der Datenbanken die Option `zoodb` aus. Überprüfen Sie, ob die Datenbank jetzt auf `zoodb` festgelegt ist, indem Sie die Anweisung **SELECT current_database();** ausführen.

1. Kopieren Sie im Abfragefenster die folgende SQL-Anweisung, markieren Sie sie und führen Sie sie in der Postgres-Datenbank aus. Es sollten mehrere Benutzerrollen zurückgegeben werden, darunter auch die Rolle **pgAdmin**, die Sie für die Verbindung verwenden:

    ```SQL
    SELECT rolname FROM pg_catalog.pg_roles;
    ```

1. Führen Sie den folgenden Code aus, um eine neue Rolle zu erstellen:

    ```SQL
    -- Make sure to change the password to a complex one
    -- and replace the password in the script below
    CREATE ROLE dbuser WITH LOGIN NOSUPERUSER INHERIT CREATEDB NOCREATEROLE NOREPLICATION PASSWORD 'R3placeWithAComplexPW!';
    GRANT CONNECT ON DATABASE zoodb TO dbuser;
    ```

    > &#128221; Ersetzen Sie das Kennwort im vorherigen Skript durch ein komplexes Kennwort.

1. Um die neue Rolle aufzulisten, führen Sie die vorherige Auswahlabfrage in **pg_catalog.pg_roles** erneut aus. Die Rolle **dbuser** sollte aufgelistet werden.

1. Um die neue Rolle zum Abfragen und Ändern von Daten in der Tabelle **animal** in der Datenbank **zoodb** zu berechtigen: 

    1. Im Fenster **Neue Abfrage** ändern Sie die Datenbank in **zoodb**. Um die Datenbank zu ändern, wählen Sie die Ellipse in der Menüleiste mit dem Symbol *Ausführen* und wählen Sie **PostgreSQL-Datenbank ändern**. Wählen Sie in der Liste der Datenbanken die Option `zoodb` aus. Überprüfen Sie, ob die Datenbank jetzt auf `zoodb` festgelegt ist, indem Sie die Anweisung **SELECT current_database();** ausführen.

    1. Führen Sie diesen Code für die `zoodb`-Datenbank aus:

        ```SQL
        GRANT SELECT, INSERT, UPDATE, DELETE ON animal TO dbuser;
        ```

## Testen der neuen Rolle

Testen wir die neue Rolle, um sicherzustellen, dass sie über die richtigen Berechtigungen verfügt.

1. Wählen Sie in der Erweiterung **PostgreSQL** die Option **+ Verbindung hinzufügen**, um eine neue Verbindung hinzuzufügen.

1. Geben Sie im Dialogfeld **NEUE VERBINDUNG** die folgenden Informationen ein:

    - **Servername**: <Ihr-Server-Name>.postgres.database.azure.com
    - **Authentifizierungstyp**: Kennwort
    - **Benutzername**: dbuser
    - **Kennwort**: Das Kennwort, das Sie beim Erstellen der neuen Rolle verwendet haben.
    - Aktivieren Sie das Kontrollkästchen **Kennwort speichern**.
    - **Datenbankname**: zoodb
    - **Verbindungsname**: <Ihr-Server-Name> + "-dbuser"

1. Überprüfen Sie die Verbindung, indem Sie **Verbindung testen** auswählen. Wenn die Verbindung erfolgreich hergestellt wurde, wählen Sie **Speichern und verbinden**, um die Verbindung zu speichern. Andernfalls überprüfen Sie die Verbindungsinformationen und versuchen Sie es erneut.

1. Öffnen Sie die Befehlspalette (Strg + Umschalt + P) und wählen Sie **PGSQL: New Query**. Wählen Sie die neue Verbindung aus, die Sie in der Liste in der Befehlspalette erstellt haben. Wenn Sie nach einem Kennwort gefragt werden, geben Sie das Kennwort ein, das Sie für die neue Rolle erstellt haben.

1. Die Verbindung sollte standardmäßig die **zoodb-Datenbank** anzeigen. Falls nicht, können Sie die Datenbank in **zoodb** ändern. Um die Datenbank zu ändern, wählen Sie die Ellipse in der Menüleiste mit dem Symbol *Ausführen* und wählen Sie **PostgreSQL-Datenbank ändern**. Wählen Sie in der Liste der Datenbanken die Option `zoodb` aus. Überprüfen Sie, ob die Datenbank jetzt auf `zoodb` festgelegt ist, indem Sie die Anweisung **SELECT current_database();** ausführen.

1. Kopieren Sie im Fenster **Neue Abfrage** die folgende SQL-Anweisung, markieren Sie sie und führen Sie sie für die Datenbank **zoodb** aus:

    ```SQL
    SELECT * FROM animal;
    ```

1. Um zu überprüfen, ob Sie über die Berechtigung „UPDATE“ verfügen, kopieren Sie den folgenden Code, markieren Sie ihn und führen Sie ihn aus:

    ```SQL
    UPDATE animal SET name = 'Linda Lioness' WHERE ani_id = 7;
    SELECT * FROM animal;
    ```

1. Führen Sie den folgenden Code aus, um festzustellen, ob Sie über DROP-Berechtigung verfügen: Überprüfen Sie den Fehlercode, wenn ein Fehler auftritt:

    ```SQL
    DROP TABLE animal;
    ```

1. Führen Sie den folgenden Code aus, um festzustellen, ob Sie über GRANT-Berechtigung verfügen:

    ```SQL
    GRANT ALL PRIVILEGES ON animal TO dbuser;
    ```

Diese Tests zeigen, dass das neue Benutzerkonto DML-Befehle (Data Manipulation Language) zum Abfragen und Ändern von Daten ausführen kann. Das neue Benutzerkonto kann jedoch keine DDL-Befehle (Data Definition Language) verwenden, um das Schema zu ändern. Darüber hinaus kann es keine neuen Berechtigungen vergeben, um die Berechtigungen zu umgehen.

## Bereinigung

1. Wenn Sie diesen PostgreSQL-Server nicht mehr für andere Übungen benötigen, um unnötige Azure-Kosten zu vermeiden, löschen Sie die in dieser Übung erstellte Ressourcengruppe.

1. Löschen Sie bei Bedarf das Git Repository, das Sie zuvor geklont haben.
