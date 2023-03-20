# Lab 4

Here we will learn substantially more sophisticated types of queries, dig deeper into SQL, and develop high-level techniques for describing a database schema.

## Table of Contents
- [Back to the tables from Lab 3](#back-to-the-tables-from-lab-3)
  - [Level Two and beyond (`ORDER BY`)](#level-two-and-beyond-order-by)
  - [The `GROUP BY` clause](#the-group-by-clause)
  - [The `HAVING` clause](#the-having-clause)
  - [SubQueries](#subqueries)
    - [The `UNION` keyword](#the-union-keyword)
- [A little more theory](#a-little-more-theory)
  - [Data Models](#data-models)
    - [Entity Relationship Model](#entity-relationship-model)
  - [Relationships between tables](#relationships-between-tables)
  - [Introduction to indexes](#introduction-to-indexes)
- [Some new developments](#some-new-developments)
- [To Do](#to-do)

## Back to the tables from Lab 3

We return to modifying the tables we made in lab 3.  Keeping in mind what you now know about data normalization and anomalies, redesign the `poorDesign` table by splitting it into multiple tables without redundancy.  Preface all your table names with `PD_` so know which ones to look at.

## Level Two and beyond (`ORDER BY`)

The next level of understanding comes from the `ORDER BY` 
clause.  As you hopefully remember from lecture, this controls the order of the entries in the table.  Try these:

```
SELECT * FROM inventory ORDER BY item;
SELECT * FROM inventory order by amount;
```
	
It works with more complicated queries too:

````
SELECT * FROM inventory, prices WHERE inventory.id=prices.id ORDER BY unit;
````
	
Additionally, just on the off chance you have enough entries in your tables for this to make sense, you can order by more than one column:

````
SELECT * FROM inventory, prices WHERE inventory.id=prices.id ORDER BY unit, item;
````

## The `GROUP BY` clause
	
The next clause on our magical mystery tour is the GROUP BY clause (now we're at Fireball level)

SOME functions are designed to work with **aggregates**.  Try these two commands:

```
SELECT * FROM poorDesign;
SELECT comboName,count(*) AS count FROM poorDesign GROUP BY comboCode;
```

There are several aggregate functions, but the SQL Server doesn't make it easy to figure out exactly what they are. You should search in the SQL Server online documents and find a few to try.  

## The `HAVING` clause

Here's something a bit more interesting... try these:

```
SELECT comboCode, count(*) FROM poorDesign WHERE item >=3 GROUP BY comboCode;
SELECT comboCode, count(*) FROM poorDesign GROUP BY comboCode;
```

(you might need to adjust the `WHERE` clause to fit the values in your data... the point is to ensure that SOMETHING gets filtered.)

And of course, by saying that, I've highlighted what I wanted to point out... namely that the `WHERE` clause filters rows **before** grouping.  This means it affects just exactly WHAT gets grouped and can affect the output of the aggregating functions.  

Now what happens if we want to restrict the `GROUPS` that we see?  That's where the `HAVING` clause comes into play.  It controls what GROUPS are displayed.... and even cooler, it can use aggregate functions of its own. Here's a perfectly valid command (which may, or may not, produce anything interesting – depending upon the contents of your tables):

```
SELECT comboCode, count(*) FROM poorDesign GROUP BY comboCode HAVING sum(comboCode)>3;
```

## SubQueries

So... next level are subqueries.  Here's an example:

```
SELECT * FROM inventory WHERE id IN (SELECT item FROM poorDesign);
```
Notice the parenthesis and the `IN` keyword (Note:  `IN` is *not* required to make the subquery work).  Notice that the subquery is in parenthesis.

Read this:  <http://beginner-sql-tutorial.com/sql-subquery.htm>
And also this: <http://en.wikipedia.org/wiki/Correlated_subquery>

Now try this, rather stupid, example of a **correlated subquery**:

```
SELECT (SELECT unit FROM inventory WHERE inventory.id=bob.id) FROM inventory AS bob;
```

The key to understanding what is happening is to realize that for EVERY row in the outer `SELECT` statement the INNER query is run.  In order for this sort of nested command to work correctly the inner `SELECT` needs to return **one value**.  This might make things a bit clearer:

```
SELECT item,unit,amount,(SELECT unit FROM inventory WHERE inventory.id=bob.id) AS sub 
   FROM inventory AS bob;
```

Pay special attention to giving aliases to the table names.  The outer query and inner query need some sort of common reference to be correlated… of course they don't HAVE to be correlated:

```
SELECT item,unit,amount,(SELECT 'bob' ) AS sub FROM inventory AS bob;
```

This sort of thing is useful for including aggregate information:

```
SELECT item,unit,amount,(SELECT avg(amount) FROM inventory WHERE unit=bob.unit GROUP by unit) AS sub FROM inventory AS bob;
```

## The `UNION` keyword

So far we have been looking at various ways to combine columns of tables. We can also use functions that transform the contents of a column, or use a function to calculate a value derived from the value of several different columns:

```
SELECT concat(item,unit) AS together FROM inventory;
```

But what if we want to merge ROWS? That's where `UNION` comes in:

```
SELECT * FROM inventory LEFT JOIN prices ON inventory.id=prices.id;
SELECT * FROM inventory RIGHT JOIN prices ON inventory.id=prices.id;
	
SELECT * FROM inventory LEFT JOIN prices ON inventory.id=prices.id UNION
    SELECT * FROM inventory RIGHT JOIN prices ON inventory.id=prices.id;
```
	
Pretty spiffy isn't it… and it really DOES act like the union… there were a few rows shared by both of the component queries and they didn't show up more than once in the final output.  You can get around this behavior (which can matter sometimes by adding the option keyword ALL:

```
SELECT * FROM inventory LEFT JOIN prices ON inventory.id=prices.id UNION ALL
    SELECT * FROM inventory RIGHT JOIN prices ON inventory.id=prices.id;
```

# A little more theory

Now that we have started to familiarize ourselves with the `SELECT` command and its various clauses and keywords, we are going to want to learn a little bit more theory.

## Data models

When designing a database you will want to have some systematic way of indicating what information the database must hold and the relationships between components of that information.  Many authors call the structure of the database a **database schema**.

### Entity Relationship model

A first pass to accomplishing this is to identify **entities** and **relationships**.  Wikipedia has a nice entry on [the entity-relationship model](http://en.wikipedia.org/wiki/Entity%E2%80%93relationship_model).  This is a level of abstraction at which *attributes*, *keys* and *relations* exist.   Pay close attention to the difference between **entity** and **entity set**.

If I can borrow an analogy from object oriented programming-- entities could be viewed as instances, entity sets as classes (which is a bit of a stretch... but work with me).  Attributes can then be viewed as fields in the class definition.  The idea of *relations* doesn't really fit this analogy without introducing more complications than are useful.  You might also find the [tutorial](http://www.cs.sfu.ca/CourseCentral/354/zaiane/material/notes/Chapter2/node1.html) useful.

#### ER exercise:

* You only need one repository per group but make sure all the members are collaborators.
* Use Google Drawing to model a few entities and relationships that are related to the Point of Sales system (be certain that every member of the group contributes-- I expect to see rectangles, diamonds, and ovals)
* Add the drawing to your repository

## Relationships between tables

One of the most important ideas is about the **type of relationships** between attributes.  These relationships may be:

* one-to-one (1:1)
* one-to-many (1:n)
* many-to-many (m:n)

You might be wondering why there is no many-to-one.  There is-- it's just a one-to-many viewed from the other direction.  Now would be a good time to read the following:

<http://www.techrepublic.com/article/relational-databases-defining-relationships-between-database-tables/>

Now a **primary key** is just a key for a table, and a **foreign key** is a an attribute in one table that corresponds to primary key in another table.  

Many database systems (SQL Server and mySQL included) allow the user to introduce **constraints** into the system.  Read up on [mysql's foreign keys](http://dev.mysql.com/doc/refman/5.6/en/create-table-foreign-keys.html), for an example.  These constraints help maintain **referential integrity**:  consistency in table spread across multiple tables.

For example, suppose you had an `inventory` table with a `vendor_id` attribute that was a foreign key in the `vendor` table.  It would probably be a bad idea if you inserted a record into `inventory` that included an invalid `vendor_id`.  

#### Foreign Key Exercise

Each group should create two tables, one named `fk_A`, the other `fk_B`:

* The first table should have a column named `idA` and `foreignID`.  The second table should have a column named `idB` and `text`.
* The columns `idA` and `idB` should be primary keys, and `foreignID` should be a foreign key that refers to `idB` in `fk_B`.  
* Experiment with creating the tables, inserting records, deleting records, and modifying values.


### Some practice with `UNION`

We have already seen how to use `UNION` to combine the results of two queries with similar structures.  Let's practice.  

First make sure these toy queries don't return anything too long:

```{sql}
SELECT * FROM randA where A BETWEEN 2 AND 9;
SELECT * FROM randB where B BETWEEN 2 AND 9;
```

Now join them into one:

```{sql}
SELECT * FROM randA WHERE A BETWEEN 2 AND 9 
UNION 
SELECT * FROM randB where B BETWEEN 2 AND 9;
```

Now try this:

```{sql}
SELECT * FROM randA WHERE A BETWEEN 2 AND 9 
UNION 
SELECT * FROM randA where A BETWEEN 2 AND 9;

```

The key to understanding that output is to realize that `UNION` truly does mean `UNION` (no repeats
allowed) – in other words the output is, by default, at least in Type 1 Normal Form. 

You can force it to include everything like this:

```{sql}
SELECT * FROM randA where A BETWEEN 2 AND 9 
UNION ALL 
SELECT * FROM randA where A BETWEEN 2 AND 9;
```

If you are using `UNION`, then you are allowed one `ORDER BY` clause and it must come AFTER the last `SELECT` (no
ordering the pieces then joining them together). This will work:

```{sql}
SELECT * FROM randA where A BETWEEN 2 AND 9 
UNION ALL 
SELECT * FROM randA where A BETWEEN 2 AND 9 
ORDER BY B;
```

And it won't even complain about ambiguity because the ORDER BY is acting on the merged records.

## Some new developments

Each database in SQL Server is stored using a **database engine**.  The defaults are a bit different between SQL Server and mySQL, but we aren't going to worry too much about those right now.

Different database engines allow for different capabilities.  The `Azure Data Studio` engine (and descendents), for example, allow for **transactions** (we are going to talk about this in more detail in a later lab).  Transactions allow you to attempt to perform a series of actions.  At the end of the sequence the transaction is either **committed** if it succeeds or **rolled-back** if it fails.  This is an extension of the core idea of **atomicity** and forms one of the pillars of the relational database **ACID** concept (**A**tomicity, **C**onsistency, **I**solation, **D**urability).  

## To Do

Make sure and complete the following:
- [ ] Include the URL pointing to your group's github repository via canvas.  Be certain your canvas groups match your github groups
- [ ] Create the `PD_` tables during your `poorDesign` rewrite.
- [ ] Each student should type up all the samples (NOTE:  copy-and-paste is bad-- just watching is bad:  type them up and talk to your group-mate(s) about what it means).  Nothing to turn in for this-- but a few tables should be generated (see canvas rubric)
- [ ] Read the contents of these links (see the lab for details of what to skim):
   * [SubQueries](http://beginner-sql-tutorial.com/sql-subquery.htm)
   * [Wikipedia page on Correlated subqueries](https://en.wikipedia.org/wiki/Correlated_subquery)
   * [Wikipedia page on ER models](https://en.wikipedia.org/wiki/Entity%E2%80%93relationship_model)
   * [ER tutorial](http://www.cs.sfu.ca/CourseCentral/354/zaiane/material/notes/Chapter2/node1.html)
   * [Relationships in relational databases](http://www.techrepublic.com/article/relational-databases-defining-relationships-between-database-tables/)
   * [Creating tables with a foreign key](http://dev.mysql.com/doc/refman/5.6/en/create-table-foreign-keys.html)
- [ ] Do these tutorials (nothing to turn in-- but do the problems)
- [ ] Do these exercises:
   - [ ] [Entity-Relation Exercise](#er-exercise)
   - [ ] [Foreign Key Exercise](#foreign-key-exercise)
