---
title: "Lazy developer syndrome"
date: "2009-01-22"
categories: 
  - "oracle"
---

To those who don't know me, I'm an Oracle DBA and I also _can_ develop as well. I detest having to work on applications which have the following construct in the code:

```sql
SELECT <stuff>
FROM TABLE
WHERE <something>
FOR UPDATE;
```

In most cases, the above is a sign of developer laziness. They cannot be bothered to write correct code to handle a situation where a row that the user has been working on (unlocked) has been amended by another user in the meantime.

In Oracle you should _lock late and lock short_ (as Dave Ensor says). Only lock a row as you are about to update it. This is optimistic locking.

_Do not_ lock it when you start working on it and keep it locked until you finish the update. This is pessimistic locking. When you write an application that does this, you end up with rows locked out all over the place as people get half way through amending some details and then have to answer the phone, or pop outside for a ciggie, or go to the loo.

Pessimistic locking - just say **NO!**

It isn't difficult, for example, to add a column to the table to hold an update counter (or similar) and to have a trigger update that counter by one each time an UPDATE is performed. The developer would read in the current value along with the data he wishes to display for amendment and the ROWID for the row in question.  Once the user has made the changes and clicked OK (or whatever) what's wrong with running the following statement :

```sql
UPDATE <table>
SET <stuff>=<new_stuff>
WHERE ROWID=:stored_rowid
AND upd_counter=:stored_upd_counter;
```

If it works, job done, slip in a `COMMIT` while no-one is looking, that will release the locks and will save the data permanently to the table with the changes made and the trigger's change to the update counter carried out as well.

If, on the other hand, it returns _no rows updated_, we know some other user got there before us and changed the value in the update counter column. Now, we can handle that by displaying a message to the user and perhaps switching to a new screen showing the user's own changes and the other user's changes and then request that our user chooses to either force through his own changes,  accept the other users changes aborting his own or to redisplay the previous screen with the new data loaded for further amendments.

It's much more friendly for all users of the application but it takes a little more thought on the developer's side of things. It's easier for a lazy developer to simply choose not to think and blindly carry on with his `SELECT ... FOR UPDATE` nonsense.

Cheers.
