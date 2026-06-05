# Migrating to a Temporal Schema

Paul A. Jungwirth

PG DATA 2026

5 June 2026

Notes:

- Hi, thanks for coming! I'm Paul Jungwirth, a freelance developer in Portland, OR.
- I do a lot of Postgres consulting and a lot of webapp development,
  and I really want to start offering temporal data features to my customers.
- I added temporal primary and foreign keys in v18, and in v19 it looks like we'll get temporal update and delete.
- So since the features are starting to land, what would it take to adopt them?
- One thing you need, of course, is a temporal database schema.
- How do you adopt that in your projects?
- If you have ordinary non-temporal tables right now, it might be pretty easy.
- If you have already built non-standard ways to capture history, you might have more work.
- I have a project like that, built before I knew anything about temporal databases.
- It's a time-tracking and invoicing application I've used in my business since 2013.
  - That's the same year I read Snodgrass's book on temporal databases, in fact.
- Of course there are some embarrassing decisions, and I won't hide them from you.
- I've been working to convert this, so I'll share some of the more interesting challenges.



# Outline

- Temporal Tables
- Queries
- DML
- DDL

Notes:

- First I want to briefly cover what temporal tables look like,
- Then how you'd query them,
- then how you'd modify them,
- and finally most of the talk will be about the data migration.



# Goals

|     |                        |
| --- | ---------------------- |
|  ☒  | Data Preserving        |
|  ☒  | Backwards Compatible   |
|  ☒  | Incremental Progress   |
|  ☐  | Online Migration       |

Notes:

- Here are some major goals of the project:
- It needs to convert the old data to the new temporal structure.
  - Obviously it needs to not lose data.
  - In my case I was already tracking changes, just in an ad hoc, ugly way.
    - For example
      - sometimes I would raise my rate,
      - or the client's PO number on the invoice changed,
      - or I finished a big project and switched to a monthly support retainer.
    - I want to keep the history so that I can run reports against previous years.
    - The report for 2022 should give the same results as before.
- Backwards compatible:
  - I want minimal application changes.
    - I don't want to re-write SQL all over the place.
    - My app is comparatively tiny. Most projects care even more about this.
    - This has been a goal of temporal database research for decades.
  - If you are just converting non-temporal tables to temporal, I think you can get 100% there.
  - If you are *also* converting some janky way of tracking history, like me, your app probably needs to adapt too.
    - I still got really close, so I was quite happy with the result.
- Incremental progress:
  - Ideally you can convert small bits of your schema at a time.
  - Big bang projects frequently fail.
    - We want a way to move forward, get it in production, and then do it again and again.
  - What makes this hard is foreign keys.
    - I've got a few approaches I'll show.
- One thing that's *not* a goal is doing an online migration:
  - I don't mean you have to shut down your database.
  - What I mean is where you deploy app changes ahead of time, so that it works on both old and new schemas,
    then you change the schema, then you remove the obsolete app code.
  - That is really an orthogonal technique, and I'd rather focus on the temporal-specific bits.
  - If you've done this before, you probably know what to do.



# Application Time

![clients table](/img/clients_rows.png)

Notes:

- So what do temporal tables look like anyway?
- If we're recording the history of a thing out in the world, we call that "application time", or an older term is "valid time".
- If each row in a table is an Aristotelian truth-proposition, application time lets us say when it was true.
- We give each row two timestamps: a start time or an end time.



# Application Time

![clients table, with rangetypes](/img/clients_rows_ranges.png)

Notes:

- [SHOW] Actually in Postgres we can do this with a single rangetype column.
- Notice an entity can have multiple rows with the same primary key!
- As long as their valid-times don't overlap, there is no contradiction.



# Application Time

![clients table, invalid](/img/clients_rows_ranges_invalid.png)

Notes:

- Here is the same thing but this last row is no good.
- Its first row covers all time since 2016,
  and the second row says something else for 2017.
- That is a contradiction!



# Application Time

![clients timeline](/img/clients_timeline.png)

Notes:

- It helps to show these records on a timeline.

- When an entity changes, we set its end time and we make a new row for the current version, with valid-time starting now.
- So there is always one row whose end-time is unbounded.
- Unless we delete the entity, and then we really just set the end-time to now.



# System Time

![bitemporal timeline](/img/bitemporal_timeline.png)

