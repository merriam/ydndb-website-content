---
layout: ydndb-article
description: "Join Query"
class: ydndb
title: Join query
introduction: "YDN-DB has facility for joining tables."
article:
  written_on: 2013-11-22
  updated_on: 2013-11-22
  order: 4
collection: ydndb-query
authors:
  - kyawtun
notes:
  console:
    Pages in this sections include the YDN-DB script and some preloaded data and utility functions, so that you follow the sample code in your browser's developer console to see in action.

---

{% wrap content %}

In previous article, [key-joining algorithms](key-joining.html) are explained. We will use these algorithm for joining multiple tables.


### Sample data

Let us load very simple database that is used by Chris Date and Hugh Darwen throughout their books (e.g., [An Introduction To Database Systems](http://c2.com/cgi/wiki?AnIntroductionToDatabaseSystems) and [The Third Manifesto](http://en.wikipedia.org/wiki/The_Third_Manifesto)) for examples.

    var schema = {
      stores: [{
        name: 'Supplier',
        keyPath: 'SID',
        indexes: [{
          name: 'SNAME'
        }, {
          name: 'STATUS'
        }, {
          name: 'CITY'
        }]
      }, {
       name: 'Part',
       keyPath: 'PID',
       indexes: [{
         name: 'PNAME'
       }, {
         name: 'COLOR'
       }, {
         name: 'WEIGHT'
       }, {
         name: 'CITY'
       }]
      }, {
        name: 'Part-Supplier',
        autoIncrement: true,
        indexes: [{
          name: 'SID'
        }, {
          name: 'PID'
        }, {
          name: 'QTY'
        }]
      }]
    };
    var db = new ydn.db.Storage('supplier-part', schema);
    db.put('Supplier', [
      {SID: "S1", SNAME: "Smith", STATUS: 20, CITY: "London"},
      {SID: "S2", SNAME: "Jones", STATUS: 10, CITY: "Paris"},
      {SID: "S3", SNAME: "Blake", STATUS: 30, CITY: "Paris"},
      {SID: "S4", SNAME: "Clark", STATUS: 20, CITY: "London"},
      {SID: "S5", SNAME: "Adams", STATUS: 30, CITY: "Athens"}
    ]);
    db.put('Part', [
      {PID: "P1", PNAME: "Nut", COLOR: "Red", WEIGHT: 12.0, CITY: "London"},
    	{PID: "P2", PNAME: "Bolt", COLOR: "Green", WEIGHT: 17.0, CITY: "Paris"},
    	{PID: "P3", PNAME: "Screw", COLOR: "Blue", WEIGHT: 17.0, CITY: "Oslo"},
    	{PID: "P4", PNAME: "Screw", COLOR: "Red", WEIGHT: 14.0, CITY: "London"},
    	{PID: "P5", PNAME: "Cam", COLOR: "Blue", WEIGHT: 12.0, CITY: "Paris"},
    	{PID: "P6", PNAME: "Cog", COLOR: "Red", WEIGHT: 19.0, CITY: "London"}
    ]);
    db.put('Part-Supplier', [
      {SID: "S1", PID: "P1", QTY: 300},
    	{SID: "S1", PID: "P2", QTY: 200},
    	{SID: "S1", PID: "P3", QTY: 400},
    	{SID: "S1", PID: "P4", QTY: 200},
    	{SID: "S1", PID: "P5", QTY: 100},
    	{SID: "S1", PID: "P6", QTY: 100},
    	{SID: "S2", PID: "P1", QTY: 300},
    	{SID: "S2", PID: "P2", QTY: 400},
    	{SID: "S3", PID: "P2", QTY: 200},
    	{SID: "S4", PID: "P2", QTY: 200},
    	{SID: "S4", PID: "P4", QTY: 300},
    	{SID: "S4", PID: "P5", QTY: 400}
    ]);

## Join operation with full table scan

A full table scan looks through all of the rows in a table – one by one – to find the data that a query is looking for. Obviously, this can cause very slow queries if you have a table with a lot of rows – just imagine how performance-intensive a full table scan would be on a table with millions of rows. Using an index can help prevent full table scans.

There are some scenarios in which a full table scan will still be performed even though an index is present on that table. Let’s go through some of those scenarios.

1. One common scenario is we don't want to index the fields. Full table scan does not require indexing.
2. Use regular expression or complex filtering in query. For example a query like `WHERE NAME LIKE ‘%INTERVIEW%` cannot be used index.
3. Query with multiple ranges.

Let us see how to use full table scan in YDN-DB for query `SELECT * FROM Supplier, Part WHERE Supplier.CITY = Part.CITY`. [`db.scan`](http://dev.yathit.com/api/ydn/db/storage.html#scan) method is used to scan multiple tables at the same time. The following code snippet use nested loop, in which inner table do a full table scan for each outer table record.


    var iter_supplier = new ydn.db.ValueIterator('Supplier');
    var iter_part = new ydn.db.ValueIterator('Part');
    var req = db.scan(function(keys, values) {
      var SID = keys[0];
      var PID = keys[1];
      console.log(SID, PID);
      if (SID && PID) {
        if (values[0].CITY == values[1].CITY) {
          console.log(values[0], values[1]);
        }
        return [undefined, true]; // advance inner loop
      } else if (!PID) {
        return [true, false]; // advance outer loop, restart inner loop
      } else {
        return []; // done
      }
    }, [iter_supplier, iter_part]);

## Index base joining

We can use index for more efficient joining.

    var iter_supplier = new ydn.db.IndexValueIterator('Supplier', 'CITY');
    var iter_part = new ydn.db.IndexValueIterator('Part', 'CITY');
    var req = db.scan(function(keys, values) {
      var SID = keys[0];
      var PID = keys[1];
      console.log(SID, PID);
      if (!SID || !PID) {
        return []; // done
      }
      var cmp = ydn.db.cmp(SID, PID); // compare keys
      if (cmp == 0) {
        console.log(values[0], values[1]);
        return [true, true]; // advance both
      } else if (cmp == 1) {
        return [undefined, SID]; // jump PID cursor to match SID
      } else {
        return [PID, undefined]; // jump SID cursor to match PID
      }
    }, [iter_supplier, iter_part]);

Notice that we only need scanning matching record. The size of records in these two table does not matter, only matching record size are relevant. Index base joining are very scalable.




{% endwrap %}
