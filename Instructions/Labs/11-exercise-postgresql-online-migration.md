---
lab:
  title: Online PostgreSQL Datenbank-Migration
  module: Migrate to Azure Database for PostgreSQL Flexible Server
---

## Online PostgreSQL Datenbank-Migration

In dieser Übung konfigurieren Sie die logische Replikation zwischen einem PostgreSQL-Quellserver und Azure Database for PostgreSQL – Flexibler Server, um eine Online-Migrationsaktivität durchführen zu können.

## Vor der Installation

Für diese Übung benötigen Sie ein eigenes Azure-Abonnement. Wenn Sie nicht über ein Azure-Abonnement verfügen, können Sie ein [kostenloses Azure-Testkonto](https://azure.microsoft.com/free) erstellen.

> **Hinweis:**: Für diese Übung ist es erforderlich, dass der Server, den Sie als Quelle für die Migration verwenden, für den Azure Database for PostgreSQ – Flexibler Server zugänglich ist, damit dieser eine Verbindung herstellen und Datenbanken migrieren kann. Dazu muss der Quellserver über eine öffentliche IP-Adresse und einen öffentlichen Port erreichbar sein. > Eine Liste der IP-Adressen der Azure-Region können Sie unter [Azure IP-Bereiche und Service-Tags – Public Cloud](https://www.microsoft.com/en-gb/download/details.aspx?id=56519) herunterladen, um die zulässigen IP-Adressbereiche in Ihren Firewall-Regeln auf der Grundlage der verwendeten Azure-Region zu minimieren.

Öffnen Sie die Firewall Ihres Servers, um der Migrationsfunktion des Azure Database for PostgreSQL – Flexibler Server den Zugriff auf den PostgreSQL-Quellserver zu ermöglichen, der standardmäßig der TCP-Port 5432 ist.
>
Wenn Sie eine Firewall-Appliance vor Ihrer Quelldatenbank verwenden, müssen Sie möglicherweise Firewall-Regeln hinzufügen, damit die Migrationsfunktion des Azure Database for PostgreSQL – Flexibler Server auf die Quelldatenbank(en) für die Migration zugreifen kann.
>
> Die maximal unterstützte Version von PostgreSQL für die Migration ist Version 16.

### Voraussetzungen

> **Hinweis**: Bevor Sie mit dieser Übung beginnen, müssen Sie die vorangegangene Übung abgeschlossen haben, um die Quell- und Zieldatenbank für die Konfiguration der logischen Replikation einzurichten, da diese Übung auf den Aktivitäten in dieser Übung aufbaut.

## Publikation erstellen – Quellserver

1. Öffnen Sie PGAdmin und stellen Sie eine Verbindung zu dem Quellserver her, der die Datenbank enthält, die als Quelle für die Datensynchronisierung mit dem Azure Database for PostgreSQL – Flexibler Server dienen soll.
1. Öffnen Sie ein neues Abfragefenster, das mit der Quelldatenbank mit den zu synchronisierenden Daten verbunden ist.
1. Konfigurieren Sie den Quellserver wal_level auf **logisch**, um die Veröffentlichung von Daten zu ermöglichen.
    1. Suchen und öffnen Sie die Datei **postgresql.conf** im Verzeichnis „bin“ des PostgreSQL-Installationsverzeichnisses.
    1. Suchen Sie die Zeile mit der Konfigurationseinstellung **wal_level**.
    1. Stellen Sie sicher, dass die Zeile unkommentiert ist und setzen Sie den Wert auf **logisch**.
    1. Speichern und schließen Sie die Datei.
    1. Starten Sie den PostgreSQL-Service neu.
1. Konfigurieren Sie nun eine Publikation, die alle Tabellen in der Datenbank enthalten wird.

    ```SQL
    CREATE PUBLICATION migration1 FOR ALL TABLES;
    ```

## Abonnement erstellen – Zielserver

1. Öffnen Sie PGAdmin und stellen Sie eine Verbindung zu dem Azure Database for PostgreSQL – Flexibler Server her, der die Datenbank enthält, die als Ziel für die Datensynchronisierung vom Quellserver dienen soll.
1. Öffnen Sie ein neues Abfragefenster, das mit der Quelldatenbank mit den zu synchronisierenden Daten verbunden ist.
1. Erstellen Sie das Abonnement für den Quellserver.

    ```sql
    CREATE SUBSCRIPTION migration1
    CONNECTION 'host=<source server name> port=<server port> dbname=adventureworks application_name=migration1 user=<username> password=<password>'
    PUBLICATION migration1
    WITH(copy_data = false)
    ;    
    ```

1. Prüfen Sie den Status der Tabellenreplikation.

    ```SQL
    SELECT PT.schemaname, PT.tablename,
        CASE PS.srsubstate
            WHEN 'i' THEN 'initialize'
            WHEN 'd' THEN 'data is being copied'
            WHEN 'f' THEN 'finished table copy'
            WHEN 's' THEN 'synchronized'
            WHEN 'r' THEN ' ready (normal replication)'
            ELSE 'unknown'
        END AS replicationState
    FROM pg_publication_tables PT,
            pg_subscription_rel PS
            JOIN pg_class C ON (C.oid = PS.srrelid)
            JOIN pg_namespace N ON (N.oid = C.relnamespace)
    ;
    ```

## Testdaten-Replikation

1. Prüfen Sie auf dem Quellserver die Anzahl der Zeilen in der Tabelle der Arbeitsaufträge.

    ```SQL
    SELECT COUNT(*) FROM production.workorder;
    ```

1. Prüfen Sie auf dem Zielserver die Zeilenzahl der Arbeitsauftragstabelle.

    ```SQL
    SELECT COUNT(*) FROM production.workorder;
    ```

1. Prüfen Sie, ob die Zeilenzählwerte übereinstimmen.
1. Laden Sie nun die Datei Lab11_workorder.csv aus dem Repository [hier](https://github.com/MicrosoftLearning/mslearn-postgresql/tree/main/Allfiles/Labs/11) nach C:\
1. Laden Sie mit dem folgenden Befehl neue Daten aus der CSV-Datei in die Arbeitsauftragstabelle auf dem Quellserver.

    ```Bash
    psql --host=localhost --port=5432 --username=postgres --dbname=adventureworks --command="\COPY production.workorder FROM 'C:\Lab11_workorder.csv' CSV HEADER"
    ```

Die Befehlsausgabe sollte `COPY 490` lauten, was bedeutet, dass 490 zusätzliche Zeilen aus der CSV-Datei in die Tabelle geschrieben wurden.

1. Prüfen Sie, ob die Zeilenzahlen für die Workorder-Tabelle in der Quelle (72591 Zeilen) und im Ziel übereinstimmen, um zu überprüfen, ob die Datenreplikation funktioniert.

## Bereinigung der Übung

Für die Azure Database for PostgreSQL, die wir in dieser Übung eingesetzt haben, fallen Gebühren an. Sie können den Server nach dieser Übung löschen. Alternativ können Sie auch die Ressourcengruppe **rg-learn-work-with-postgresql-eastus** löschen, um alle Ressourcen zu entfernen, die wir im Rahmen dieser Übung bereitgestellt haben.