Notes:

- Briefly, there is another dimension you could track, called System Time or Transaction Time.
- This is the history of the database itself: who changed what when.
- Often you need this for auditing or compliance.
- We don't have this yet in core Postgres, but there are several extensions that can do it for you.
- Whereas Application Time is your responsibility, System Time is maintained automatically by the database.
- I'm not going to go into this, because it's usually transparent.
- You can start capturing system time without changing anything in your application.



# Primary Keys

```sql
-- old:
CONSTRAINT legacy_pk EXCLUDE
  (id WITH =, valid_at WITH &&)

-- new:
CONSTRAINT shiny_pk PRIMARY KEY
  (id, valid_at WITHOUT OVERLAPS)
```

Notes:

- A temporal primary key follows the semantics we've covered:
  - You can have duplicates, but they can't overlap.
- Previously you could achieve this with an exclusion constraint.
- Now there is special syntax you can use.
- You can do this for unique constraints too, btw.
- The advantage is that Postgres understands this is a primary key,
  so it unlocks more behavior: some optimizations, logical replication,
  best of all: [SHOW]



# Foreign Keys

```sql
CONSTRAINT fk_work_categories_on_client_id
  FOREIGN KEY (client_id, PEROID valid_at)
  REFERENCES clients (id, PERIOD valid_at)
```

Notes:
- Foreign keys!
- You know a foreign key has to point to a primary key or unique constraint.
- Same with a temporal foreign key: it has to reference a temporal primary key or unique constraint.
- The reference is valid as long as the referenced row exists.
- In my app, a work category is a child of clients.
- I use this to distinguish between development, managing, etc.
- It can't exist before the client does or after.



# Foreign Keys

![referencing more than one row](/img/clients_fk_work_categories_timeline.png)

Notes:

- I want to point out a subtlety here: a foreign key can depend on more than one row.
- Maybe a client changes but its work categories don't. That's okay!
- Here there are two rows in the `clients` table, but just one row in `work_categories`.
- The work category outlives either one, taken separately.
- When Postgres checks a temporal foreign key, it aggregates all the referenced rows' application times, and compares against that.



# Foreign Keys

| referencing | referenced | supported? |
| ----------- | ---------- | ---------- |
|   normal    |   normal   |     ☒      |
|  temporal   |  temporal  |     ☒      |
|  temporal   |   normal   |     ☒      |
|   normal    |  temporal  |     ☐      |

Notes:

- So in a schema migration, you are going to want to change your primary keys and foreign keys to temporal versions.
- The problem is that you're pulling at a thread, and if you keep pulling the whole sweater unravels.
- Can you migrate just a small part of your application at a time?
- Well, a temporal table can point to a regular table, no problem.
  - Just make a regular foreign key column and use it.
- How do you make a non-temporal table point at a temporal table?
  - It has to reference a unique constraint, but you can't do that.
  - Stay tuned!



# Queries

![snapshot query](/img/snapshot_query.png)

Notes:

- So let's talk about querying your tables.
- If you consider a temporal table at a certain point in time,
  it becomes just a regular table.
- One of the main researchers of temporal data, Richard Snodgrass, calls this is snapshot query, also a time-slice query.
  - a temporal table goes in, a regular table comes out.
- This is usually what you want: what is true right now?
- For a legacy app, that's what *all* your queries are asking.



# Queries

```sql
SELECT  * EXCEPT valid_at
FROM    t
WHERE   valid_at @> current_timestamp;
```

Notes:

- So here is SQL to ask that question.
  - We're using the rangetype "contains" operator.
  - This gives us everything that is true right now.
- If you add this to all your queries, you'll get the same shape you had from non-temporal queries.
  - Technically you're also selecting the new `valid_at` column,
    but you could avoid that by replacing the star with an explicit column list.
  - The `EXCEPT` I'm using here is not real syntax, sadly.



# Queries

σ(R ⨯̂ S) = σ(R) × σ(S)

![snapshot reducilibity](/img/snapshot_reducibility.png)

Notes:

- Fortunately, snapshotting is isomorphic over temporal operators.
- This is from a Snodgrass paper from 1993.
- What I mean is you can take the snapshot before you do stuff like joins, or after.
- So even big complicated queries don't need any more changes,
  except to filter the base rels as I showed a moment ago.



