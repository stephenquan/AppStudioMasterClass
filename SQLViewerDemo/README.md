# Introduction to Sql Beta in AppStudio 2.0

## Summary

Sql was introduced as a Beta feature in the AppFramework as part of AppStudio 2.0. What this means is we are looking forward to your feedback, use cases and what you want us to improve on. Future changes may affect your app and may require additional changes to support it.

In AppStudio 2.0 you can:
 - read/write SQLite databases
 - read ODBC databases (via SQLite virtual tables)
 - read Shapefiles (via SQLite virtual tables)
 - read DBF files (via SQLite virtual tables)
 - read CSV files (via SQLite virtual tables)
 - read/write WKB point geometry
 - read WKB line and polygon geometry
 - read Shapefile geometry

To get a taste of the Sql support, we will be covering the following.

1. Minimal app - open a SQLite database

2. Running queries

3. Error handling

4. Looping through results

5. Prepared and parameterized queries

6. Autoincrement field

7. SqlQueryModel

## 1. Minimal app - open a SQLite database

This minimal working sample opens a SQLite database in your home directory called ArcGIS/Data/Sql/sample.sqlite. The code works on all platforms. On Android, Linux, Windows and Mac platform, the database created and can be access by other apps. On iOS, the database will be sandboxed to your application, i.e. only your application may access it.

```qml
import QtQuick 2.8
import ArcGIS.AppFramework 1.0
import ArcGIS.AppFramework.Sql 1.0

Item {
  width: 640
  height: 480

  FileFolder {
    id: fileFolder
    path: "~/ArcGIS/Data/Sql"
  }

  SqlDatabase {
    id: db
    databaseName: fileFolder.filePath("sample.sqlite")
  }

  Component.onCompleted: {
    fileFolder.makeFolder();
    db.open();
  }
}
```

## 2. Running queries

To run a SQL query, we use the SqlDatabase's query method. This function is overloaded, i.e. there are multiple, very useful ways of calling query. The most simplest is passing in a single string parameter and that query will be prepared and executed all in one go.

```qml
var query = db.query("SELECT COUNT(*) as Count FROM sqlite_master");
if (query.first()) {
  console.log(query.values.Count);
  query.finish();
}
```

## 3. Error handling

If the query failed, the error parameter will be not null. You can test for it, and, if it exists, it will be set to an JSON error object.

```qml
var query = db.query("CRAP");
if (query.error) {
  console.log(JSON.stringify(query.error, undefined, 2));
} else {
  // success
}
```

Output:

```
{
  "isValid": true,
  "type": 2,
  "databaseText": "near \"CRAP\": syntax error",
  "driverText": "Unable to execute statement",
  "text": "near \"CRAP\": syntax error Unable to execute statement",
  "nativeErrorCode": "1"
}
```

## 4. Looping through results

If your query is a select statement, it will return data via the values JSON object property. The values property will contain values corresponding to one row of results. To get all the results we need to access them in a loop. Note that when we iterate through results, it's always important to call finish(). If you forget, the query could lock the database in an unfinished transaction which may prevent future operations such as DROP TABLE.

```qml
var ok = query.first();
while (ok) {
  var id = query.values.RoadID;
  var name = query.values.RoadName;
  var rdtype = query.values.RoadType;
  ok = query.next();
}
ok.finish();
```

Output:

```
qml: {"RoadID":1,"RoadName":"Coventry","RoadType":"St"}
qml: {"RoadID":2,"RoadName":"Sturt","RoadType":"St"}
qml: {"RoadID":3,"RoadName":"Kings","RoadType":"Way"}
```

## 5. Prepared and parametized queries

A number of commands have been overloaded to support parametized syntax. Using parametized queries diligently can stop accidental bugs or malicious attacks via SQL injection. You can bind to a parameter via name (e.g. ":name") or via position (e.g. "?"). In practice, I always recommend binding parameters by name, because it's stricter and safer. Parameterized queries go well with prepared queries. This is when you offer one SQL statement for repeated execution. The following shows how you can use this approach to populate a table.

```qml
var insert = db.query();
insert.prepare("INSERT INTO Roads (RoadName, RoadType) VALUES (:name, :type)");
insert.executePrepared( { "name": "Bank", "type": "St" } );
insert.executePrepared( { "name": "Dorcas", "type": "St" } );
```

## 6. Autoincrement field

If your table has an autoincrement field you may want to query its value so that you can use it. This is useful if that field is used in a relationship. i.e. you want to populate a related table using the value of the autoincrement field. The value of the last autoincrement operation is in the insertId property.

```qml
var query = db.query("INSERT INTO Roads (RoadName, RoadType) VALUES ('Park', 'St')");
var roadID = query.insertId;
var query2 = db.query(
    "INSERT INTO Inspections (RoadID, Quality) VALUES (:id, :quality)",
    { "id": roadID, "quality": "good" } );
```

## 7. SqlQueryModel

SqlQueryModel and SqlTableModel are read-only data models for SQL result sets. The following demonstrates how you can populate a TableView using a SqlQueryModel.

```qml
import QtQuick 2.8
import ArcGIS.AppFramework 1.0
import ArcGIS.AppFramework.Sql 1.0

Item {
  width: 640
  height: 480

  TableView {
    id: tableView
	anchors.fill: parent
    TableViewColumn {
      role: "RoadID"
      title: "Road ID"
    }
    TableViewColumn {
      role: "RoadName"
      title: "Road Name"
    }
    TableViewColumn {
      role: "RoadType"
      title: "Road Type"
    }
  }

  FileFolder {
    id: fileFolder
    path: "~/ArcGIS/Data/Sql"
  }

  SqlDatabase {
    id: db
    databaseName: fileFolder.filePath("sample.sqlite")
  }

  Component.onCompleted: {
    fileFolder.makeFolder();
    db.open();
    var queryModel = db.queryModel("SELECT * FROM Roads");
    tableView.model = queryModel;
  }
}
```

Output:

![https://raw.githubusercontent.com/stephenquan/AppStudioMasterClass/master/images/SQLViewer.png](https://raw.githubusercontent.com/stephenquan/AppStudioMasterClass/master/images/SQLViewer.png)
