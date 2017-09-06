# Lab 4

Here we will learn substantially more sophisticated types of queries, dig deeper into SQL, Develop high-level techniques for describing a database schema.

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
- [Working with indexes](#working-with-indexes)
  - [Some concrete examples](#some-concrete-examples)
  - [Some practice with unions](#some-practice-with-union)
- [Some new developments](#some-new-developments)
- [To Do](#to-do)

## Back to the tables from Lab 3

We return to modifying the tables we made in lab 3.  Keeping in mind what you now know about data normalization and anomalies, redesign the `poorDesign` table by splitting it into multiple tables without redundancy.  Preface all your table names with `PD_` so know which ones to look at.

## Level Two and beyond (`ORDER BY`)

The next level of understanding (this is  the first one you learn *after* Magic Missile) comes from the `ORDER BY` 
clause.  As you might expect, this controls the order of the entries in the table.  Try these:

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
There are several aggregate functions, but the mariaDB manuals certainly doesn't make it easy to figure out exactly what they are.  Here is the list from the mySQL documentation:

<http://dev.mysql.com/doc/refman/5.7/en/group-by-functions.html>

I checked them and they all work.  You should try a few.  Pay attention to the difference between 

`count(*)`, `count(<col name>)` and `count(DISTINCT <col name>)`

They are subtle, but useful.

## The `HAVING` clause

Here's something a bit more interesting... try these:

```
SELECT comboCode, count(*) FROM poorDesign WHERE item >=3 GROUP BY comboCode;
SELECT comboCode, count(*) FROM poorDesign GROUP BY comboCode;
```

(you might need to adjust the `WHERE` clause to fit the values in your data... the point is to ensure that SOMETHING gets filtered.)

And of course, by saying that, I've highlighted what I wanted to point out... namely that the `WHERE` clause filters rows **before** grouping.  This means it affects just exactly WHAT gets grouped and can affect the output of the aggregating functions.  

Now what happens if we want to restrict the `GROUPS` that we see?  That's where the `HAVING` clause comes into play.  It controls what GROUPS are displayed.... and even cooler.. it can use aggregate functions of its own... here's a perfectly valid command (which may, or may not, produce anything interesting-- depending upon the contents of your tables):

```
SELECT comboCode, count(*) FROM poorDesign GROUP BY comboCode HAVING sum(comboCode)>3;
```

You are now ready to read this (although there are still a few missing pieces):

<https://mariadb.com/kb/en/select/>

And with those pieces in place, go through and do these tutorials:

<http://www.sqlcourse2.com/index.html>

## SubQueries

So... next level are subqueries (we're looking at Wizard's Eye here).  Here's an example:

```
SELECT * FROM inventory WHERE id IN (SELECT item FROM poorDesign);
```
Notice the parenthesis and the `IN` keyword (Note:  `IN` is *not* required to make the subquery work).  Notice that the subquery is in parenthesis.

Read this:  <http://beginner-sql-tutorial.com/sql-subquery.htm>
And also this: <http://en.wikipedia.org/wiki/Correlated_subquery>

Now try this, rather stupid, example of a **correlated subquery**:

```
SELECT (SELECT unit FROM inventory WHERE inventory.id=bob.id) FROM inventory AS bob;
````

The key to understanding what is happening is to realize that for EVERY row in the outer `SELECT` statement the INNER query is run.  In order for this sort of nested command to work correctly the inner `SELECT` needs to return **one value**.  This might make things a bit clearer:

```
SELECT item,unit,amount,(SELECT unit FROM inventory WHERE inventory.id=bob.id) AS sub FROM inventory AS bob;
```

Pay special attention to giving aliases to the table names.  The outer query and inner query need some sort of common reference to be correlated… of course they don't HAVE to be correlated:

```
SELECT item,unit,amount,(SELECT 'bob' ) AS sub FROM inventory AS bob;
```

This sort of thing is useful for including aggregate information:

```
SELECT item,unit,amount,(SELECT avg(amount) FROM inventory WHERE unit=bob.unit GROUP by unit) AS sub FROM inventory AS bob;
```

So far we have been looking at various ways to combine cols of tables-- perhaps we used a function to transform the contents of a column, or perhaps we used a function to calculate a value derived from the value of several different columns:

```
SELECT concat(item,unit) AS together FROM inventory;
```

### The `UNION` keyword

As an example of merging two columns into one aggregated one.  But what if we want to merge ROWS?
That's where `UNION` comes from:

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

## Data Models

When designing a database you will want to have some systematic way of indicating what information the database must hold and the relationships between components of that information.  Many authors call the structure of the database a **database schema**.

### Entity Relationship Model

A first pass to accomplishing this is to identify **entities** and **relationships**.  Wikipedia has a nice entry on <http://en.wikipedia.org/wiki/Entity%E2%80%93relationship_model>.  This is a level of abstraction at which *attributes*, *keys* and *relations* exist.   Pay close attention to the difference between **entity** and **entity set**.

If I can borrow an analogy from object oriented programming-- entities could be viewed as instances, entity sets as classes (which is a bit of a stretch... but work with me).  Attributes can then be viewed as fields in the class definition.  The idea of *relations* doesn't really fit this analogy without introducing more complications than are useful.  You might also find the [tutorial](http://www.cs.sfu.ca/CourseCentral/354/zaiane/material/notes/Chapter2/node1.html) useful.

<span name="er-exercise"></span>**ER Exercise:**

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

Many database systems (mariaDB and mySQL included) allow the user to introduce **constraints** into the system.  Read up on [mysql's foreign keys](http://dev.mysql.com/doc/refman/5.6/en/create-table-foreign-keys.html), for an example.  These constraints help maintain **referential integrity**:  consistency in table spread across multiple tables.

For example, suppose you had an `inventory` table with a `vendor_id` attribute that was a foreign key in the `vendor` table.  It would probably be a bad idea if you inserted a record into `inventory` that included an invalid `vendor_id`.  

<span name="foreign-key-exercise"></span>**Foreign Key Exercise**:  Each group should create two tables, one named `fk_A`, the other `fk_B`.  
* The first table should have a column named `idA` and `foreignID`.  The second table should have a column named `idB` and `text`.
* The columns `idA` and `idB` should be primary keys, and `foreignID` should be a foreign key that refers to `idB` in `fk_B`.  
* Experiment with creating the tables, inserting records, deleting records, and modifying values.

Read this [introductory tutorial](http://www.databasejournal.com/sqletc/article.php/1469521/Introduction-to-Relational-Databases.htm).  Much of it will be review, but that's okay.

## Introduction to indexes

An **index** is a data structure associated to a table that makes look-ups quicker.  Faster look-ups allow for faster queries, faster updates, and faster deletes.  They do come with a price though-- that extra structure has to be maintained.  You pay the cost with increased storage requirements and (occasionally) slower operations;  Operations like inserting, deleting, and updating *may* take more time because of that overhead, but they *may* also take less time if `WHERE` clauses are involved and the indexes speed up the searching.  **Note:**  In Database Land, the plural of index is **indexes**, not indices.  

[This is a good overview](https://mariadb.com/kb/en/mariadb/what-is-an-index/), but be sure to read [this MariaDB page too](https://mariadb.com/kb/en/mariadb/getting-started-with-indexes/).

In MariaDB, indexes come in three primary forms:

1. **B-tree**
1. **Hash**
1. **R-tree**

**B-trees** are a data-structure (perhaps more accurately a family of data-structures) that were designed to optimize typical data-base style manipulations-- lookup data is stored in a way that tries efficiently use relatively slow physical disk access and minimize the time spent necessary to keep the tree's order in place.  Lookups are `O(lg n)`:  For more information check out the [wikipedia page on B-trees](https://en.wikipedia.org/wiki/B-tree).  I am **not** expecting you to know all the ins-and-outs, but it is worth having an idea of what's going on.  Your text from Algorithms (CLSR) has a good writeup on B-Trees.  B-tree indexes are used for column comparisons using the >, >=, =, >=, < or BETWEEN operators, as well as for LIKE comparisons that begin with a constant.
For example, the query 

```
SELECT * FROM Employees WHERE First_Name LIKE 'Maria%'
```

can make use of a B-tree index, while 

```SELECT * FROM Employees WHERE First_Name LIKE '%aria'```
cannot.

B-tree indexes also permit leftmost prefixing for searching of rows.

An index that is a B-tree index can **dramatically** improve query times-- they can also greatly improve performance on **ordering** indexed data.

In contrast **Hash** indexes use a hash data structure.  They are **very** fast.  Hash indexes can only be used for equality comparisons, so those using the = or <=> operators. They cannot be used for ordering, and provide no information to the optimizer on how many rows exist between two values.

Hash indexes do not permit leftmost prefixing - only the whole index can be used.

The structure of your data should help you determine which one is best to use.

Professor Michalis Vazigiannis has a [nice series of powerpoint slides](http://www.enseignement.polytechnique.fr/informatique/profs/Michalis.Vazirgiannis/course_slides/4_indexing_hashing.pdf) on the topic.  Don't worry too much about implementation details, but you should at least skim through all the slides.

**R-tree indexes(https://mariadb.com/kb/en/mariadb/spatial-index/)** may be created when an a `CREATE INDEX` (or quivalent) command includes the `SPATIAL` keyword.  The exact structure used by MariaDB can vary (Both B-trees and R-trees are possible).  These indices are used for data like geographic information.

## Working with indexes

Indexes can be added or removed after a table is created using `ALTER TABLE`.  They can also be incorporated into a `CREATE TABLE` command.  The type of index is controlled via the `USING` keyword.  There are also "shortcut" commands in SQL that simplify things:  `CREATE INDEX` and `DROP INDEX` being two of them.

### Some concrete examples

So let's see how much of a difference this makes. Go back to the command line, and we will create two
text files called `rndA` and `rndB`. Both will contain two columns of random integers between 1 and 500
and each will be 1000 lines long. Since I'm not 100% certain what we have on-hand we'll just use stock
commands:
```{bash}
for i in `seq 1 1000`;
do
 echo "scale=0; $RANDOM/32.767"|bc
done > col1A
```
(Don't be surprised if the prompt changes after the first line). 

Now repeat for `col2A`, `col1B`, and `col2B`.

At this point you should have 4 files: col1A, col1B, col2A, col2B.  Now we are going to use them to make the files  `randA` and `randB`:

```
paste col1A col2A > randA
paste col1B col2B > randB
```

Now each group should make two **TABLES** `randA` and `randB` with columns A and B both of which contain integers and no primary key. Then use

```{sql}
LOAD DATA LOCAL INFILE 'randA' INTO TABLE randA;
```

(make sure to write down the time somewhere). Now fill the `randB` table with the corresponding values
from `randB`.

Let's start simply: Find all the rows in `randA` where A is between 1 and 10. (that takes about 0.09
seconds for me)

Now preface that with EXPLAIN:

```{sql}
EXPLAIN SELECT * FROM randA WHERE A BETWEEN 1 AND 10;
```

Pay attention to the rows value.

Now try this one (it took me FAR too long to come up with a query that confuses the built-in optimizer
that MariaDB uses):

```{sql}
SELECT count(*) FROM randA, randB, randA AS C WHERE (randA.A<randB.A OR randA.A > randB.b) AND
C.B=randB.b;
```

It took me about a minute for it to run. Now add indices to the relevant columns (well... all of them)

```{sql}
ALTER TABLE randA ADD INDEX A (A);
ALTER TABLE randB ADD INDEX A (A);
ALTER TABLE randA ADD INDEX B (B);
ALTER TABLE randB ADD INDEX B (B);
```

Now run the same query again (mine was MUCH faster the second time)  
After you've run it a second time, preface it with `EXPLAIN` and examine the rows. Now go back, remove
the keys and run it again (just to make sure it's not a side effect of caching):

```{sql}
ALTER TABLE randA DROP INDEX A;
ALTER TABLE randB DROP INDEX A;
ALTER TABLE randA DROP INDEX B;
ALTER TABLE randB DROP INDEX B;
SELECT count(*) FROM randA, randB, randA AS C WHERE (randA.A<randB.A OR randA.A > randB.b) AND
C.B=randB.b;
EXPLAIN
SELECT count(*) FROM randA, randB, randA AS C WHERE (randA.A<randB.A OR randA.A > randB.b) AND
C.B=randB.b;
```

Now, let's drop the tables and recreate them (this time with the keys). Then we'll fill the data (our data
set is so small this won't make much of a difference-- but it's good for you to know how to do this):
```{sql}
DROP TABLE randA; DROP TABLE randB;
```
Now recreate them, but this time with indexes from the beginning (hint:  that's something you'll need to do yourself).

Now fill the first one without disabling the indexes:
```{sql}
LOAD DATA LOCAL INFILE 'randA' INTO TABLE randA;
```

Before filling the second one, disable the keys:
```{sql}
ALTER TABLE randB DISABLE KEYS;
LOAD DATA LOCAL INFILE 'randB' INTO TABLE randB;
ALTER TABLE randB ENABLE KEYS;
```
The last line enabled the keys.  It is more efficient to parse the entire file at once, instead of line-by-line.
A piece of software that I wrote back in 2008 could read in millions of lines, and disabling the keys
before importing saved over half an hour of time.

### Some practice with `UNION`

We have already seen how to use `UNION` to combine the results of two queries with similar structures.  Let's practice.  

First make sure these toy queries don't return anything too long:

```{sql}
SELECT * FROM randA where A BETWEEN 2 AND 9;
SELECT * FROM randB where B BETWEEN 2 AND 9;
```

Now join them into one:

```{sql}
SELECT * FROM randA where A BETWEEN 2 AND 9 UNION SELECT * FROM randB where B BETWEEN 2 AND 9;
```

Now try this:

```{sql}
SELECT * FROM randA where A BETWEEN 2 AND 9 UNION SELECT * FROM randA where A BETWEEN 2 AND 9;
```
The key to understanding that output is to realize that `UNION` truly does mean `UNION` (no repeats
allowed-- in other words the output is, by default, at least in Type 1 Normal Form). 

You can force it to include everything like this:
```{sql}
SELECT * FROM randA where A BETWEEN 2 AND 9 UNION ALL SELECT * FROM randA where A BETWEEN 2 AND 9;
```
If using union, then you are allowed one ORDER BY clause and it must come AFTER the last select (no
ordering the pieces then joining them together). This will work:
```{sql}
SELECT * FROM randA where A BETWEEN 2 AND 9 UNION ALL SELECT * FROM randA where A BETWEEN 2 AND 9 ORDER BY B;
```
And it won't even complain about ambiguity because the ORDER BY is acting on the merged records.

## Some new developments

Each database in MariaDB is stored using a **database engine**.  The defaults are a bit different between MariaDB and mySQL, but we aren't going to worry too much about those right now.

Different database engines allow for different capabilities.  The `innoDB` engine (and descendents), for example, allow for **transactions** (we are going to talk about this in more detail in a later lab).  Transactions allow you to attempt to perform a series of actions.  At the end of the sequence the transaction is either **committed** if it succeeds or **rolled-back** if it fails.  This is an extension of the core idea of **atomicity** and forms one of the pillars of the relational database **ACID** concept (**A**tomicity, **C**onsistency, **I**solation, **D**urability).  

Different database engines also can provide different types of indexes.  The `TokuDB` engine makes use of [fractal tree indexes](https://en.wikipedia.org/wiki/Fractal_tree_index) in a way that makes it particularly well suited to dealing with large data (Peter has used them in his research).



## To Do

Make sure and complete the following:
* Submit the assignment in canvas and include the names of people in your group as well as the URL pointing to your group's github repository
* Create the `PD_` tables during your `poorDesign` rewrite.
* Each student should type up all the samples (NOTE:  copy-and-paste is bad-- just watching is bad:  type them up and talk to your group-mate(s) about what it means).  Nothing to turn in for this-- but a few tables should be generated (see below)
* Read the contents of these links (see the lab for details of what to skim):
   * [Aggregation functions](http://dev.mysql.com/doc/refman/5.7/en/group-by-functions.html)
   * [The SELECT statement](https://mariadb.com/kb/en/mariadb/select/)
   * [SubQueries](http://beginner-sql-tutorial.com/sql-subquery.htm)
   * [Wikipedia page on Correlated subqueries](https://en.wikipedia.org/wiki/Correlated_subquery)
   * [Wikipedia page on ER models](https://en.wikipedia.org/wiki/Entity%E2%80%93relationship_model)
   * [ER tutorial](http://www.cs.sfu.ca/CourseCentral/354/zaiane/material/notes/Chapter2/node1.html)
   * [Relationships in relational databases](http://www.techrepublic.com/article/relational-databases-defining-relationships-between-database-tables/)
   * [Creating tables with a foreign key](http://dev.mysql.com/doc/refman/5.6/en/create-table-foreign-keys.html)
   * [Good overview and recap](http://www.databasejournal.com/sqletc/article.php/1469521/Introduction-to-Relational-Databases.htm)
   * [Overview of indexes](https://mariadb.com/kb/en/mariadb/what-is-an-index/) **Note:** be sure to read all the pages-- this one has arrows to lead to the next page!
   * [Wikipedia entry on B Trees](https://en.wikipedia.org/wiki/B-tree)
   * [Amazing set of slides on indexing and hashing](http://www.enseignement.polytechnique.fr/informatique/profs/Michalis.Vazirgiannis/course_slides/4_indexing_hashing.pdf)
   * [Spatial indexes in MariaDB](https://mariadb.com/kb/en/mariadb/spatial-index/)
   * [Fractal Tree indexes](https://en.wikipedia.org/wiki/Fractal_tree_index)
* Do these tutorials (nothing to turn in-- but do the problems)
   * [A second SELECT tutorial](http://www.sqlcourse2.com/)
* Do these exercises:
   * [Entity-Relation Exercise](#er-exercise)
   * [Foreign Key Exercise](#foreign-key-exercise)
   * Make sure your tables exist for the shell/mysql index and union examples.