# VIEWs
<!-- .slide: style="font-size:85%; text-align:left" -->

```sql[|15|16|1-14|8|3-7|9-13|16] CREATE VIEW clients AS
SELECT  id, name, slug,
        ..., position,
        -- created_at: The very first version's lower(valid_at):
        -- range_agg does the right thing for NULL endpoints, unlike MIN.
        (SELECT lower(range_agg(valid_at))
          FROM  temporal.clients c2
          WHERE c2.id = clients.id) AS created_at,
        lower(valid_at) AS updated_at,
        -- deleted_at: The very last version's upper(valid_at), which is usually NULL:
        -- range_agg does the right thing for NULL endpoints, unlike MAX.
        (SELECT upper(range_agg(valid_at))
          FROM  temporal.clients c2
          WHERE c2.id = clients.id) AS deleted_at,
        archived
FROM    temporal.clients
WHERE   valid_at @> current_timestamp at time zone 'UTC'
```

Notes:

- This sounds like a job for views!
- If you put your new temporal tables in a different schema, say "temporal", [SHOW]
  then you can replace your old tables with views that filter on the current time. [SHOW]
  - From the app's perspective, nothing has changed.
- Here I'm *omitting* the new `valid_at` column: the column list is exactly the same as before. [SHOW]
- Also I'm synthesizing some Ruby on Rails columns from the valid time.
  - `updated_at` is just the beginning of the current version.[SHOW]
  - `created_at` is the beginning of the earliest version.[SHOW]
  - `deleted_at` is the end of the latest version. Usually this is just null.[SHOW]
  - Going back to this filter: [SHOW]
    - Don't judge me for using timestamp *without* time zone!
      - This is what Rails understands, and anyway in 2013 I didn't know any better.
      - In a Rails app, this is always in UTC.
- So with this view in place, your old queries Just Work^TM.



# UPDATE

```sql
UPDATE work_categories
  FOR PORTION OF valid_at
  FROM '2019-01-01' TO '2020-01-01'
  SET rate = 1.10 * rate;
  WHERE client_id = 5;
```

Notes:

- Okay on to updating!
  - We call that foreshadowing. . . .
- Here is how you update a temporal table.
- For example we can raise our rate by 10%.
- We're targeting just 2019.
  - Normally we'd probably target everything from `current_timestamp` and beyond.



# UPDATE

![temporal update: before](/img/temporal_update_before.png)

Notes:

- Here is a timeline of the record before our update.



# UPDATE

![temporal update: goal](/img/temporal_update_goal.png)

Notes:

- Here is our goal: just 2019 has been updated.
  - This requires three rows to model.



# UPDATE

![temporal update: after](/img/temporal_update_after.png)

Notes:

- The update command first does this:
  - It sets the columns we want, and it also shrinks the valid-time to only what we targeted.



# UPDATE

![temporal update: leftovers](/img/temporal_update_leftovers.png)

Notes:

- Then Postgres inserts leftovers to preserve the untouched history.
- So you can change any part of history . . .
- but if you're migrating an existing app, your existing code is always changing things "from now 'til futher notice."
- You could do this with an `INSTEAD OF` trigger on your non-temporal view.
- Then you don't have to rewrite any update code.



# UPDATE

```sql[|5-8|5-8,13]
CREATE FUNCTION update_work_categories_view() RETURNS trigger
LANGUAGE plpgsql AS
$$
BEGIN
  UPDATE temporal.work_categories
    FOR PORTION OF valid_at
      FROM current_timestamp AT TIME ZONE 'UTC'
      TO NULL
    SET client_id = NEW.client_id,
        name      = NEW.name,
        rate      = NEW.rate
    WHERE id = NEW.id;
  RETURN NEW; -- Enables RETURNING
END;
$$;
```

Notes:

- Here is a function for that.
  It sets all the columns,
  but adds a `FOR PORTION OF` clause targeting the future.[SHOW]
- If you say `RETURN NEW`, then your outermost update can use a RETURNING clause.[SHOW]
- This is especially useful for INSTEAD OF INSERT, so that you can get the id generated by the base table.
  - That part is like magic; I honestly didn't expect it to work.



# UPDATE

