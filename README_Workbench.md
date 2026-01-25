# README_Workbench.md

# MycoDB Workbench + AdventureWorks (Micro-Swarm)

Dieses Handbuch ist **speziell** auf die AdventureWorks-MySQL-Datenbank im `build\Release` Ordner angepasst.
Es erklaert den kompletten Workflow: **SQL-Dump -> .myco**, **CLI/Shell** und **GUI-Workbench**.

Hinweis: Umlaute werden in ASCII geschrieben (ae/oe/ue), damit die Datei plattformneutral bleibt.

---

## 1) Quick Start (AdventureWorks im Release-Ordner)

### 1.1 Dateien (bereit im `build\Release`)

- `build\Release\micro_swarm.exe`
- `build\Release\micro_swarm.dll`
- `build\Release\AdventureWorks-MySQL-with-FKs.sql`
- `build\Release\AdventureWorks-MySQL-with-FKs.myco`
- `build\Release\AdventureWorks-MySQL-with-FKs.ppm`

### 1.2 Schnellstart: Shell

```powershell
.\build\Release\micro_swarm.exe --mode db_shell --db .\build\Release\AdventureWorks-MySQL-with-FKs.myco --db-radius 1000
```

Beispiel-Session:
```
myco> tables
myco> schema product
myco> sql SELECT ProductID,Name,ListPrice FROM product ORDER BY ListPrice DESC LIMIT 5
```

### 1.3 Schnellstart: GUI Workbench

1) **API laden**
   - API-Pfad: `build\Release\micro_swarm.dll`
2) **DB oeffnen**
   - DB-Pfad: `build\Release\AdventureWorks-MySQL-with-FKs.myco`
3) **Query ausfuehren**
   - z.B. `SELECT ProductID,Name,ListPrice FROM product LIMIT 50`

---

## 2) Datenbank-Workflow (Kurzuebersicht)

1) **SQL-Dump importieren** (`db_ingest`)
   - SQL wird eingelesen, Payloads werden erzeugt und per Schwarm geclustert.
   - Ergebnis ist eine `.myco` Datei (Snapshot).

2) **Abfragen ausfuehren** (`db_query`)
   - Fuehrt SQL-Abfrage auf dem Snapshot aus.
   - Nutzt raeumliche Naehe (Radius) und faellt sonst auf Feldsuche zurueck.

3) **Interaktive Shell** (`db_shell`)
   - Direkte Arbeit mit Shell-Kommandos und SQL-Light.
   - Schreiboperationen landen im Delta-Store und koennen spaeter gemerged werden.

4) **GUI Workbench**
   - Komfortable Oberflaeche fuer Abfragen, Fokussteuerung, Paging, Export und Delta-Management.

---

## 3) Schema-Ueberblick (AdventureWorks MySQL)

Die Datenbank enthaelt **70 Tabellen**. Hier die wichtigsten Bereiche:

- **Sales**: `salesorderheader`, `salesorderdetail`, `salesorderheadersalesreason`, `salesreason`, `salesperson`, `salesterritory`, `shipmethod`
- **Customers/People**: `customer`, `contact`, `individual`, `store`, `storecontact`, `customeraddress`, `address`, `stateprovince`, `countryregion`
- **Products**: `product`, `productsubcategory`, `productcategory`, `productmodel`, `productdescription`, `productreview`, `productinventory`, `productcosthistory`
- **Purchasing**: `purchaseorderheader`, `purchaseorderdetail`, `vendor`, `vendoraddress`, `vendorcontact`
- **Manufacturing**: `billofmaterials`, `workorder`, `workorderrouting`, `scrapreason`, `location`, `shift`
- **Reference**: `unitmeasure`, `currency`, `currencyrate`, `culture`, `salestaxrate`

Vollstaendige Tabellenliste (70):
```
address,addresstype,awbuildversion,billofmaterials,contact,contactcreditcard,contacttype,countryregion,countryregioncurrency,creditcard,culture,currency,currencyrate,customer,customeraddress,databaselog,department,document,employee,employeeaddress,employeedepartmenthistory,employeepayhistory,errorlog,illustration,individual,jobcandidate,location,product,productcategory,productcosthistory,productdescription,productdocument,productinventory,productlistpricehistory,productmodel,productmodelillustration,productmodelproductdescriptionculture,productphoto,productproductphoto,productreview,productsubcategory,productvendor,purchaseorderdetail,purchaseorderheader,salesorderdetail,salesorderheader,salesorderheadersalesreason,salesperson,salespersonquotahistory,salesreason,salestaxrate,salesterritory,salesterritoryhistory,scrapreason,shift,shipmethod,shoppingcartitem,specialoffer,specialofferproduct,stateprovince,store,storecontact,transactionhistory,transactionhistoryarchive,unitmeasure,vendor,vendoraddress,vendorcontact,workorder,workorderrouting
```

