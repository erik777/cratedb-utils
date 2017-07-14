# Crate Table Design with objects

If you are used to a traditional SQL database, and putting your JSON or XML documents in a string/varchar/clob/blob/text field, you are going to love the `object` datatype in CrateDB.  This tutorial takes you through table creation, and with hands on experience, you learn what you can and cannot do with this capability, and how to do it.  

## Presumptions

You ideally have a working copy of CrateDB running if you want to try the examples.  You might find it a good read even if not in front of your CrateDB CLI (crash), as it is very detailed.  

You will want to read Crate's documentation on [table schema creation](https://crate.io/docs/crate/reference/sql/ddl/basics.html#schemas) and [Data Types](https://crate.io/docs/crate/reference/sql/data_types.html), reading in detail the part on the [object](https://crate.io/docs/crate/reference/sql/data_types.html#object) type.  You should have a basic understand of the difference between `dynamic` and `ignored` objects.  


## Table definition

    CREATE TABLE mydocs (
			doctype string,
			owner_mid string,
			doc object(ignored),
			ext object(ignored),
			meta object(dynamic) as (
				created timestamp
			)
    	) PARTITIONED BY (doctype);

### Sample inserts

	INSERT INTO mydocs(doctype, owner_mid) 
	  VALUES('test', 'me');
	  
	cr> select * from mydocs;
	+------+---------+------+------+-----------+
	|  doc | doctype |  ext | meta | owner_mid |
	+------+---------+------+------+-----------+
	| NULL | test    | NULL | NULL | me        |
	+------+---------+------+------+-----------+
	SELECT 1 row in set (0.002 sec)
	
	INSERT INTO mydocs(doctype, owner_mid, doc) 
	  VALUES('test', 'me', 
	    { sid = 'SERVICE123',
	      qty = 5,
	      ccid = 'cust456',
	      cmid = 'TPC-MI-789' });
	
	cr> select * from mydocs;
	+--------------------------------------------------------------------------+---------+------+------+-----------+
	| doc                                                                      | doctype |  ext | meta | owner_mid |
	+--------------------------------------------------------------------------+---------+------+------+-----------+
	| NULL                                                                     | test    | NULL | NULL | me        |
	| {"ccid": "cust456", "cmid": "TPC-MI-789", "qty": 5, "sid": "SERVICE123"} | test    | NULL | NULL | me        |
	+--------------------------------------------------------------------------+---------+------+------+-----------+
	SELECT 2 rows in set (0.004 sec)
		
	cr> SELECT doc FROM mydocs WHERE NOT doc IS NULL;
	+--------------------------------------------------------------------------+
	| doc                                                                      |
	+--------------------------------------------------------------------------+
	| {"ccid": "cust456", "cmid": "TPC-MI-789", "qty": 5, "sid": "SERVICE123"} |
	+--------------------------------------------------------------------------+
	SELECT 1 row in set (0.002 sec)
	
	cr> SELECT doc['ccid'] FROM mydocs WHERE NOT doc IS NULL;
	+-------------+
	| doc['ccid'] |
	+-------------+
	| cust456     |
	+-------------+
	SELECT 1 row in set (0.002 sec)
		
The `ext` field can be externally provided by a customer.  Ideally, we don't even want to parse it before inserting.  Let's insert it as a JSON string:

    INSERT INTO mydocs(doctype, owner_mid, ext) 
      VALUES('test', 'me', '{"ccid": "cust456", "cmid": "TPC-MI-789", "qty": 5, "sid": "SERVICE123"}');

	cr> SELECT * FROM mydocs WHERE NOT ext IS NULL;
	+------+---------+--------------------------------------------------------------------------+------+-----------+
	|  doc | doctype | ext                                                                      | meta | owner_mid |
	+------+---------+--------------------------------------------------------------------------+------+-----------+
	| NULL | test    | {"ccid": "cust456", "cmid": "TPC-MI-789", "qty": 5, "sid": "SERVICE123"} | NULL | me        |
	+------+---------+--------------------------------------------------------------------------+------+-----------+

We can still query by field values despite not knowing anything about the JSON at the time of the insert:	

	SELECT 1 row in set (0.002 sec)
	cr> SELECT ext['ccid'] FROM mydocs WHERE NOT ext IS NULL;
	+-------------+
	| ext['ccid'] |
	+-------------+
	| cust456     |
	+-------------+
	SELECT 1 row in set (0.002 sec)