```sql
CREATE TRIGGER trg_update_work_categories
  INSTEAD OF UPDATE ON work_categories
  FOR EACH ROW
  EXECUTE FUNCTION update_work_categories_view();
```

Notes:

- Here we install it as an `INSTEAD OF` trigger.



# DELETE

```sql
DELETE FROM clients
  FOR PORTION OF valid_at
    FROM current_timestamp AT TIME ZONE 'UTC'
    TO NULL
  WHERE name = 'Nadir';
```

Notes:

- Here's a delete.
- This client was too annoying, so we fired them.
- It all works the same as update.
- This is like a super-powered soft delete, btw.
  - We kept the old record.
  - Unlike regular soft-deletes, we can still validate our foreign keys.
  - How many times have you seen a soft-deleted record still referenced by not-deleted records?
    That shouldn't happen, right?
  - A temporal foreign key will see the problem and prevent the change.
  - With a temporal foreign key, the referencing row can't outlive the referenced row.
  - Again, we could use an `INSTEAD OF` trigger so we don't have to change any code.



# INSERT

```sql[|10|11-12]
CREATE FUNCTION insert_work_categories_view()
RETURNS trigger
LANGUAGE plpgsql
AS
$$
BEGIN
  INSERT INTO temporal.work_categories
    (client_id, name, rate, valid_at) VALUES
    (NEW.client_id,  NEW.name, NEW.rate,
    tsrange(current_timestamp AT TIME ZONE 'UTC', NULL))
    RETURNING id INTO NEW.id;
  RETURN NEW;
END;
$$;
```

Notes:

- There is no new syntax for a temporal insert.
  - Just insert it with the application-time you want.
  - Here is a trigger function to translate inserts against the view into inserts against the new temporal table. It just adds a valid-time starting today. [SHOW]
- But there is a subtlety here. [SHOW]
- The id of the underlying temporal table comes from a sequence,
  and if we just return `NEW`, then the outer insert won't see it.
- So we have to attach it to the outer `NEW`.



# Legacy Tables

![client tables](/img/legacy_clients.png)

Notes:

- If you were already doing something weird to capture history, you might have more work to do.
- In my case I had a `clients` table and a `client_snapshots` table, with nearly all the same columns.
- Then I had lots of annoying application code to do the right thing.
- Whenever I made a new client snapshot, I also updated the clients table.
- So the current version was always in *both* the `client_snapshots` table and the `clients` table.
- Also I had this `replaced_by_id` column to put the snapshots in order.
- One thing that helps: since this is a Rails app, I had a `created_at` column.

- For clients I also had a `deleted_at` column for soft-deletes
  *and* an `archived` flag.
  - Having both seems to be an oversight.
  - Either one hides the client from the main data-entry widgets,
    but neither hides all their past data.
  - But distinguishing between past clients and current clients is a good use of application time.
  - Hopefully a temporal schema lets me drops *both* of these columns.

- Then I had "work categories", which is like a broad classification of the work I'm doing: Development, Consulting, Design, etc.
  - In theory a different work category could have a different rate.
  - Work categories were children of their client snapshot, so they were effectively versioned too.
    - So each time I change a client, I make new work category records.
    - We don't have to do this anymore!
  - As I mentioned before, if a work category doesn't change, it can reference one client version for some of its history, and another client version for the rest.

- So the goal is to get rid of the `client_snapshots` table and give these other two an application-time column.
  - This is more ambitious than transparently putting a history table behind a view.
  - Ultimately I wanted to lose the snapshots table altogether.
  - But I still didn't want to make the app fully temporal-aware,
    so I will use the view + history table here too, just without snapshots.
  - l figured this would let remove a lot of annoying code from the app.
  - I was able to get through all that before this talk,
    but even better I broke it down into steps.
  - I didn't completely drop the `client_snapshots` table right away; at first I made it into another view.



# Legacy Tables

![legacy acts](/img/legacy_acts.png)

Notes:

- Of course there is more to this schema.
- For instance here is an acts table.
- These other two are the same as before.
- The acts table has what I did: it's the time-tracking table.
- This is just an example of something that references `client_snapshots` and `work_categories`.
- There are also invoices and email templates.
  - Yes every application eventually sends mail!



# Migration Plan

- Clean up bad data.
  - Add more constraints.
