# Field of OBJECT(dynamic) missing from SELECT result

A field in object(dynamic) called `core` is included in the CREATE TABLE. When you INSERT with value of `test`, then SELECT `core`, *it is missing from the result*.  E.g., `{"docsubtype": "parse"}` instead of `{"doctype":"test","docsubtype":"parse"}`  But, when you select `core['doctype']`, you get back the value 'test', as expected, suggesting it is "hidden" when querying the OBJECT.  

To reproduce:

	cr> CREATE TABLE mytable (
		core object(dynamic) as (
			doctype string,
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
	