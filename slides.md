# Migrating to a Temporal Schema

Paul A. Jungwirth

PG-Data 2026

June TODO

Notes:

- Hi, thanks for coming! I'm Paul Jungwirth, a freelance developer in Portland, OR.
- I do a lot of Postgres consulting and a lot of webapp development,
  and I really want to start offering temporal data features to my customers.
- I added temporal primary and foreign keys in v18, and in v19 it looks like we'll get temporal update and delete.
- So since the features are starting to land, what would it take to adopt them?
- How can you migrate from a regular schema to a temporal schema?



# Outline

- Temporal Tables
- Queries
- DML
- DDL

Notes:

- So I want to briefly cover what temporal tables look like,
- Then how you'd query them,
- then how you'd modify them,
- and finally most of the talk will be about the data migration.


# Application Time

TODO

Notes:

- If we're recording the history of a thing out in the world, we call that "application time", or an older term is "valid time".
- If each row in a table is an Aristotelian truth-proposition, application time lets us say when it was true.
- We give each row two timestamps: a start time or an end time.
  - In Postgres we can do this with a single rangetype column.
- An entity can have multiple rows, each one a different version.
- As long as their valid-times don't overlap, there is no contradiction.
- When an entity changes, we set its end time and we make a new row for the current version, with valid-time starting now.
- So there is always one row whose end-time is unbounded.
- Unless we delete the entity, and then we really just set the end-time to now.


# System Time

Notes:

- Briefly, there is another dimension you could track, called System Time or Transaction Time.
- This is the history of the database itself: who changed what when.
- Often you need this for auditing or compliance.
- We don't have this yet in core Postgres, but there are several extensions that can do it for you.
- Whereas Application Time is your responsibility, System Time is maintained automatically by the database.
- I'm not going to go into this, because it's usually transparent.
- You can add system time without changing anything in your application.


# Primary Keys

# Foreign Keys

# Queries

TODO: snapshot query

Notes:

- If you consider a temporal table at a certain point in time,
  it becomes just a regular table.
- One of the main researchers of temporal data, Richard Snodgrass, calls this is snapshot query:
  - a temporal table goes in, a regular table comes out.


# Queries

```sql
SELECT  *
FROM    t
WHERE   valid_at @> current_time;
```

Notes:

- Here is the rangetype "contains" operator.
- This gives us everything that is true right now.
- If you add this to all your queries, you'll get the same shape you had before.
  - Technically you're also selecting the new `valid_at` column,
    but you could avoid that by replacing the star with an explicit column list.



# Queries

TODO: quote from the paper

Notes:

- Snodgrass has a paper from 1993 with a lot of helpful algebraic results about temporal relational operators.
- He developed a query language called TQuel, an extension to Quel, the query language of this old database called Ingress.
- This result is very important for us.
  - It says that if you snapshot the result of some temporal operators,
    it's the same as doing non-temporal operators on snapshots of the base relations.
  - In other words we can push down the filtering, and do the same query we were doing before.
    - We don't really need all these temporal operators, if in the end we only care about a snapshot result.
- So this means you don't need to change any queries you have today,
  except to filter the base rels as I showed a moment ago.



# DML

```sql
UPDATE products
  FOR PORTION OF valid_at
  FROM current_time TO NULL
  SET price = 1.10 * price;
```

Notes:

- Here is how we update the database.
- For example we can raise all the prices by 10%.



# DML

TODO: picture of temporal update

Notes:

- Graphing this as a timeline,
  we still have the old record at $10, now bounded,
  and the new record valid from today.
- For your existing DML, you can just use `FOR PORTION OF` from now 'til infinity,
  and everything acts like before.
- You're capturing history, but you're not revising it.
- But if you wanted to revise it, you can add new features to do that.
- Say you gave someone a promotion, but HR didn't update their employment record right away.
- They can use a `FOR PORTION OF` to make the new position start on the right day.



# DML

```sql
DELETE FROM products
  FOR PORTION OF valid_at
  FROM current_time TO NULL
  WHERE color = 'yellow';
```

Notes:

- Here's a delete.
- We immediately discontinue everything yellow.
- It's the same idea.



# DML

TODO: picture of temporal delete before and after

Notes:

- So before our record was valid indefinitely,
- but now it's validity ends today.
- This is like a super-powered soft delete, btw.
- We kept the old record.
- One difference is we can still validate foreign keys.
- With regular soft-delete, you could have a non-deleted record pointing to a soft-deleted one, and your foreign keys won't catch it.
- With a temporal foreign key, the referencing row can't outlive the referenced row.



# 
