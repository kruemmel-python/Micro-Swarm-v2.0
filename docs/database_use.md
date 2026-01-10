# MycoDB Anleitung (Micro-Swarm)

Diese Anleitung beschreibt den kompletten Workflow: von MySQL Workbench ueber den SQL-Dump bis zur Ingestion, interaktiven Abfrage und Ausgabe in Micro-Swarm (MycoDB).

## 1a) Was MycoDB ist / was es nicht ist

MycoDB ist:
- raeumliches Clustering + lokales Retrieval von relationalen Payloads
- Shell-Navigation und ein SQL-Light Interpreter als Komfort-Layer
- optimiert fuer lokale Abfragen (Radius/Fokus); globale Abfragen sind moeglich, aber konzeptionell der Worst Case.

MycoDB ist nicht:
- keine ACID-DB, keine Transaktionen, keine Konkurrenzkontrolle
- kein Query-Optimizer/Kostenmodell im DB-Sinne
- keine vollstaendige SQL-Compliance
- kein Ersatz fuer Indexstrukturen mit garantierter Worst-Case-Komplexitaet

Diese Abgrenzung ist wichtig, weil "DB" sonst zu klassischen Postgres-Erwartungen fuehrt.

## 1) Voraussetzungen

- MySQL Workbench installiert
- Micro-Swarm gebaut (Release-Binary)
- Genug freier Speicher fuer Dump und .myco Datei

Empfohlene Struktur im Projektordner:

```
./
  build/Release/micro_swarm.exe
  data/
    dump.sql
    shop.myco
    clusters.ppm
```

## 1b) Datenvolumen und Komplexitaet (kurz)

Speicherkosten skalieren grob mit `payload_count`, der Grid-Groesse (`size * size`) und Ergebnis-Buffer groessen (Trefferlisten). `--size` erhoeht die Grid-Flaeche quadratisch (256 -> 512 = 4x Zellen). Lokale Queries skalieren mit Radius und Cluster-Dichte; `--db-radius` steuert die Suchflaeche naeherungsweise quadratisch mit dem Radius (doppelt so grosser Radius ~ 4x Flaeche). Worst Case ist ein globaler Scan (z. B. grosser Radius oder Fokus aus).

## 2) SQL-Dump in MySQL Workbench erstellen

1. MySQL Workbench starten.
2. Verbindung zur Datenbank oeffnen.
3. Menue: Server -> Data Export.
4. Schema auswaehlen.
5. Unter Export Options:
   - Dump Structure and Data aktivieren.
   - Export to Self-Contained File auswaehlen.
   - Include Column Names aktivieren (wichtig fuer saubere Queries).
6. Dateipfad waehlen, z. B. `C:\Users\<name>\Downloads\dump.sql`.
7. Start Export.

Hinweis: Micro-Swarm liest `INSERT INTO ... VALUES (...)` Statements. Mit Spaltenliste sind Queries ueber Spaltennamen moeglich.
Wenn der Dump `CREATE TABLE` enthaelt, werden Spaltennamen automatisch aus dem Schema gelesen.

## 3) SQL-Dump in Micro-Swarm einspielen (db_ingest)

Wechsle in das Release-Verzeichnis:

```powershell
cd C:\Users\<name>\Downloads\superpromt\micro_DB\build\Release
```

Starte die Ingestion:

```powershell
.\micro_swarm.exe --mode db_ingest --input C:\path\to\dump.sql --steps 5000 --agents 512 --size 256 --output shop.myco --db-dump clusters.ppm --db-dump-scale 6
```

Parameter-Erklaerung:
- `--mode db_ingest` aktiviert den Datenbank-Ingestion-Modus
- `--input` Pfad zum SQL-Dump
- `--steps` Anzahl der Schwarm-Schritte (mehr Schritte = staerkeres Clustering)
- `--agents` Anzahl Traeger-Agenten
- `--size` Grid-Groesse (z. B. 256x256)
- `--output` Ziel .myco Datei
- `--db-dump` optionales PPM-Bild der Cluster
- `--db-dump-scale` vergroessert das Bild (z. B. 6 = 6x groesser)

Erwartete Ausgabe:

```
ingest_done payloads=<N> tables=<T>
```

## 3a) Reproduzierbare Runs (golden path)

Empfohlene Konfiguration fuer reproduzierbare Reports:
- fixer SQL-Dump (gleiche Datei, gleiche Reihenfolge)
- feste Parameter: `--seed`, `--size`, `--agents`, `--steps`
- feste Query-Parameter: `--db-radius` bzw. `radius` in der Shell
- falls GPU/OpenCL aktiv ist, koennen numerische Abweichungen auftreten; fuer strikte Replays GPU nicht aktivieren oder toleranzbasiert vergleichen

Beispiel:

```powershell
.\micro_swarm.exe --mode db_ingest --input C:\path\to\dump.sql --steps 5000 --agents 512 --size 256 --seed 42 --output shop.myco
```