What if the JSON is malformed on INSERT?

	cr> INSERT INTO mydocs(doctype, owner_mid, ext) 
	          VALUES('test', 'me', '{"ccid" = "bad456", "cmid" is "bad-MI-789", "qty" = 5, "sid" is missing}');
	SQLActionException[RuntimeException: com.fasterxml.jackson.core.JsonParseException: Unexpected character ('=' (code 61)): was expecting a colon to separate field name and value
	 at [Source: [B@6721bc51; line: 1, column: 10]]

It is does not accept it.  We have to be willing to choose between accepting potentially malformed JSON into a `string` field type instead of an `object`, or being able to query fields in the document.  To ensure malformed JSON does not prevent insert of the entire row, we can catch the error, then insert a null or a JSON error in its place.  

Because we chose `ignored` for the object type for `ext`, we can insert different data types for the same field name without an issue:

    INSERT INTO mydocs(doctype, owner_mid, ext) 
      VALUES('test', 'me', '{"ccid": 56, "cmid": false, "qty": "60", "sid": null}');
      
This works as expected      
      
	cr> SELECT * FROM mydocs WHERE ext IS NOT NULL;
	+------+---------+--------------------------------------------------------------------------+------+-----------+
	|  doc | doctype | ext                                                                      | meta | owner_mid |
	+------+---------+--------------------------------------------------------------------------+------+-----------+
	| NULL | test    | {"ccid": 56, "cmid": false, "qty": "60", "sid": null}                    | NULL | me        |
	| NULL | test    | {"ccid": "cust456", "cmid": "TPC-MI-789", "qty": 5, "sid": "SERVICE123"} | NULL | me        |
	+------+---------+--------------------------------------------------------------------------+------+-----------+
	SELECT 2 rows in set (0.003 sec)

However, there is an issue when trying to query a number when there is a text value in one of the rows
	
	cr> SELECT * FROM mydocs WHERE ext['ccid'] = 56;
	SQLActionException[SQLParseException: Cannot cast 'cust456' to type long]
	
But, you can query a string value even though there is a number
	
	cr> SELECT * FROM mydocs WHERE ext['ccid'] = 'cust456';
	+------+---------+--------------------------------------------------------------------------+------+-----------+
	|  doc | doctype | ext                                                                      | meta | owner_mid |
	+------+---------+--------------------------------------------------------------------------+------+-----------+
	| NULL | test    | {"ccid": "cust456", "cmid": "TPC-MI-789", "qty": 5, "sid": "SERVICE123"} | NULL | me        |
	+------+---------+--------------------------------------------------------------------------+------+-----------+
	SELECT 1 row in set (0.002 sec)
	
This can be resolved by simply using a string as the input to the query.  Event the numeric value can match

	cr> SELECT * FROM mydocs WHERE ext['qty'] = '5';
	+------+---------+--------------------------------------------------------------------------+------+-----------+
	|  doc | doctype | ext                                                                      | meta | owner_mid |
	+------+---------+--------------------------------------------------------------------------+------+-----------+
	| NULL | test    | {"ccid": "cust456", "cmid": "TPC-MI-789", "qty": 5, "sid": "SERVICE123"} | NULL | me        |
	+------+---------+--------------------------------------------------------------------------+------+-----------+
	SELECT 1 row in set (0.004 sec)
	
### Booleans

Similar issue with mixed boolean types

	cr> SELECT * FROM mydocs WHERE ext['cmid'] = true;
	SQLActionException[SQLParseException: Cannot cast 'TPC-MI-789' to type boolean]
	
While putting the boolean value in a string does not produce an error, it also fails to match our insert where we set it to false:

	cr> SELECT * FROM mydocs WHERE ext['cmid'] = 'true';
	+-----+---------+-----+------+-----------+
	| doc | doctype | ext | meta | owner_mid |
	+-----+---------+-----+------+-----------+
	+-----+---------+-----+------+-----------+
	SELECT 0 rows in set (0.003 sec)
	cr> SELECT * FROM mydocs WHERE ext['cmid'] = 'false';
	+-----+---------+-----+------+-----------+
	| doc | doctype | ext | meta | owner_mid |
	+-----+---------+-----+------+-----------+
	+-----+---------+-----+------+-----------+
	SELECT 0 rows in set (0.003 sec)

You can CAST or TRY_CAST.  In our case, TRY_CAST is required

	cr> SELECT ext['cmid'], CAST(ext['cmid'] as boolean) FROM mydocs;
	SQLActionException[SQLParseException: Cannot cast 'TPC-MI-789' to type boolean]
	
	cr> SELECT ext['cmid'], TRY_CAST(ext['cmid'] as boolean) FROM mydocs;
	+-------------+----------------------------------+
	| ext['cmid'] | TRY_CAST(ext['cmid'] AS boolean) |
	+-------------+----------------------------------+
	| NULL        | NULL                             |
	| NULL        | NULL                             |
	| FALSE       | FALSE                            |
	| TPC-MI-789  | NULL                             |
	| NULL        | NULL                             |
	| NULL        | NULL                             |
	+-------------+----------------------------------+
	SELECT 6 rows in set (0.003 sec)
	
	cr> SELECT * FROM mydocs WHERE TRY_CAST(ext['cmid'] as boolean) = false;
	+------+---------+-------------------------------------------------------+------+-----------+
	|  doc | doctype | ext                                                   | meta | owner_mid |
	+------+---------+-------------------------------------------------------+------+-----------+
	| NULL | test    | {"ccid": 56, "cmid": false, "qty": "60", "sid": null} | NULL | me        |
	+------+---------+-------------------------------------------------------+------+-----------+
	SELECT 1 row in set (0.005 sec)

What if you want to treat nulls as false, selecting all that are not true, including those missing the field?  

	cr> SELECT count(*) FROM mydocs WHERE TRY_CAST(ext['cmid'] as boolean) = FALSE OR TRY_CAST(ext['cmid'] as boolean) IS NULL;
	+----------+
	| count(*) |
	+----------+
	|        6 |
	+----------+
	SELECT 1 row in set (0.003 sec)
	

What if booleans are all valid, but only some have the value?  Prior inserts do not have our new field, "realboolean".  

    INSERT INTO mydocs(doctype, owner_mid, ext) 
      VALUES('test', 'me', '{"ccid": "cust456", "realboolean": true}');
    INSERT INTO mydocs(doctype, owner_mid, ext) 
      VALUES('test', 'me', '{"ccid": "not456", "realboolean": false}');
      
	cr> SELECT * FROM mydocs WHERE ext['realboolean'] = true;
	+------+---------+------------------------------------------+------+-----------+
	|  doc | doctype | ext                                      | meta | owner_mid |
	+------+---------+------------------------------------------+------+-----------+
	| NULL | test    | {"ccid": "cust456", "realboolean": true} | NULL | me        |
	+------+---------+------------------------------------------+------+-----------+
	SELECT 1 row in set (0.010 sec)
	cr> SELECT * FROM mydocs WHERE ext['realboolean'] = false;
	+------+---------+------------------------------------------+------+-----------+
	|  doc | doctype | ext                                      | meta | owner_mid |
	+------+---------+------------------------------------------+------+-----------+
	| NULL | test    | {"ccid": "not456", "realboolean": false} | NULL | me        |
	+------+---------+------------------------------------------+------+-----------+
	SELECT 1 row in set (0.010 sec)

Both true and false queries work without error.  But, both only return rows that have this field defined.  

Sometimes putting a boolean in a string in your predicate can not have expected results:
	
	cr> SELECT * FROM mydocs WHERE ext['realboolean'] = 'false';
	+-----+---------+-----+------+-----------+
	| doc | doctype | ext | meta | owner_mid |
	+-----+---------+-----+------+-----------+
	+-----+---------+-----+------+-----------+
	SELECT 0 rows in set (0.004 sec)
	
	cr> SELECT * FROM mydocs WHERE ext['realboolean'] = false;
	+------+---------+------------------------------------------+------+-----------+
	|  doc | doctype | ext                                      | meta | owner_mid |
	+------+---------+------------------------------------------+------+-----------+
	| NULL | test    | {"ccid": "not456", "realboolean": false} | NULL | me        |
	+------+---------+------------------------------------------+------+-----------+
	SELECT 1 row in set (0.002 sec)

This does not appear to be an issue when you CAST

	cr> SELECT * FROM mydocs WHERE CAST(ext['realboolean'] as boolean) = 'false';
	+------+---------+------------------------------------------+------+-----------+
	|  doc | doctype | ext                                      | meta | owner_mid |
	+------+---------+------------------------------------------+------+-----------+
	| NULL | test    | {"ccid": "not456", "realboolean": false} | NULL | me        |
	+------+---------+------------------------------------------+------+-----------+
	SELECT 1 row in set (0.003 sec)
	cr> SELECT * FROM mydocs WHERE CAST(ext['realboolean'] as boolean) = false;
	+------+---------+------------------------------------------+------+-----------+
	|  doc | doctype | ext                                      | meta | owner_mid |
	+------+---------+------------------------------------------+------+-----------+
	| NULL | test    | {"ccid": "not456", "realboolean": false} | NULL | me        |
	+------+---------+------------------------------------------+------+-----------+
	SELECT 1 row in set (0.002 sec)


### Limiting based on field's existence

How do we separate data that has our 'realboolean' field from that which doesn't?  In this case, using IS NULL does not work

	cr> SELECT COUNT(*) FROM mydocs WHERE ext['realboolean'] IS NULL;
	+----------+
	| count(*) |
	+----------+
	|        6 |
	+----------+
	SELECT 1 row in set (0.020 sec)
	cr> SELECT COUNT(*) FROM mydocs WHERE ext['realboolean'] IS NOT NULL;
	+----------+
	| count(*) |
	+----------+
	|        0 |
	+----------+
	SELECT 1 row in set (0.003 sec)

However, you can CAST the field to boolean.  If it does not exist, then it will be null.

	cr> SELECT * FROM mydocs WHERE CAST(ext['realboolean'] as boolean) IS NOT NULL;
	+------+---------+------------------------------------------+------+-----------+
	|  doc | doctype | ext                                      | meta | owner_mid |
	+------+---------+------------------------------------------+------+-----------+
	| NULL | test    | {"ccid": "cust456", "realboolean": true} | NULL | me        |
	| NULL | test    | {"ccid": "not456", "realboolean": false} | NULL | me        |
	+------+---------+------------------------------------------+------+-----------+
	SELECT 2 rows in set (0.004 sec)
	
	cr> SELECT * FROM mydocs WHERE CAST(ext['realboolean'] as boolean) IS NULL;
	+--------------------------------------------------------------------------+---------+--------------------------------------------------------------------------+------+-----------+
	| doc                                                                      | doctype | ext                                                                      | meta | owner_mid |
	+--------------------------------------------------------------------------+---------+--------------------------------------------------------------------------+------+-----------+
	| NULL                                                                     | test    | NULL                                                                     | NULL | me        |
	| {"ccid": "cust456", "cmid": "TPC-MI-789", "qty": 5, "sid": "SERVICE123"} | test    | NULL                                                                     | NULL | me        |
	| NULL                                                                     | test    | {"ccid": 56, "cmid": false, "qty": "60", "sid": null}                    | NULL | me        |
	| NULL                                                                     | test    | {"ccid": "cust456", "cmid": "TPC-MI-789", "qty": 5, "sid": "SERVICE123"} | NULL | me        |
	+--------------------------------------------------------------------------+---------+--------------------------------------------------------------------------+------+-----------+
	SELECT 4 rows in set (0.002 sec)


### Dynamic fields
Our `meta` field is `dynamic` because we want it to be consistent (enforced data types) while still having the ability to add fields via our application without doing an explicit schema change.  

So far, nothing we did added new fields to our schema since `doc` and `ext` both used `ignored`.

	cr> select column_name, data_type, column_default, is_nullable from information_schema.columns where table_name = 'mydocs';
	+-----------------+-----------+----------------+-------------+
	| column_name     | data_type | column_default | is_nullable |
	+-----------------+-----------+----------------+-------------+
	| doc             | object    |           NULL | TRUE        |
	| doctype         | string    |           NULL | TRUE        |
	| ext             | object    |           NULL | TRUE        |
	| meta            | object    |           NULL | TRUE        |
	| meta['created'] | timestamp |           NULL | TRUE        |
	| owner_mid       | string    |           NULL | TRUE        |
	+-----------------+-----------+----------------+-------------+
	SELECT 6 rows in set (0.001 sec)

Let's start clean

	cr> DELETE FROM mydocs;
	DELETE OK, -1 rows affected  (0.038 sec)

	cr> INSERT INTO mydocs (doctype, meta) VALUES (
	      'dynadoc', { created = '2017-01-02T13:14:15' } )
	INSERT OK, 1 row affected  (0.074 sec)

and kick the tires on dynamically created fields

	INSERT INTO mydocs (doctype, meta) VALUES (
		      'dynadoc', { created = '2017-01-02T13:14:15',
		      		newnum = 51, newstr = '51', str2 = 'true', newbool = true } );
	
	cr> select * from mydocs;
	+------+---------+------+-------------------------------------------------------------------------------------------+-----------+
	|  doc | doctype |  ext | meta                                                                                      | owner_mid |
	+------+---------+------+-------------------------------------------------------------------------------------------+-----------+
	| NULL | dynadoc | NULL | {"created": 1483362855000}                                                                |      NULL |
	| NULL | dynadoc | NULL | {"created": 1483362855000, "newbool": true, "newnum": 51, "newstr": "51", "str2": "true"} |      NULL |
	+------+---------+------+-------------------------------------------------------------------------------------------+-----------+
	SELECT 2 rows in set (0.002 sec)

	cr> select column_name, data_type, column_default, is_nullable from information_schema.columns where table_name = 'mydocs' and co
	    lumn_name like 'meta%';
	+-----------------+-----------+----------------+-------------+
	| column_name     | data_type | column_default | is_nullable |
	+-----------------+-----------+----------------+-------------+
	| meta            | object    |           NULL | TRUE        |
	| meta['created'] | timestamp |           NULL | TRUE        |
	| meta['newbool'] | boolean   |           NULL | TRUE        |
	| meta['newnum']  | long      |           NULL | TRUE        |  
	| meta['newstr']  | string    |           NULL | TRUE        |
	| meta['str2']    | string    |           NULL | TRUE        |
	+-----------------+-----------+----------------+-------------+
	SELECT 6 rows in set (0.002 sec)

We can see it added our 4 new fields wtih the types we expected.  The rules are basically that after your initial insert, your following inserts need to match the types of the new fields.  `newbool` must always be a valid boolean value.  

Since the use case here is extending the table schema as the application changes, adhering to the same data type for each insert should not be an issue.  	

#### Objects in Dynamic Objects

How far can we go here?  Can we use our doc/doctype convention within a dynamic object?  Let's give it a whirl.  

	INSERT INTO mydocs (doctype, meta) VALUES (
		      'dynadoc', { created = '2017-01-02T13:14:15',
		      		doctype = 'attributes', doc = { color = 'blue', size = 'large' }
		      		 } );
	INSERT INTO mydocs (doctype, meta) VALUES (
		      'dynadoc', { created = '2017-01-02T13:14:15',
		      		doctype = 'shipping', doc = { ups = 14.50, fedex = 19.75 }
		      		 } );
	
Both inserted without a problem.  But, won't this just merge our two docs together since both were called doc?  

	cr> SELECT * FROM mydocs WHERE doctype = 'dynadoc';
	+------+---------+------+------------------------------------------------------------------------------------------------+-----------+
	|  doc | doctype |  ext | meta                                                                                           | owner_mid |
	+------+---------+------+------------------------------------------------------------------------------------------------+-----------+
	| NULL | dynadoc | NULL | {"created": 1483362855000}                                                                     |      NULL |
	| NULL | dynadoc | NULL | {"created": 1483362855000, "newbool": true, "newnum": 51, "newstr": "51", "str2": "true"}      |      NULL |
	| NULL | dynadoc | NULL | {"created": 1483362855000, "doc": {"color": "blue", "size": "large"}, "doctype": "attributes"} |      NULL |
	| NULL | dynadoc | NULL | {"created": 1483362855000, "doc": {"fedex": 19.75, "ups": 14.5}, "doctype": "shipping"}        |      NULL |
	+------+---------+------+------------------------------------------------------------------------------------------------+-----------+
	
	cr> select column_name, data_type, column_default, is_nullable from information_schema.columns where table_name = 'mydocs' and column_name like 'meta[_doc%';
	+----------------------+-----------+----------------+-------------+
	| column_name          | data_type | column_default | is_nullable |
	+----------------------+-----------+----------------+-------------+
	| meta['doc']          | object    |           NULL | TRUE        |
	| meta['doc']['color'] | string    |           NULL | TRUE        |
	| meta['doc']['fedex'] | float     |           NULL | TRUE        |
	| meta['doc']['size']  | string    |           NULL | TRUE        |
	| meta['doc']['ups']   | float     |           NULL | TRUE        |
	| meta['doctype']      | string    |           NULL | TRUE        |
	+----------------------+-----------+----------------+-------------+
	SELECT 6 rows in set (0.001 sec)

The short answer is yes, `doc` now has 4 possible columns.  This isn't necessarily an issue unless one `doctype` uses a different datatype for the same column name.  Perhaps one `doctype` uses an ordinal number for color instead of text.  

To solve that problem, let's try using the `doctype` value as the name of the document.

	INSERT INTO mydocs (doctype, meta) VALUES (
		      'flexdoc', { created = '2017-01-02T13:14:15',
		      		doctype = 'attributes', attributes = { color = 'blue', size = 'large' }
		      		 } );
	INSERT INTO mydocs (doctype, meta) VALUES (
		      'flexdoc', { created = '2017-01-02T13:14:15',
		      		doctype = 'shipping', shipping = { ups = 14.50, fedex = 19.75 }
		      		 } );

Your application would first look at the doctype.  It would then use this as the column name to pull the document itself.  

	cr> select * from mydocs where doctype = 'flexdoc';
	+------+---------+------+-------------------------------------------------------------------------------------------------------+-----------+
	|  doc | doctype |  ext | meta                                                                                                  | owner_mid |
	+------+---------+------+-------------------------------------------------------------------------------------------------------+-----------+
	| NULL | flexdoc | NULL | {"created": 1483362855000, "doctype": "shipping", "shipping": {"fedex": 19.75, "ups": 14.5}}          |      NULL |
	| NULL | flexdoc | NULL | {"attributes": {"color": "blue", "size": "large"}, "created": 1483362855000, "doctype": "attributes"} |      NULL |
	+------+---------+------+-------------------------------------------------------------------------------------------------------+-----------+
	SELECT 2 rows in set (0.003 sec)
	
	cr> select meta['doctype'], meta['shipping'], meta['shipping']['fedex'] - meta['shipping']['ups'] savings from mydocs where doctype = 'flexdoc' and meta['doctype'] = 'shipping';
	+-----------------+-------------------------------+---------+
	| meta['doctype'] | meta['shipping']              | savings |
	+-----------------+-------------------------------+---------+
	| shipping        | {"fedex": 19.75, "ups": 14.5} |    5.25 |
	+-----------------+-------------------------------+---------+
	SELECT 1 row in set (0.003 sec)

How do our new column definitions look now?  

cr> select column_name, data_type, column_default, is_nullable from information_schema.columns where table_name = 'mydocs' and column_name like 'meta[_a%' or column_name like 'meta[_sh%';
+-----------------------------+-----------+----------------+-------------+
| column_name                 | data_type | column_default | is_nullable |
+-----------------------------+-----------+----------------+-------------+
| meta['attributes']          | object    |           NULL | TRUE        |
| meta['attributes']['color'] | string    |           NULL | TRUE        |
| meta['attributes']['size']  | string    |           NULL | TRUE        |
| meta['shipping']            | object    |           NULL | TRUE        |
| meta['shipping']['fedex']   | float     |           NULL | TRUE        |
| meta['shipping']['ups']     | float     |           NULL | TRUE        |
+-----------------------------+-----------+----------------+-------------+
SELECT 6 rows in set (0.002 sec)

Looks great!  While we didn't giving shipping a color, you can see that if we did, it could now have its own datatype.  

Of course, values in the `color` for a document would have to be text (or internally castable to text) going forward for new `attribute` document inserts.  That rule still applies for that `column_name`.  It just wouldn't constrain the datatype in the other document types.  
	 
## Crate Reference Documentation

[Table Basics - Schema](https://crate.io/docs/crate/reference/sql/ddl/basics.html#schemas)
[Crate Data Types](https://crate.io/docs/crate/reference/sql/data_types.html#sql-ddl-datatypes)
