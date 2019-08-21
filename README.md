# oereb-nutzungsplanung

Gretljobs für die Entwicklung des Datenumbaus der Nutzungsplanung (Kanton Solothurn) in die Transferstruktur des Rahmenmodelles.

## Datenumbauprozess

Momentan handelt es sich um separate Gradle Builds. Je nach Weiterverwendung des Codes / des Repos kann man das ohne Problem anpassen.

### (1) Starten der lokalen Edit-DB:

```
gradle -I $PWD/init.gradle -b initdb/build.gradle startAndWaitOnHealthyContainer
```

Startet einen Docker-Container mit einer PostgreSQL/PostGIS-DB, welche die Edit-DB simuliert. Es werden keine Daten peristiert, alles wird nur im Docker-Container gespeichert und sind nach dem Stoppen des Containers verloren. Die DB startet mit zwei leeren Schemen:

- arp_npl: Nutzungsplanung im kantonalen Modell
- arp_npl_oereb: Rahmenmodell (Transferstruktur)

### (2) Import der notwendigen Daten:
Es müssen folgende Daten in die Edit-DB importiert werden, damit der Datenumbau durchgeführt werden kann und Daten modellkonform in die Transferstruktur exportiert werden können:

- Bundesgesetze und -Verordnungen
- Kantonale Gesetzt und Verordnungen
- Zuständige Stellen
- Nutzungsplanung im kantonalen Modell

```
gradle -I $PWD/init.gradle -b initdb/build.gradle importFederalLegalBasisToOereb importCantonalLegalBasisToOereb importResponsibleOfficesToOereb replaceLandUsePlansData
```

oder

```
gradle -I $PWD/init.gradle -b initdb/build.gradle importData
```

### (3) Datenumbau und -export
Die Daten werden mit einem `SQLExecutor`-Task umgebaut und anschliessend in die Transferstruktur exportiert, geprüft und auf S3 hochgeladen. 

```
gradle -I $PWD/init.gradle -b transfer/build.gradle insertToOereb updateSymbols exportLandUsePlansOereb validateLandUsePlansExport uploadToS3
```

## ÖREB-Datenbank mit Daten
Es kann ein Docker-Image mit einer ÖREB-Datenbank erstellt werden, die sämtliche Daten beinhaltet. Der Herstellen des Images kann einige Zeit in Anspruch nehmen. Insbesondere der Import der amtlichen Vermessung (via ITF) dauert lange, falls sämtliche Gemeinden importiert werden.

Es wird auf Basis von sogis/oereb-db ein Docker-Container gestartet. Die DB-Daten werden im Ordner `/tmp/primary/` gespeichert. Nach dem Import der gewünschten Dateien wird dieser Container heruntergefahren, das Verzeichnis `/tmp/primary/` in das Build-Verzeichnisses des Gretl-Jobs kopiert und anschliessend in das Docker-Image sogis/oereb-db-data gebrannt.

```
gradle -I $PWD/init.gradle -b dbimage/build.gradle importAllDataToOerebDb buildFatOerebDbDockerImage
```

oder falls man nur die Nutzungsplanung ersetzen will und die Daten der temporären ÖREB-Datenbank noch vorhanden sind:

```
gradle -I $PWD/init.gradle -b dbimage/build.gradle importLandUsePlansDataToOerebDb buildFatOerebDbDockerImage
```



## Bemerkungen

- Gradle wird ohne Daemon gestartet. Gefühlt immer noch bessere Erfahrungen bezüglich "out of memory" v.a. wenn das XTF gross wird. Ausgeschaltet wird der Daemon in der `gradle.properties`-Datei, die dort liegen muss wo Gradle ausgeführt wird.
- Der Gretl-Job für das Herstellen des Docker-Images _mit_ den Daten sollte, wenn weitere kantonale Themen vorhanden sind, in einem separaten Repository verwaltet werden. In diesem Fall (falls mit Subprojekten gearbeitet wird) können die Imports auch parallisiert werden.