## 4) Cluster-Bild ansehen (optional)

Das PPM-Bild zeigt die Tabellen-Cluster. Du kannst es oeffnen mit:

```powershell
start clusters.ppm
```

Oder mit einem Bildbetrachter deiner Wahl.

## 5) Interaktive Abfragen in Micro-Swarm (db_shell)

Starte die Shell:

```powershell
.\micro_swarm.exe --mode db_shell --db shop.myco --db-radius 5
```

Shell-Cheat-Sheet (12 Zeilen):

```
tables
schema <table>
goto <payload_id>
focus
radius <N>
unfocus
<Table> <PKValue>
<Table> <col>=<val>
show <col1,col2,...>
sql <statement>
sort <col|index> [asc|desc] [num][, <col|index> [asc|desc] [num] ...]
format <table|csv|json>
```

Beispieleingaben:

```
Album 1
Track AlbumId=1
goto 1234
radius 12
tables
stats
schema Track
Track AlbumId=1 show TrackId,Name,Milliseconds
TrackId=13
sql SELECT TrackId,Name FROM Track LIMIT 5
sql SELECT TrackId,Name FROM Track WHERE TrackId=1
sql SELECT TrackId,Name FROM Track WHERE AlbumId=1 ORDER BY TrackId
sql SELECT TrackId,Name FROM Track WHERE AlbumId='1' ORDER BY TrackId


```

Bedeutung:
- `Album 1` sucht nach Primary Key `AlbumId=1`.
- `Track AlbumId=1` sucht alle Tracks mit Foreign Key auf AlbumId.
- `goto 1234` setzt den Fokus auf Payload ID 1234.
- `radius 12` setzt den Suchradius fuer alle folgenden Anfragen.
- `tables` listet alle Tabellen.
- `schema <table>` zeigt bekannte Spalten.
- `show ...` filtert die Ausgabe auf Spalten.
- `<Column>=<Value>` ohne Tabellennamen sucht global.
- `sql <statement>` fuehrt SQL-Light aus (Phase 1).
- `sort <col|index> [asc|desc] [num][, <col|index> [asc|desc] [num] ...]` sortiert das letzte SQL-Result oder das letzte Shell-Result.
- `sort reset` stellt das letzte SQL-Result wieder her.
- `format <table|csv|json>` setzt das SQL-Ausgabeformat.

Solange ein Fokus gesetzt ist, werden alle Suchanfragen lokal um diesen Punkt ausgefuehrt.

## 6) Query-Format (SQL-Light, optional)

Wenn du statt der Shell den Query-Modus nutzen willst:

```powershell
.\micro_swarm.exe --mode db_query --db shop.myco --query "SELECT * FROM Orders WHERE UserID=5" --db-radius 5
```

Optional:

```powershell
.\micro_swarm.exe --mode db_query --db shop.myco --query "SELECT Name FROM Artist ORDER BY ArtistId LIMIT 5" --sql-format csv
```

Formate:

```
SELECT * FROM <Table> WHERE <Column>=<Value>
```

SQL-Light (Phase 1):

```
SELECT <cols> FROM <table> [JOIN ...]
WHERE ... (AND/OR/NOT, IN, BETWEEN, LIKE)
GROUP BY ... HAVING ...
ORDER BY ... LIMIT ... OFFSET ...
DISTINCT, UNION, UNION ALL
RIGHT JOIN, CROSS JOIN
SUBQUERY: IN (SELECT ...), EXISTS (SELECT ...)
REGEXP, WITH (CTE)
IS NULL / IS NOT NULL
ORDER BY Spaltenindex
Funktionen: LOWER, UPPER, LENGTH, SUBSTRING, CONCAT
FROM (SELECT ...) als Subquery
```

Hinweise:
- Subqueries koennen aeussere Spalten referenzieren (korreliert).
- CTEs sind **nicht rekursiv**.
- Numerische Literale koennen ohne Quotes genutzt werden (z. B. `AlbumId=1`).
- `ORDER BY` sortiert numerisch, wenn beide Werte als Zahl parsebar sind.

## 7) Fehlerdiagnose

- `hits=0`:
  - Pruefe Tabellennamen und Spaltennamen (Gross-/Kleinschreibung egal).
  - Falls der Dump ohne Spaltenliste ist, nutze `Album 1` (Primary Key) oder `col0`.
- `SQL-Fehler: Keine INSERT-Statements gefunden`:
  - Workbench Export erneut pruefen.
- `MYCO-Fehler`:
  - Dateipfade und Schreibrechte pruefen.

## 8) Tipps fuer gutes Clustering

- Hoehere `--steps` fuer bessere Clusterbildung.
- Groesseres Grid (`--size`) wenn viele Payloads vorhanden sind.
- `--agents` groesser als Anzahl Tabellen * 50 ist ein guter Start.

---

Ende.