- Create `temporal` schema with tables.
- Move data over from the old tables.
- Staging:
  - `temporal.clients` with `client_snapshots` view
  - remove `client_snapshots`
  - `temporal.work_categories` with view
  - remove `work_categories` view

Notes:

- So this is our plan.
- First of all, doing this migration will probably uncover a lot of bad data.
  - I had way fewer constraints than I would add now,
    and I guess I never revisited that.
- The best way to do this is to add as many constraints as you can *before* you do the temporal migration.
- And then I was able to do this in steps.
  - Most things have foreign keys to a client snapshot, not the overall client.
    - That's how I know what PO number to use, how many retainer hours to work, etc.
    - So replacing that with a view would be very helpful!
  - Then I'd like to discard the "client snapshot" concept altogether.
    - That of course requires a lot of app changes.
    - Actually it was pretty easy, like changing two or three files.
      - Except nearly every *test* had to be changed too, because their setup was creating `client_snapshots`.
  - And then I also converted `work_categories` to a temporal table.
  - Here again a view was helpful for a while so reporting and invoicing still gave correct answers.



# Incremental Progress

1. Add a dummy `valid_at` to the non-temporal table.
1. Put a traditionally-unique column on the temporal table.
1. Define a "loose" foreign key function.
1. Create separate id tables.

Notes:

- But to break it down into steps, I had to solve foreign keys between legacy and temporal tables.
  - All tables are connected.
  - So you need non-temporal tables to reference temporal ones, and that's the kind of foreign key we don't have.
- I have four solutions for this:
- 1. One solution is to add a `valid_at` column to your non-temporal table, and set it to a short value. You don't have to use it in your primary key, just the foreign key.
  - Just copy a time when you know the reference is valid, maybe just a second from right now.
  - I wish you could use an empty range here, as a way of saying you'll take any time at all.
    - Unfortunately empty ranges don't work that way.
    - The contains operator does, but not the overlaps operator.
  - `NULL` doesn't work either, since if any part of a foreign key is null, it isn't checked at all.
- 2. Add another column that has a regular unique constraint.
  - This is what I did simulating `client_snapshots`.
    - Not really satisfying, but gives you minimal app changes.
    - It lets you escape actual temporal semantics, joining on the key rather than on time, so you could write a join that doesn't make sense.
    - If you aren't careful, the records will be out of sync.



# Loose FK

```sql[|13-22|24-32]
CREATE OR REPLACE FUNCTION add_loose_foreign_key(
  from_table regclass,
  fk_column  text,
  to_table   regclass,
  pk_column  text DEFAULT 'id',
  name       text DEFAULT NULL
) RETURNS void LANGUAGE plpgsql AS $fn$
DECLARE
  v_suffix   text := loose_fk_trigger_suffix(from_table, fk_column, to_table, pk_column, name);
  v_ref_name text := 'loose_fk_ref_' || v_suffix;
  v_inv_name text := 'loose_fk_inv_' || v_suffix;
BEGIN
  EXECUTE format(
    'CREATE CONSTRAINT TRIGGER %1$I
       AFTER INSERT OR UPDATE OF %2$I ON %3$s
       FROM %4$s
       NOT DEFERRABLE INITIALLY IMMEDIATE
       FOR EACH ROW
       EXECUTE PROCEDURE loose_fk_check_ref(%5$L, %6$L, %7$L)',
    v_ref_name, fk_column, from_table, to_table,
    fk_column, to_table::text, pk_column
  );

  EXECUTE format(
    'CREATE CONSTRAINT TRIGGER %1$I
       AFTER UPDATE OF %2$I OR DELETE ON %3$s
       FROM %4$s
       NOT DEFERRABLE INITIALLY IMMEDIATE
       FOR EACH ROW
       EXECUTE PROCEDURE loose_fk_check_inv(%5$L, %6$L, %7$L)',
    v_inv_name, pk_column, to_table, from_table,
    from_table::text, fk_column, pk_column
  );
```

Notes:

- You could define a "loose" foreign key function.
  - Foreign keys are just triggers; you can make your own.
  - You check foreign keys in four places: if the referenced table is updated or deleted, if the referencing table is inserted or updated.

- I've done this by hand before, but here I asked Claude, and it spat out this.
  - This is the function to add a foreign key.
    - It takes the table and column names.
    - Then it puts a trigger on the referencing table,[SHOW]
    - and another trigger on the referenced table.[SHOW]



