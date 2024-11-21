---
lab:
  title: Konfigurieren Sie Systemparameter und erkunden Sie Metadaten mit Systemkatalogen und -ansichten.
  module: Configure and manage Azure Database for PostgreSQL
---

# Konfigurieren Sie Systemparameter und erkunden Sie Metadaten mit Systemkatalogen und -ansichten.

In dieser Übung werden Sie sich Systemparameter und Metadaten in PostgreSQL ansehen.

## Vor der Installation

> [!IMPORTANT]
> Sie benötigen ein eigenes Azure-Abonnement für die Übungen in diesem Modul. Wenn Sie kein Azure-Abonnement haben, können Sie unter [Cloudentwicklung dank kostenlosem Azure-Konto](https://azure.microsoft.com/free/) ein kostenloses Testkonto erstellen.

## Erstellen der Übungsumgebung

### Bereitstellen von Ressourcen in Ihrem Azure-Abonnement

Dieser Schritt führt Sie durch die Verwendung von Azure CLI-Befehlen aus der Azure Cloud Shell, um eine Ressourcengruppe zu erstellen und ein Bicep-Skript auszuführen, um die für diese Übung erforderlichen Azure-Services in Ihrem Azure-Abonnement bereitzustellen.

> Hinweis
>
> Wenn Sie mehrere Module in diesem Lernpfad absolvieren, können Sie die Azure-Umgebung für alle gemeinsam nutzen. In diesem Fall müssen Sie diesen Schritt der Ressourcenzuteilung nur einmal ausführen.

1. Öffnen Sie einen Webbrowser, und navigieren Sie zum [Azure-Portal](https://portal.azure.com/).

2. Wählen Sie das Symbol **Cloud Shell** in der Symbolleiste des Azure-Portals aus, um einen neuen [Cloud Shell](https://learn.microsoft.com/azure/cloud-shell/overview)-Bereich am unteren Rand Ihres Browserfensters zu öffnen.

    ![Screenshot der Azure-Symbolleiste mit dem Cloud Shell-Symbol, das durch eine rote Box hervorgehoben ist.](media/07-portal-toolbar-cloud-shell.png)

    Wählen Sie bei Aufforderung die erforderlichen Optionen aus, um eine *Bash*-Shell zu öffnen. Wenn Sie zuvor eine *PowerShell*-Konsole verwendet haben, wechseln Sie zu einer *Bash*-Shell.

3. Geben Sie an der Cloud Shell-Eingabeaufforderung Folgendes ein, um das GitHub-Repository mit den Übungsressourcen zu klonen:

    ```bash
    git clone https://github.com/MicrosoftLearning/mslearn-postgresql.git
    ```

4. Als Nächstes führen Sie drei Befehle aus, um Variablen zu definieren und so die redundante Eingabe zu reduzieren, wenn Sie Azure-CLI-Befehle zum Erstellen von Azure-Ressourcen verwenden. Die Variablen stehen für den Namen, den Sie Ihrer Ressourcengruppe zuweisen (`RG_NAME`), für die Azure-Region (`REGION`), in der die Ressourcen bereitgestellt werden und für ein zufällig generiertes Kennwort für den PostgreSQL-Administrator-Login (`ADMIN_PASSWORD`).

    Im ersten Befehl ist die Region, die der entsprechenden Variablen zugewiesen ist, `eastus`, aber Sie können sie auch durch einen Ort Ihrer Wahl ersetzen.

    ```bash
    REGION=eastus
    ```

    Mit dem folgenden Befehl weisen Sie den Namen für die Ressourcengruppe zu, die alle in dieser Übung verwendeten Ressourcen enthalten wird. Der Name der Ressourcengruppe, der der entsprechenden Variablen zugewiesen ist, lautet `rg-learn-work-with-postgresql-$REGION`, wobei `$REGION` der Ort ist, den Sie oben angegeben haben. Sie können den Namen jedoch auch in einen anderen Namen für die Ressourcengruppe ändern, der Ihren Wünschen entspricht.

    ```bash
    RG_NAME=rg-learn-work-with-postgresql-$REGION
    ```

    Der letzte Befehl generiert nach dem Zufallsprinzip ein Kennwort für das PostgreSQL-Admin-Login. Kopieren Sie sie an einen sicheren Ort, damit Sie sie später verwenden können, um eine Verbindung zu Ihrem flexiblen PostgreSQL-Server herzustellen.

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
    az deployment group create --resource-group $RG_NAME --template-file "mslearn-postgresql/Allfiles/Labs/Shared/deploy-postgresql-server.bicep" --parameters adminLogin=pgAdmin adminLoginPassword=$ADMIN_PASSWORD
    ```

    Das Bicep-Bereitstellungsskript stellt die für diese Übung erforderlichen Azure-Service in Ihrer Ressourcengruppe bereit. Die eingesetzten Ressourcen sind eine Azure-Datenbank für PostgreSQL – Flexibler Server. Das Bicep-Skript erstellt auch eine Datenbank, die in der Befehlszeile als Parameter konfiguriert werden kann.

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
    {"status":"Failed","error":{"code":"DeploymentFailed","target":"/subscriptions/{subscriptionId}/resourceGroups/{resourceGrouName}/providers/Microsoft.Resources/deployments/{deploymentName}","message":"At least one resource deployment operation failed. Please list deployment operations for details. Please see https://aka.ms/arm-deployment-operations for usage details.","details":[{"code":"ResourceDeploymentFailure","target":"/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/Microsoft.DBforPostgreSQL/flexibleServers/{serverName}","message":"The resource write operation failed to complete successfully, because it reached terminal provisioning state 'Failed'.","details":[{"code":"RegionIsOfferRestricted","message":"Subscriptions are restricted from provisioning in this region. Please choose a different region. For exceptions to this rule please open a support request with Issue type of 'Service and subscription limits'. See https://review.learn.microsoft.com/en-us/azure/postgresql/flexible-server/how-to-request-quota-increase for more details."}]}]}}
    ```

- Wenn das Skript aufgrund der Anforderung, die Vereinbarung über die verantwortungsvolle KI zu akzeptieren, keine KI-Ressource erstellen kann, kann der folgende Fehler auftreten. Verwenden Sie in diesem Fall die Benutzeroberfläche des Azure-Portals, um eine Azure-KI-Services-Ressource zu erstellen und führen Sie das Bereitstellungsskript dann erneut aus.

    ```bash
    {"code": "InvalidTemplateDeployment", "message": "The template deployment 'deploy' is not valid according to the validation procedure. The tracking id is 'f8412edb-6386-4192-a22f-43557a51ea5f'. See inner errors for details."}
     
    Inner Errors:
    {"code": "ResourceKindRequireAcceptTerms", "message": "This subscription cannot create TextAnalytics until you agree to Responsible AI terms for this resource. You can agree to Responsible AI terms by creating a resource through the Azure Portal then trying again. For more detail go to https://go.microsoft.com/fwlink/?linkid=2164190"}
    ```

### Herstellen einer Verbindung mit der Datenbank über Azure Data Studio

1. Falls noch nicht geschehen, klonen Sie die Lab-Skripte aus dem GitHub-Repository [PostgreSQL Labs](https://github.com/MicrosoftLearning/mslearn-postgresql.git) lokal:
    1. Öffnen Sie eine Befehlszeile/ein Terminal.
    1. Führen Sie den folgenden Befehl aus:
       ```bash
       md .\DP3021Lab
       git clone https://github.com/MicrosoftLearning/mslearn-postgresql.git .\DP3021Lab
       ```
       > HINWEIS
       > 
       > Wenn **git** nicht installiert ist, [laden Sie die App ***git*** herunter und installieren Sie sie](https://git-scm.com/download) und versuchen Sie, die vorherigen Befehle erneut auszuführen.
1. Wenn Sie Azure Data Studio noch nicht installiert haben, [laden Sie es herunter und installieren Sie ***Azure Data Studio***](https://go.microsoft.com/fwlink/?linkid=2282284).
1. Wenn Sie die Erweiterung **PostgreSQL** in Azure Data Studio noch nicht installiert haben, installieren Sie sie jetzt.
1. Öffnen Sie Azure Data Studio.
1. Wählen Sie **Verbindungen** aus.
1. Wählen Sie **Server** und dann **Neue Verbindung** aus.
1. Wählen Sie unter **Verbindungstyp** die Option **PostgreSQL** aus.
1. Geben Sie unter **Servername** den Wert ein, den Sie bei der Serverbereitstellung angegeben haben.
1. In **Benutzername** geben Sie **pgAdmin** ein.
1. Geben Sie in **Kennwort** das zufällig generierte Kennwort für das **pgAdmin** Login ein, das Sie generiert haben
1. Klicken Sie auf **Kennwort speichern**.
1. Klicken Sie auf das Menü **Verbinden**
1. Wenn Sie die zoodb-Datenbank noch nicht erstellt haben, wählen Sie **Datei**, **Datei öffnen** und navigieren Sie zu dem Ordner, in dem Sie die Skripts gespeichert haben. Wählen Sie **../Allfiles/Labs/02/Lab2_ZooDb.sql** und **Öffnen**.
   1. Markieren Sie die Anweisungen **DROP** und **CREATE** und führen Sie sie aus.
   1. Zeigen Sie oben im Bildschirm mithilfe des Dropdownpfeils die Datenbanken auf dem Server an, einschließlich zoodb und Systemdatenbanken. Wählen Sie die Datenbank **zoodb**.
   1. Markieren Sie die Abschnitte **Tabellen erstellen**, **Fremdschlüssel erstellen** und **Tabellen befüllen** und führen Sie sie aus.
   1. Markieren Sie die 3 **SELECT**-Anweisungen am Ende des Skripts und führen Sie sie aus, um zu überprüfen, ob die Tabellen erstellt und aufgefüllt wurden.

## Aufgabe 1: Untersuchen Sie den Vakuum-Prozess in PostgreSQL

1. Falls nicht geöffnet, öffnen Sie Azure Data Studio.
1. Wählen Sie in Azure Data Studio **Datei** und dann **Datei öffnen** aus, und navigieren Sie danach zu den Skripts für das Lab. Wählen Sie **../Allfiles/Labs/07/Lab7_vacuum.sql** und wählen Sie dann **Öffnen**. Stellen Sie bei Bedarf erneut eine Verbindung mit dem Server her.
1. Wählen Sie die Datenbank **zoodb** aus der Datenbank-Dropdown-Liste.
1. Markieren Sie den Abschnitt **Check zoodb database is selected** (Überprüfen, ob die zoodb-Datenbank ausgewählt ist), und führen Sie ihn aus. Bei Bedarf können Sie zoodb mithilfe der Dropdownliste als aktuelle Datenbank auswählen.
1. Markieren Sie den Abschnitt **Display dead tuples** (Inaktive Tupel anzeigen), und führen Sie ihn aus. Diese Abfrage zeigt die Anzahl der inaktiven und aktiven Tupel in der Datenbank an. Notieren Sie sich die Anzahl der inaktiven Tupel.
1. Markieren Sie den Abschnitt **Gewicht ändern** und führen Sie ihn 10 Mal hintereinander aus. Diese Abfrage aktualisiert die Gewichtungsspalte für alle Tiere.
1. Führen Sie den Abschnitt unter **Display dead tuples** (Inaktive Tupel anzeigen) erneut aus. Notieren Sie sich die Anzahl der inaktiven Tupel, nachdem die Aktualisierungen abgeschlossen wurden.
1. Führen Sie den Abschnitt unter **Manually run VACUUM** (VACUUM manuell ausführen) aus, um den VACUUM-Prozess auszuführen.
1. Führen Sie den Abschnitt unter **Display dead tuples** (Inaktive Tupel anzeigen) erneut aus. Notieren Sie sich die Anzahl der inaktiven Tupel, nachdem der VACUUM-Prozess ausgeführt wurde.

## Aufgabe 2: Konfigurieren Sie die Parameter des Autovacuum-Servers

1. Navigieren Sie im Azure-Portal zu Ihrer Instanz von Azure Database for PostgreSQL Flexible Server.
1. Klicken Sie unter **Einstellungen** auf **Serverparameter**.
1. Geben Sie in der Suchleiste **`vacuum`** ein. Suchen Sie die folgenden Parameter, und ändern Sie die Werte wie folgt:
    1. autovacuum = ON (sollte standardmäßig „ON“ lauten)
    1. autovacuum_vacuum_scale_factor = 0.1
    1. autovacuum_vacuum_threshold = 50

    Dies entspricht der Ausführung des autovacuum-Prozesses, wenn 10 Prozent einer Tabelle zum Löschen von Zeilen markiert sind oder 50 Zeilen in einer beliebigen Tabelle aktualisiert oder gelöscht wurden.

1. Wählen Sie **Speichern**. Der Server wird neu gestartet.

## Aufgabe 3: Anzeigen von PostgreSQL-Metadaten im Azure-Portal

1. Wechseln Sie zum [Azure-Portal](https://portal.azure.com), und melden Sie sich an.
1. Suchen Sie nach **Azure Database for PostgreSQL** und wählen Sie es aus.
1. Wählen Sie die Instanz von Azure Database for PostgreSQL – Flexibler Server aus, die Sie für diese Übung erstellt haben.
1. Wählen Sie unter **Überwachung** die Option **Metriken** aus.
1. Wählen Sie **Metrik** und dann **CPU-Prozentsatz** aus.
1. Beachten Sie, dass verschiedene Metriken zu Ihren Datenbanken angezeigt werden.

## Aufgabe 4: Anzeigen von Daten in Systemkatalogtabellen

1. Wechseln Sie zu Azure Data Studio.
1. Wählen Sie unter **SERVER** Ihren PostgreSQL-Server aus, und warten Sie, bis eine Verbindung hergestellt wurde und ein grüner Kreis auf dem Server angezeigt wird.
1. Klicken Sie mit der rechten Maustaste auf die Server, und wählen Sie **Neue Abfrage** aus.
1. Geben Sie den folgenden SQL-Code ein, und wählen Sie **Ausführen** aus:

    ```sql
    SELECT datname, xact_commit, xact_rollback FROM pg_stat_database;
    ```

1. Beachten Sie, dass Sie Commits und Rollbacks für jede Datenbank anzeigen können.

## Anzeigen einer komplexen Metadatenabfrage über eine Systemansicht

1. Klicken Sie mit der rechten Maustaste auf die Server, und wählen Sie **Neue Abfrage** aus.
1. Geben Sie den folgenden SQL-Code ein, und wählen Sie **Ausführen** aus:

    ```sql
    SELECT *
    FROM pg_catalog.pg_stats;
    ```

1. Beachten Sie, dass Sie eine große Menge an Statistikinformationen einsehen können.
1. Mithilfe von Systemsichten können Sie die Komplexität des SQL-Codes reduzieren, den Sie schreiben müssen. Die vorherige Abfrage benötigt den folgenden Code, wenn Sie die Sicht **pg_stats** nicht verwenden:

    ```sql
    SELECT n.nspname AS schemaname,
    c.relname AS tablename,
    a.attname,
    s.stainherit AS inherited,
    s.stanullfrac AS null_frac,
    s.stawidth AS avg_width,
    s.stadistinct AS n_distinct,
        CASE
            WHEN s.stakind1 = 1 THEN s.stavalues1
            WHEN s.stakind2 = 1 THEN s.stavalues2
            WHEN s.stakind3 = 1 THEN s.stavalues3
            WHEN s.stakind4 = 1 THEN s.stavalues4
            WHEN s.stakind5 = 1 THEN s.stavalues5
            ELSE NULL::anyarray
        END AS most_common_vals,
        CASE
            WHEN s.stakind1 = 1 THEN s.stanumbers1
            WHEN s.stakind2 = 1 THEN s.stanumbers2
            WHEN s.stakind3 = 1 THEN s.stanumbers3
            WHEN s.stakind4 = 1 THEN s.stanumbers4
            WHEN s.stakind5 = 1 THEN s.stanumbers5
            ELSE NULL::real[]
        END AS most_common_freqs,
        CASE
            WHEN s.stakind1 = 2 THEN s.stavalues1
            WHEN s.stakind2 = 2 THEN s.stavalues2
            WHEN s.stakind3 = 2 THEN s.stavalues3
            WHEN s.stakind4 = 2 THEN s.stavalues4
            WHEN s.stakind5 = 2 THEN s.stavalues5
            ELSE NULL::anyarray
        END AS histogram_bounds,
        CASE
            WHEN s.stakind1 = 3 THEN s.stanumbers1[1]
            WHEN s.stakind2 = 3 THEN s.stanumbers2[1]
            WHEN s.stakind3 = 3 THEN s.stanumbers3[1]
            WHEN s.stakind4 = 3 THEN s.stanumbers4[1]
            WHEN s.stakind5 = 3 THEN s.stanumbers5[1]
            ELSE NULL::real
        END AS correlation,
        CASE
            WHEN s.stakind1 = 4 THEN s.stavalues1
            WHEN s.stakind2 = 4 THEN s.stavalues2
            WHEN s.stakind3 = 4 THEN s.stavalues3
            WHEN s.stakind4 = 4 THEN s.stavalues4
            WHEN s.stakind5 = 4 THEN s.stavalues5
            ELSE NULL::anyarray
        END AS most_common_elems,
        CASE
            WHEN s.stakind1 = 4 THEN s.stanumbers1
            WHEN s.stakind2 = 4 THEN s.stanumbers2
            WHEN s.stakind3 = 4 THEN s.stanumbers3
            WHEN s.stakind4 = 4 THEN s.stanumbers4
            WHEN s.stakind5 = 4 THEN s.stanumbers5
            ELSE NULL::real[]
        END AS most_common_elem_freqs,
        CASE
            WHEN s.stakind1 = 5 THEN s.stanumbers1
            WHEN s.stakind2 = 5 THEN s.stanumbers2
            WHEN s.stakind3 = 5 THEN s.stanumbers3
            WHEN s.stakind4 = 5 THEN s.stanumbers4
            WHEN s.stakind5 = 5 THEN s.stanumbers5
            ELSE NULL::real[]
        END AS elem_count_histogram
    FROM pg_statistic s
     JOIN pg_class c ON c.oid = s.starelid
     JOIN pg_attribute a ON c.oid = a.attrelid AND a.attnum = s.staattnum
     LEFT JOIN pg_namespace n ON n.oid = c.relnamespace
    WHERE NOT a.attisdropped AND has_column_privilege(c.oid, a.attnum, 'select'::text) AND (c.relrowsecurity = false OR NOT row_security_active(c.oid));
    ```

## Bereinigung der Übung

1. Für die Azure Database for PostgreSQL, die wir in dieser Übung eingesetzt haben, fallen Gebühren an. Sie können den Server nach dieser Übung löschen. Alternativ können Sie auch die Ressourcengruppe **rg-learn-work-with-postgresql-eastus** löschen, um alle Ressourcen zu entfernen, die wir im Rahmen dieser Übung bereitgestellt haben.
1. Löschen Sie bei Bedarf den Ordner .\DP3021Lab.