---

## 4) Wichtige Tabellen und Kernspalten

### 4.1 Sales

`salesorderheader`
- `SalesOrderID`, `OrderDate`, `CustomerID`, `ContactID`, `TerritoryID`
- `SubTotal`, `TaxAmt`, `Freight`, `TotalDue`

`salesorderdetail`
- `SalesOrderID`, `SalesOrderDetailID`, `ProductID`
- `OrderQty`, `UnitPrice`, `UnitPriceDiscount`, `LineTotal`

### 4.2 Customer / Contact

`customer`
- `CustomerID`, `TerritoryID`, `AccountNumber`, `CustomerType`

`contact`
- `ContactID`, `FirstName`, `LastName`, `EmailAddress`, `Phone`

`store`
- `CustomerID`, `Name`, `SalesPersonID`

`customeraddress`
- `CustomerID`, `AddressID`, `AddressTypeID`

`address`
- `AddressID`, `AddressLine1`, `City`, `StateProvinceID`, `PostalCode`

### 4.3 Product

`product`
- `ProductID`, `Name`, `ProductNumber`, `ListPrice`
- `ProductSubcategoryID`, `ProductModelID`

`productsubcategory`
- `ProductSubcategoryID`, `ProductCategoryID`, `Name`

`productcategory`
- `ProductCategoryID`, `Name`

---

## 5) SQL-Datei benutzen (db_ingest)

### 5.1 Beispiel: SQL -> .myco

Wenn du die DB neu clustern willst (optional):

```powershell
.\build\Release\micro_swarm.exe --mode db_ingest \
  --input .\build\Release\AdventureWorks-MySQL-with-FKs.sql \
  --steps 6000 \
  --agents 1024 \
  --size 512 \
  --output .\build\Release\AdventureWorks-MySQL-with-FKs.myco \
  --db-dump .\build\Release\AdventureWorks-MySQL-with-FKs.ppm \
  --db-dump-scale 6
```

Hinweise:
- SQL enthaelt `CREATE DATABASE adventureworks` und `USE adventureworks`.
- MycoDB arbeitet mit den Tabellennamen ohne Schema.
- Mehr Schritte/Agenten -> bessere Cluster, aber laengerer Lauf.

---

## 6) Abfragen per CLI (db_query)

### 6.1 Beispiel

```powershell
.\build\Release\micro_swarm.exe --mode db_query \
  --db .\build\Release\AdventureWorks-MySQL-with-FKs.myco \
  --query "SELECT SalesOrderID,OrderDate,TotalDue FROM salesorderheader ORDER BY OrderDate DESC LIMIT 10" \
  --db-radius 1000
```

Tipps:
- `--db-radius` steuert die lokale Suche im Raum.
- Bei zu kleinem Radius koennen Treffer ausbleiben.

---

## 7) Interaktive Shell (db_shell)

### 7.1 Start

```powershell
.\build\Release\micro_swarm.exe --mode db_shell --db .\build\Release\AdventureWorks-MySQL-with-FKs.myco --db-radius 1000
```

### 7.2 Basisbefehle

- `help`                  -> Befehlsliste
- `tables`                -> Tabellenliste
- `schema <table>`        -> Spaltenliste
- `stats`                 -> Status/Infos
- `format table|csv|json` -> Ausgabeformat
- `limit <n|off>`         -> Default-Limit fuer Shell/SQL

### 7.3 Shell-Style Queries

```
product 1
salesorderheader 43659
customer 10
```

### 7.4 SQL-Light in der Shell

```
sql SELECT ProductID,Name,ListPrice FROM product ORDER BY ListPrice DESC LIMIT 5
sql SELECT SalesOrderID,OrderDate,TotalDue FROM salesorderheader WHERE CustomerID=10 LIMIT 10
sql SELECT SalesOrderID,ProductID,OrderQty,LineTotal FROM salesorderdetail WHERE SalesOrderID=43659 LIMIT 10
```

### 7.5 Fokus und Radius

- `goto <payload_id>` setzt einen Fokuspunkt.
- `radius <n>` aendert den Radius.
- Alle folgenden Anfragen nutzen den Fokus als Zentrum.

### 7.6 Delta-Store (Schreiben)