# Loose FK

```sql[|11-14|16-22]
CREATE OR REPLACE FUNCTION loose_fk_check_ref()
RETURNS trigger LANGUAGE plpgsql AS $fn$
DECLARE
  v_fk_column text := TG_ARGV[0];
  v_to_table  text := TG_ARGV[1];
  v_pk_column text := TG_ARGV[2];
  v_is_null   boolean;
  v_found     integer;
  v_fk_value  text;
BEGIN
  -- MATCH SIMPLE: a NULL fk passes the constraint.
  EXECUTE format('SELECT ($1).%I IS NULL', v_fk_column)
    USING NEW INTO v_is_null;
  IF v_is_null THEN RETURN NULL; END IF;

  -- FOR KEY SHARE matches built-in RI: lock the referenced row so a
  -- concurrent UPDATE-of-key or DELETE blocks until we commit.
  EXECUTE format(
    'SELECT 1 FROM %1$s t
     WHERE t.%2$I = ($1).%3$I FOR KEY SHARE OF t',
    v_to_table, v_pk_column, v_fk_column
  ) USING NEW INTO v_found;

  IF v_found IS NULL THEN
    v_fk_value := to_jsonb(NEW) ->> v_fk_column;
    RAISE foreign_key_violation USING
      MESSAGE = format(
        'insert or update on table "%s" violates loose foreign key constraint',
        TG_TABLE_NAME
      ),
      DETAIL = format(
        'Key (%s)=(%s) is not present in table "%s".',
        v_fk_column, v_fk_value, v_to_table
      );
  END IF;

  RETURN NULL;
END;
$fn$;
```

Notes:

- Here is the trigger function to check the referencing side.
- This basically mirrors Postgres's own foreign key code.
  - If the reference has nulls, we don't bother to check.[SHOW]
  - Otherwise we check the referenced table for a match.[SHOW]
  - We use a `KEY SHARE` lock, just like Postgres.
  - If there's a problem we raise an error.
- Postgres does include some snapshot trickery, so under `REPEATABLE READ` you might notice issues.
  - For `READ COMMITTED` or `SERIALIZABLE` this should be fine.
- This is just for `NO ACTION` foreign keys.
  - For temporal semantics, this means the reference is valid if the temporal object existed at any point in time.
  - For a `CASCADE` foreign key, it might not be such a good idea: what is it supposed to do?



# ID Tables
<!-- .slide: style="font-size:85%" -->

![`client_ids` table](/img/client_ids_table.png)

```sql
CREATE TABLE temporal.single_clients (id integer PRIMARY KEY);

CREATE FUNCTION trg_generate_temporal_client_id()
RETURNS trigger LANGUAGE plpgsql
AS $$
BEGIN
  IF NEW.id IS NULL THEN
    NEW.id := nextval(
      pg_get_serial_sequence('temporal.clients', 'id'));
  END IF;
  INSERT INTO temporal.single_clients (id) VALUES (NEW.id)
    ON CONFLICT DO NOTHING;
  RETURN NEW;
END;
$$;
```

Notes:

- I also tried out making a separate id table.
  - Say you haven't migrated your invoices table yet, but it needs to reference a client.
    - You could put a non-temporal table in the middle, which only has one column.
  - Like foreign keys, effectively you're saying that those other tables can reference a client if it *ever existed*.
  - This idea seems quite desperate, to be honest.
  - I just wanted to prove that it worked.
    - Once all the tests passed, I took it out.

  - It wasn't *that* bad.
    - I used a `BEFORE INSERT` trigger on the real table to insert a row into the ids table.
    - Here is that function.
    - If we don't already have an id, we get one.
    - We use the sequence on the real table, so that it moves.
    - Then we insert it here too.
- Then the new temporal clients table references this,
  and I can point non-temporal tables at it too.



# Clients
<!-- .slide: style="font-size:85%" -->

