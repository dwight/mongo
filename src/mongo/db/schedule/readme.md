Notes for scheduler and locking
===============================

HLM
---

HLM is the "hierarchical lock manager" (schedule/hlm.h).

The old d_concurrency locks had two levels, top and db-level, i.e. big R and W were the top level
things.

Here we have more levels.  Currently three:

 top
  db
   page

It would not be hard to add an extra level, e.g. collection, if desired.  But maybe if one just 
exclusively locks NSD that happens anyway?

The top and db levels are rwlocks.  Shared state on them means more granular locking exists 
further down.  

The old Lock::GlobalWrite and Lock::GlobalRead are hooked in to lock top exclusively.  Note it's an 
exclusive lock even if 'R' requested.

The old Lock::DBWrite and Lock::DBRead now use LockMid in exclusive mode.  So they result in 
a shared top lock and an exclusive db level lock.

Page Level Locking
------------------

Declare your intent to use page level locking for a code block (and below) with: 

    {    
         GranularForDB(dbOrNS) g;
         ...
    }

This will lock top and the db level in shared state (unless already locked more strictly; recursive
locking IS allowed. That is to say, if someone did a Lock::DBWrite up the call stack somewhere, you 
can still invoke code that uses the granular mechanisms -- it just won't be granular.)

Once this intent has been declared, you are responsible for locking pages you will be touching.  
Typically this is done as: 

    void f(void *mmf_ptr)
    {
        // GranularForDB was done previously and still in scope

        RecLocker::Scoped lk(mmf_ptr);

        // now use mmf_ptr...
    }

You can also lock a mmf_ptr without scoping.  In that case, ~GranularForDB destructor will release them
for you.  

ExtentManager::RecordFor() will automatically lock any record that it returns, so 
generally you don't have to be explicit.

Page Granularity
----------------

Currently, pages are 8KB.  So RecLocker::Scoped(mmf_ptr) will quantize the parameter it receives to an 8K 
multiple.

That said, the current convention assumed in the code would be that for a document, to lock it, 
just lock its starting address, regardless of its size.  As long as all lockers follow that 
rule, you don't have to lock intra-document addresses, even if the document is larger than 8KB.
(Note that if you do lock intra-document, as long as you previously locked the start of the document, 
that is ok, as no one else will have access anyway...)

This is set at 8KB at the moment for a couple reasons:
1) btree buckets (v1 indexes anyway) are 8KB. so with this page sizing for locks, there is no chance 
   someone will mistakenly lock the 2nd half of a btree bucket before the 1st half, potentially causing 
   a deadlock.  this may be impossible as the code is written already anyway though.
2) Speed. The code automatically locks things for you in places (grep for RecLocker). It might as written
   do several locks for a fairly contiguous "object" incidentally.  Likewise the tagging notion becomes 
   faster.

"Tagging"
---------

Pages may be "tagged".  Then, later, one can if appropriate call unlockNonTagged().

The code "tags" locked pages that we mutate.  Thus, unlockNonTagged() basically means "unlock non-mutated pages".
However, you might at times tag a page just to maintain isolation I suppose, so the mechanism is thus named
more generically.

ThreadLocalIntents::push() calls RecLocker::tag(), so basically all mutations are automatically tagged.

Consider for example, the end of checkAndInsert(), which now reads:

        ...
        DEV RecLocker::assertTagged( status.getValue().rec() ); // it was tagged by our write intent declaration...
        RecLocker::unlockNonTagged();
        logOp("i", ns, js);
    }

The intention is to get out of as many locks as possible before doing the logOp work.  Thus we can pipeline
some, while we are logging the op, someone else can begin a write operation on our db.  Since the record we 
just inserted is still locked, it is isolated and not readable until the logOp finishes.

Similarly, if adding keys to N indexes, as we move through the indexes, we can unlockNonTagged (this has 
not yet been hooked in...)

Deadlocks
---------

Generally speaking, there are few provisions for deadlocks as-is.

Things to fix
-------------

1) NSD is locked, so there isn't really parallelism yet, beyond collection level.  It probably makes sense 
to have an NSD-specific locking mechanism: for example on writes we increment nRecords in NSD, but that 
doesn't mean we need to have the NSD exclusively locked for most of our write's duration.

The 8KB granularity doesn't really work with NSD as there may be two adjacent ones within one 8KB page,
resulting in an accidental lock of the other NSD.  However given the above comment on an NSD specific 
mechanism, that would go away anyway.

2) The implementation of the leaf layer of locks is provisional and will need to be better later.  
See llCache in hlm.cpp.

3) The leaf layer of locks are not rwlocks. We might want that.

4) Make RecLocker::tag() faster.  Or rather, profile everything and fix the obvious slow points.

5) Hook into Runner, and make it not keep things locked that it shouldn't keep locked.

Misc
----

* Perhaps when yieldSometimtes is called, we can release a bunch of locks, as that is the contract
for yieldSometimes.  Although maybe Runner just handles everything for us regardless.

