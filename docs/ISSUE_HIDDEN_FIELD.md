
does 'doctype' have special meaning in crate?  It is a field in object(dynamic) called `core`, included in CREATE TABLE. I do inserts, with value of `test`.  When I do SELECT `core`, *it is missing from the result*.  E.g., `{"docsubtype": "parse"}` instead of `{"doctype":"test","docsubtype":"parse"}`  But, when I select `core['doctype']`, I get back the value 'test', as it should, suggesting it is "hidden" when querying the OBJECT.  


```cr> select core from mytable;

+-------------------------+
| core                    |
+-------------------------+
| {"docsubtype": "parse"} |
+-------------------------+```

```cr> select core['doctype'] from cloudcollection;
+-----------------+
| core['doctype'] |
+-----------------+
| test            |
+-----------------+```

To reproduce:

	cr> CREATE TABLE mytable (
		core object(dynamic) as (
			doctype string NOT NULL,
			docsubtype string,
			devid string 
		) NOT NULL
		) PARTITIONED BY (core['doctype']) ;
	CREATE OK, 1 row affected  (0.010 sec)
	
	cr> INSERT INTO mytable(core) VALUES ('{"doctype": "test","docsubtype": "parse"}');
	INSERT OK, 1 row affected  (0.076 sec)

	cr> SELECT core FROM mytable;
	+-------------------------+
	| core                    |
	+-------------------------+
	| {"docsubtype": "parse"} |
	+-------------------------+
	SELECT 1 row in set (0.001 sec)
	
	cr> SELECT core['doctype'] FROM mytable;
	+-----------------+
	| core['doctype'] |
	+-----------------+
	| test            |
	+-----------------+
	SELECT 1 row in set (0.003 sec)
	
	cr> DROP TABLE mytable;
	DROP OK, 1 row affected  (0.025 sec)

Removing PARTITIONED BY allows the field to show up:

	cr> CREATE TABLE mytable (
		core object(dynamic) as (
			doctype string NOT NULL,
			docsubtype string,
			devid string 
		) NOT NULL
		) ;
	CREATE OK, 1 row affected  (0.010 sec)
	
	cr> INSERT INTO mytable(core) VALUES ('{"doctype": "test","docsubtype": "parse"}');
	INSERT OK, 1 row affected  (0.076 sec)

	cr> SELECT core FROM mytable;
	+--------------------------------------------+
	| core                                       |
	+--------------------------------------------+
	| {"docsubtype": "parse", "doctype": "test"} |
	+--------------------------------------------+

	SELECT 1 row in set (0.001 sec)
	
	cr> SELECT core['doctype'] FROM mytable;
	+-----------------+
	| core['doctype'] |
	+-----------------+
	| test            |
	+-----------------+
	SELECT 1 row in set (0.003 sec)
	
	cr> DROP TABLE mytable;
	DROP OK, 1 row affected  (0.025 sec)
	