```sql[|2-4,9|6-7|10|11-14|8,15]
CREATE TABLE temporal.clients (
  id integer NOT NULL GENERATED BY DEFAULT AS IDENTITY
    /* REFERENCES temporal.single_clients (id) */,
  valid_at tsrange NOT NULL,
  ...,
  position integer NOT NULL DEFAULT 0,
  archived boolean NOT NULL DEFAULT false,
  client_snapshot_id SERIAL NOT NULL,
  CONSTRAINT clients_pkey PRIMARY KEY (id, valid_at WITHOUT OVERLAPS),
  CONSTRAINT clients_slug_key UNIQUE (slug, valid_at WITHOUT OVERLAPS),
  -- acts_as_list will trip this if we check
  -- in the middle of a transaction (same as for non-temporal):
  CONSTRAINT clients_position_key UNIQUE (position, valid_at WITHOUT OVERLAPS)
    DEFERRABLE INITIALLY DEFERRED,
  CONSTRAINT clients_client_snapshot_id_key UNIQUE (client_snapshot_id)
);
EOQ
```

Notes:

- Okay I think we are finally ready to make our temporal clients table.
- So we have an id and a `valid_at` column, which are the primary key. [SHOW]
- In the experiment with a separate ids table, the id was both part of the primary key and a foreign key!
  - Just writing my own "loose" foreign key trigger function was easier and a lot cleaner though.
  - I was afraid the weird id tables would become permanent.
- There are a couple default columns.[SHOW]
  - This is a problem for our non-temporal updatable view.
  - Our `INSTEAD OF` trigger explicitly sets each column, so it will set them to `NULL`, not the default.
  - We could detect nulls and fill in the default.
  - But then the user-facing behavior of the view not quite the same as a regular table, where passing an explicit null doesn't give you the default.
  - But it turns out views can have defaults too!
  - You can't put it into the `CREATE VIEW` command, but you can `ALTER VIEW` to add them.
  - That's great because our trigger function can stay nice and simple.
- Here is a temporal unique constraint, on the `slug` column.[SHOW]
  - This is just like a primary key, except it can have nulls.
  - A slug is the name, lowercased, with spaced removed.
  - So it's nice in a URL or a filename, like for invoice PDFs.
- Then we have a position column.[SHOW]
  - This is used to define a sort order.
  - I want that to be unique.
  - It needs to be deferrable, because when you move a client up or down you have to change two rows.
- And we have a client snapshot id column.[SHOW]
  - This is the original id.
  - Keeping it lets us build a view to replace the old table.
  - This is a non-temporal unique column.
  - So that lets us keep the same foreign keys as before.
    - We don't even need our loose foreign keys yet, but I didn't keep this around for long.
  - This is a serial column, so future clients will get a new id.
  - OTOH if we do a temporal update, we *also* need to get a new id.
    - So we add a trigger for that.



# Clients

```sql
CREATE OR REPLACE FUNCTION trg_generate_temporal_client_snapshot_id()
RETURNS trigger
LANGUAGE plpgsql
AS
$$
BEGIN
  SELECT nextval(
           pg_get_serial_sequence('temporal.clients',
                                  'client_snapshot_id'))
    INTO NEW.client_snapshot_id;
  RETURN NEW;
END;
$$;
```

Notes:

- Here is that trigger function.
- This only works if every update has an unbounded end time.
  - But that's fine for now. We're going through our updatable view, and the snapshots table is going away soon.



# Clients
<!-- .slide: style="font-size:85%" -->

```sql[|3|12|4-7|8-11]
INSERT INTO temporal.clients
(id, ..., valid_at, name, position, client_snapshot_id)
SELECT  cs.client_id, cs.name, ...,
        tsrange(
          cs.created_at,
          lead(cs.created_at)
            OVER (PARTITION BY cs.client_id ORDER BY cs.created_at)),
        dense_rank()
          OVER (ORDER BY c.position
                ROWS UNBOUNDED PRECEDING EXCLUDE TIES)
          AS position,
        cs.id
FROM    client_snapshots cs
JOIN    clients c ON c.id = cs.client_id;
```

Notes:

- Okay we've got a table, let's put data into it!
- Here is SQL to fill the client history table.
- Each snapshot becomes a new row.
- I'm preserving the old client ids.[SHOW]
  - This is very helpful throughout the migration.
- I also keep the old snapshot ids. [SHOW]
  - Foreign keys referencing client snapshots can point here instead.
    - Later I was able to replace those with references to `client_id` instead.
  - Just don't forget, after inserting these records, we need to catch up that sequence!
