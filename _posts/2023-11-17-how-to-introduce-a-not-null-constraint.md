---
layout: post
title: How to Introduce a NOT NULL Constraint in PostgresSQL?
date: 2023-11-16 17:19:00 +0300
tags: postgresql ruby
excerpt: How difficult is adding a <code>NOT NULL</code> constraint to a PostgreSQL table? It is straightforward at first glance, but looking deeper could disrupt your application. Hold on, and prepare to learn the safe way to orchestrate such changes.
---

{% include cover_image.html url="/assets/2023-11-17-how-to-introduce-a-not-null-constraint/cover.jpeg" %}

Sometimes, you need to introduce a `NOT NULL` constraint to a PostgreSQL table. This task could be straightforward at 
first glance. However, it is more challenging than you may think. Hold on, and prepare to read how to sefely 
orchestrate such changes.

# Why is Adding a Not-NULL Constraint Risky?

Consider the following problem. You want to add a not-NULL constraint to the existing `user_id` column of 
the `contacts` table in the PostgreSQL database. The naive approach would be running the following SQL query:

```sql 
ALTER TABLE contacts 
ALTER COLUMN user_id SET NOT NULL;
```

Unfortunately, this operation is not always safe for two reasons:
1. PostgreSQL acquires an `ACCESS EXCLUSIVE` lock, which prevents any writes into the table during the operation.
2. PostgreSQL performs a full scan to guarantee that the constraint is met for all records in the table.

At first, those details may not sound like a show stopper, but imagine you have a big enough table, and it's 
locked for writes while the PostgreSQL is performing a full scan to enforce the constraint. Your application may 
experience an outage. Don't worry! It does not mean that your application is bound to fail. There is a safe way to 
add this constraint.

To introduce a not-NULL constraint, you must ensure the following:
* All the new records satisfy this constraint
* All the existing records satisfy the new constraint.

# New Records

Your code should not create new records that violate the non-NULL requirement. You better know your codebase. 
But to give you a sense of the required change, it could be something like this:

```diff
@contact = Contact.create!(
  value: params[:phone],
  contact_type: :phone_number,
+ user: @current_user
)
```

Additionally, a good idea would be to add a model validation for the field to ensure no new record is added 
without the required `user_id` column.

```ruby
validates :user_id, presence: true 
```

And finally, you need to add a NOT NULL constraint. But what exactly is a "[check constraint]"? The check constraint is 
an additional constraint that could be added to the table's column. The check constraint is an additional constraint 
that could be added to the table's column. The good news about check constraints is that you can avoid validating 
all the records. That means that while the operation still acquires the `ACCESS EXCLUSIVE` lock, it takes a fraction 
of a second without the full scan. Just don't forget to specify `NOT VALID,` which tells PostgreSQL that you're aware 
of the invalid records in the table and don't want to validate them.

```sql
ALTER TABLE contacts 
ADD CONSTRAINT contacts_user_id_not_null 
CHECK (user_id IS NOT NULL) NOT VALID;
```

Defined this way, check constraint will affect only newly created and updated records but allow existing data to be invalid.

# Existing Records

Now, it's time to take care of existing invalid records. Again, you better know your database. Perhaps it could be 
something like this:

```sql 
UPDATE contacts
SET user_id = profiles.user_id
FROM profiles
WHERE contacts.profile_id = profiles.id
  AND contacts.user_id IS NULL;
```

Maybe you don't need such records at all:

```sql 
DELETE FROM contacts
WHERE user_id IS NULL;
```

Use your judgment, but remember that you must get rid of such records eventually. After performing data migration, 
it's safe to enable check constraint validation for all the records.

```sql 
ALTER TABLE contacts 
VALIDATE CONSTRAINT contacts_user_id_not_null;
```

Since the constraint already covers the record inserts and updates, this operation does not block the entire table 
for writing; instead, it acquires a `SHARE UPDATE EXCLUSIVE` lock. Users will still be able to read and write in the table.

# Set NOT NULL for the Column

Is it necessary to add a column constraint? Not really. A check constraint with enabled validation has no 
functional difference compared to setting a column as not null. However, it may be preferable to use the NOT NULL 
column because it is more portable and may play better with validations implemented in your ORM.

```sql 
ALTER TABLE contacts
ALTER COLUMN user_id SET NOT NULL;
```

As previously discussed, such an operation usually acquires an exclusive write lock and performs a full table scan. 
However, when a not-null check constraint is already enforced, Postgress skips the full scan:

> SET NOT NULL may only be applied to a column provided none of the records in the table contain a NULL value 
> for the column. Ordinarily this is checked during the ALTER TABLE by scanning the entire table; however, if a 
> valid CHECK constraint is found which proves no NULL can exist, then the table scan is skipped.

See [ALTER TABLE] documentation for the details.

```sql 
ALTER TABLE contacts
DROP CONSTRAINT contacts_user_id_not_null;
```

And finally, we can drop the check constraint since the “not-null” constraint already covers this restriction.

# Summary 

It is possible to safely add a `NOT NULL` constraint to a PostgreSQL table, but this is not a trivial task 
involving three steps:

1. **First step.** Update application code to prevent new records from violating the `NOT NULL` requirement.
2. **Second step.** Use temporary check constraint:
   * Add check constraints with `NOT VALID` option to introduce the `NOT NULL` constraint without a full table scan.
   * Update existing records to comply with the new constraint.
3. **Third step.** Replace temporary check constraint with `NOT NULL` constraint:
   * Validate the constraint for existing records.
   * Set the column as `NOT NULL`.
   * Drop check constraints.

[check constraint]: https://www.postgresql.org/docs/current/ddl-constraints.html
[ALTER TABLE]: https://www.postgresql.org/docs/12/sql-altertable.html