- `INSERT/UPDATE/DELETE` schreiben in den Delta-Store.
- `delta` zeigt offene Writes/Tombstones.
- `merge` schreibt dauerhaft ein und re-clustert.
- `undo` macht die letzte Delta-Aktion rueckgaengig.

### 7.7 LIMIT-Warnung

Die Shell warnt bei `SELECT *` ohne `LIMIT/OFFSET`, da grosse Tabellen sehr langsam sein koennen.
Empfehlung: immer `LIMIT` oder Paging nutzen.

---

## 8) Beispiel-Queries fuer AdventureWorks

### 8.1 Sales: letzte Bestellungen

```sql
SELECT SalesOrderID, OrderDate, TotalDue
FROM salesorderheader
ORDER BY OrderDate DESC
LIMIT 10;
```

### 8.2 Sales + Products (Order-Details)

```sql
SELECT d.SalesOrderID, d.ProductID, p.Name, d.OrderQty, d.LineTotal
FROM salesorderdetail d
JOIN product p ON p.ProductID = d.ProductID
WHERE d.SalesOrderID = 43659
LIMIT 20;
```

### 8.3 Umsatz nach Produktkategorie

```sql
SELECT c.Name AS Category, SUM(d.LineTotal) AS Revenue
FROM salesorderdetail d
JOIN product p ON p.ProductID = d.ProductID
JOIN productsubcategory s ON s.ProductSubcategoryID = p.ProductSubcategoryID
JOIN productcategory c ON c.ProductCategoryID = s.ProductCategoryID
GROUP BY c.Name
ORDER BY Revenue DESC
LIMIT 10;
```

### 8.4 Kunden je Stadt

```sql
SELECT a.City, COUNT(*) AS CustomerCount
FROM customeraddress ca
JOIN address a ON a.AddressID = ca.AddressID
GROUP BY a.City
ORDER BY CustomerCount DESC
LIMIT 10;
```

### 8.5 Sales nach Territory

```sql
SELECT t.Name AS Territory, SUM(h.TotalDue) AS Revenue
FROM salesorderheader h
JOIN salesterritory t ON t.TerritoryID = h.TerritoryID
GROUP BY t.Name
ORDER BY Revenue DESC
LIMIT 10;
```

### 8.6 Store + Contact

```sql
SELECT s.Name AS Store, c.FirstName, c.LastName, c.EmailAddress
FROM store s
JOIN storecontact sc ON sc.CustomerID = s.CustomerID
JOIN contact c ON c.ContactID = sc.ContactID
LIMIT 20;
```

---

## 9) GUI Workbench nutzen (MycoDB Workbench)

Die Workbench arbeitet direkt mit `micro_swarm.dll` und einer `.myco` Datei.
Die komplette Oberflaeche ist in `docs/workbench.md` dokumentiert. Hier die wichtigsten Punkte:

### 9.1 Verbindung

- **API Pfad:** `build\Release\micro_swarm.dll`
- **DB Pfad:** `build\Release\AdventureWorks-MySQL-with-FKs.myco`

### 9.2 Query-Mode

- **Auto**: erkennt SQL automatisch
- **SQL**: erzwingt SQL-Ausfuehrung
- **Shell**: behandelt Input als Shell-Befehl

### 9.3 Paging

- Auto-Paging wird aktiviert, wenn SQL ohne `LIMIT/OFFSET` ausgefuehrt wird.
- `Page Size` steuert die Groesse.

### 9.4 Fokus-Tools

- **Set Focus**: Fokus ueber `payload_id`
- **Use Selection**: Fokus auf ausgewaehlte Zeile
- **Clear Focus**: Fokus deaktivieren

### 9.5 Delta-Management

- **Undo**: letztes Delta rueckgaengig
- **Merge**: Delta-Store dauerhaft einclustern
- **Diff-Tab**: zeigt Delta- und Tombstone-Infos

### 9.6 Export

- **Export CSV/JSON** Buttons exportieren den aktiven Result-Tab.
- **Export-Tab** erlaubt gezielte Spaltenauswahl.

---

## 10) Troubleshooting (kurz)

- **Keine Treffer**: Radius erhoehen oder Filter lockern.
- **DB: aus**: `.myco` Pfad setzen und DB oeffnen.
- **API: aus**: DLL-Pfad pruefen, API laden.
- **Query ungueltig**: SQL pruefen oder `sql` prefix setzen.

---

## 11) Quellen / Referenzen

AdventureWorks MySQL Dump:
```
https://github.com/tapsey/AdventureWorksMYSQL
```

- `README.md` (Abschnitt **MycoDB**) fuer CLI-Parameter.
- `docs/workbench.md` fuer das komplette GUI-Handbuch.