- Then I build the valid-time. [SHOW]
  - Window functions are very helpful here.
    - A row's valid-time is just the creation time of the snapshot until the next snapshot.
- I was worried the position column would be tricky,[SHOW]
  - first because of its temporal unique constraint,
  - and also I was worried about gaps.
  - It practice these weren't an issue,
    but you could use `dense_rank` as here to remove gaps.
  - Unique violations are bad data you'll have to clean up.



# `client_snapshots`
<!-- .element: class="r-fit-text" -->

```sql
SELECT  client_snapshot_id AS id,
        id AS client_id,
        lead(client_snapshot_id)
          OVER (PARTITION BY id ORDER BY id)
          AS replaced_by_id,
        name,
        ...,
        lower(valid_at) AS created_at,
        -- In practice the old data often has created_at <> updated_at,
        -- because we set replaced_by_id,
        -- but the app doesn't care if we lie about it:
        lower(valid_at) AS updated_at,
FROM    temporal.clients
```

Notes:

- Here is the `client_snapshots` view.
- It's really simple: just one-to-one from the new history table, with minor translation.
- Computing the `replaced_by_id` column is the only interesting bit.
  - Again, the lead window function is what we need.
- I don't need this to be an updatable view: the idea is that these don't get updated, right?
- And anyway I've already managed to drop this view entirely now.



# `work_categories`

```sql
SELECT  original_id AS id,
        c.client_snapshot_id,
        wc.name, wc.rate, wc.position,
        -- created_at: The very first version's lower(valid_at):
        -- range_agg does the right thing for NULL endpoints, unlike MIN.
        (SELECT lower(range_agg(valid_at))
          FROM  temporal.clients c2
          WHERE c2.id = clients.id) AS created_at,
        lower(wc.valid_at) AS updated_at,
        wc.retainer_hours
FROM    temporal.work_categories wc
JOIN    temporal.clients c
ON      c.id = wc.client_id AND c.valid_at && wc.valid_at
WHERE   wc.valid_at @> current_timestamp at time zone 'UTC'
```

Notes:

- Then in another step, I moved `work_categories` to a temporal table.
- Here is the view to simulate the old legacy table.
- As I said, in the past I made new work category rows here for each snapshot, even if they didn't change.
  That means that a work category doesn't have a constant id across versions.
  Fortunately the *name* is unique (within a client), so I still have a key.
- My migration plan here has to be right, because work categories have the hourly rate I'm charging.
  - So each act has to join to the *right time*.
  - At first I preserved the old work category ids, just as I preserved the snapshot ids.
    Then I didn't have to update any of those queries right away.
  - Later I dropped the legacy work category ids and used loose foreign keys to the new id that's shared across versions.
  - And then I had to join based on time instead of the legacy id.

- I asked myself if the new `work_categories` table needed to store the old `client_snapshot_id` too?
  - Because often the app joins between work categories and their client snapshot by that id.
  - But I don't actually need it!
  - A work category's valid time is going to be defined by its client snapshot's valid-time,
    so by construction I can join based on that.



# Thanks!

https://github.com/pjungwir/migrating-to-a-temporal-schema

Notes:

- Okay, that's it!
- This migration is still in-progress, but I feel I've got a methodology and a clear path forward.
- Thanks for letting me share this work with you!
- Here is a link to these slides.[SHOW]



# References
<!-- .slide: class="references" -->

- Richard Snodgrass. "An Overview of TQuel." *Temporal Databases: Theory, Design, and Implementation*, 1993.
- Anton Dignös, Michael H. Böhlen, Johann Gamper. "Temporal Alignment." SIGMOD, 2012. https://files.ifi.uzh.ch/boehlen/Papers/modf174-dignoes.pdf
- Paul Jungwirth. Temporal Ops Postgres Extension. https://github.com/pjungwir/temporal_ops
- Boris Novikov. "PostgreSQL Temporal Aggregates: SUM, AVG & COUNT Across Time." Red Gate, 2024. https://www.red-gate.com/simple-talk/databases/postgresql/making-temporal-databases-work-part-2-computing-aggregates-across-temporal-versions/
- Paul Jungwirth. These slides. https://github.com/pjungwir/migrating-to-a-temporal-schema

Notes:

- Here are references to other things I mentioned or that seem related.
- But you can't read that, so you should start here.[BACK]
