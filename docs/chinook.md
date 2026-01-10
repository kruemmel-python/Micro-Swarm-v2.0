# Chinook: MycoDB Beispiele (Micro-Swarm)

Diese Datei zeigt praxisnahe Beispiele fuer die Chinook-Datenbank mit dem interaktiven `db_shell` Modus.
Alle Beispiele sind kompatibel mit dem aktuellen MycoDB-Parser und der Shell-Syntax.

Ausgangspunkt:

```powershell
.\micro_swarm.exe --mode db_shell --db chinook_optimized.myco --db-radius 15
```

## Shell-Kommandos (Meta)

```
help
focus
radius 25
unfocus
tables
stats
schema Track
sql SELECT TrackId,Name FROM Track WHERE AlbumId=1 ORDER BY TrackId LIMIT 5
exit
```

Kurzerklaerung:
- `help`: zeigt alle Befehle
- `focus`: zeigt aktuellen Fokuspunkt (falls gesetzt)
- `radius N`: setzt den Suchradius fuer alle folgenden Abfragen
- `unfocus`: Fokus entfernen
- `tables`: Tabellenliste
- `stats`: Payload-Counts pro Tabelle
- `schema <table>`: Spaltenliste
- `sql <statement>`: SQL-Light Query
- `exit`: Shell beenden

## 1) Primary-Key Queries (direkte ID)

```
Album 1
Artist 5
Track 12
Genre 3
MediaType 2
Employee 1
Invoice 10
InvoiceLine 20
Playlist 4
PlaylistTrack 15
```

Bedeutung: Zeigt jeweils den Datensatz mit der ID.

## 2) Foreign-Key Queries (Relationen)

```
Album ArtistId=1
Track AlbumId=1
Track GenreId=1
Track MediaTypeId=1
InvoiceLine InvoiceId=1
Invoice CustomerId=1
PlaylistTrack PlaylistId=1
```

Bedeutung: Zeigt alle Datensaetze mit einem Foreign-Key-Wert.

## 3) Navigation per Fokus (goto)

Beispiel: zuerst Album 1 finden, dann den Fokus setzen und lokal suchen.

```
Album 1
goto 1
Track AlbumId=1
```

Hinweis:
- `goto <payload_id>` nutzt die Payload-ID (nicht zwingend die SQL-ID).
- Die Payload-ID wird in jeder Trefferzeile als `id=<...>` ausgegeben.

## 4) Radius-gesteuerte Lokalsuche

Groesserer Radius fuer groessere Cluster:

```
radius 40
Track AlbumId=1
```

Kleiner Radius fuer sehr schnelle Antwort:

```
radius 5
Track AlbumId=1
```

## 4b) Spalten filtern (show)

```
Track AlbumId=1 show TrackId,Name,Composer
```

## 4c) Globale Spaltenabfrage (ohne Tabellenname)

```
TrackId=13
AlbumId=1
```

## 4d) SQL-Light Beispiele

```
sql SELECT TrackId,Name FROM Track WHERE AlbumId=1 ORDER BY TrackId LIMIT 5
sql SELECT AlbumId, COUNT(*) AS C FROM Track GROUP BY AlbumId HAVING C > 5 ORDER BY C DESC LIMIT 10
sql SELECT t.Name, a.Title FROM Track t JOIN Album a ON t.AlbumId=a.AlbumId WHERE t.TrackId=13
sql SELECT Name FROM Artist WHERE Name LIKE 'A%' ORDER BY ArtistId
sql SELECT TrackId FROM Track WHERE AlbumId=1 UNION SELECT TrackId FROM Track WHERE AlbumId=2
sql SELECT TrackId FROM Track WHERE AlbumId=1 UNION ALL SELECT TrackId FROM Track WHERE AlbumId=1
sql SELECT t.Name, a.Title FROM Track t RIGHT JOIN Album a ON t.AlbumId=a.AlbumId WHERE a.AlbumId=1
sql SELECT * FROM Track CROSS JOIN MediaType LIMIT 3
sql SELECT Name FROM Artist WHERE Name REGEXP '^A'
sql SELECT Name FROM Artist WHERE ArtistId IN (SELECT ArtistId FROM Album WHERE AlbumId=1)
sql SELECT Name FROM Artist WHERE EXISTS (SELECT AlbumId FROM Album WHERE AlbumId=1)
sql SELECT Name FROM Artist a WHERE EXISTS (SELECT AlbumId FROM Album WHERE Album.ArtistId=a.ArtistId)
sql SELECT LOWER(Name) AS n FROM Artist ORDER BY n LIMIT 5
sql SELECT TrackId, Name FROM Track ORDER BY 1 LIMIT 5
sql SELECT Name FROM Artist WHERE Name IS NOT NULL LIMIT 3
sql SELECT * FROM (SELECT ArtistId, Name FROM Artist) a WHERE a.ArtistId=1
sql WITH top_albums AS (SELECT AlbumId FROM Track GROUP BY AlbumId HAVING COUNT(*) > 5) SELECT AlbumId FROM top_albums ORDER BY AlbumId
```

## 5) Kombi-Workflow (typisch)

```
Artist 1
Album ArtistId=1
goto 1
Track AlbumId=1
radius 8
Track AlbumId=1
```

## 6) Fallback ohne Spaltenliste (wenn SQL-Dump ohne Column-Names)

Wenn der Dump keine Spaltenliste enthaelt, sind Spalten intern `col0`, `col1`, ...
Dann funktioniert:

```
Album 1
Track col2=1
```

(Die genaue Spaltenposition haengt vom Dump ab.)

---

Hinweis fuer Chinook:
- Tabellen sind kleingeschrieben im Dump, Micro-Swarm ist case-insensitive.
- `PlaylistTrack` ist die Join-Tabelle zwischen Playlist und Track.
- `InvoiceLine` referenziert `InvoiceId`.
- Wenn Spaltennamen fehlen, ingest erneut mit Dump inkl. `CREATE TABLE`.

Ende.
