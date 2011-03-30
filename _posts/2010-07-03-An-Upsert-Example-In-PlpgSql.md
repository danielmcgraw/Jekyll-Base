---
layout: post
title: An Upsert Example In plpgsql
---

Postgres is a pretty good RDBMS. Its missing something though. Actually its missing a lot of things, but we're going to talk about upserts. No that's not a typo. An upsert is the intersection of an update and an insert. When used it will either update, if the record exists, or insert a new record. I think this should be built in, but sadly it is not. This can be fixed with plpgsql. Below is a simple plpgsql function I wrote. It takes a column name, numeric index (think numeric primary key), and the new data. In a loop it checks to see if we can update, if not then it inserts. You might be wondering what the loop is for. If so, good question. It loops because there is a possibility that the update will fail because there is no record yet and before the upsert's insert happens a matching record is inserted. This will cause the insert to error and if we ended here we would not get the proper data state. This is where the loop comes in. It allows the insert error and then goes back up to the update.

This is not a solution to all upserts, but it is a good basis if you need to write your own. Check out the code below and please ask questions. 

{% highlight sql %}
CREATE FUNCTION upsert_table(text, integer, boolean) RETURNS VOID AS $$
DECLARE 
    res INTEGER;
    col ALIAS FOR $1;
    num ALIAS FOR $2;
    dat ALIAS FOR $3;
BEGIN
    LOOP
	-- First try to update.
	EXECUTE 'UPDATE table' 
		|| ' SET '
		|| quote_ident(col)
	  	|| ' = '
		|| dat
		|| ' WHERE index = '
		|| num
		|| ';';
	-- Check to see if the update affected any rows.
	GET DIAGNOSTICS res = ROW_COUNT;
	IF res = 1 THEN
	   RETURN;
	END IF;
	-- Since its not there we try to insert the key
 	-- Notice if we had a concurrent key insertion we would error
	BEGIN
		EXECUTE 'INSERT INTO table (index, '
			|| quote_ident(col)
			|| ') VALUES ('
			|| num
			|| ', '
			|| dat
			|| ');';
		 RETURN;
	EXCEPTION WHEN unique_violation THEN
	-- Loop and try the update again
	   END;
    END LOOP;
END;
$$ LANGUAGE 'plpgsql';
{% endhighlight %}