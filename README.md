# Hash Hash Hash
Hash Hash Hash implements a hash table in 3 different ways:
1. base: Single-threaded hash table, no possibility for race condition
2. v1: Multi-threaded hash table with coarse locking granularity
3. v2: Multi-threaded hash table with fine locking granularity

## Building
```shell
make
```

## Running
```shell
./hash-table-tester -t num_threads -s num_entries_per_thread
```
default (no option) # of threads = 4
default # of entries per thread = 25000

Which will print out information about the the performance of the base, v1, and v2 implementations of the hash table.


## First Implementation
In the `hash_table_v1_add_entry` function, I added an add_entry_mutex, which was initialized alongside the hash table itself,
and locked it right at the start of the function using:

pthread_mutex_lock(&add_entry_mutex);

The lock was then unlocked at the either after an existing list_entry has been updated (which would immmediately return) 
or at end of the function, after the new list_entry was added to the bucket.
This was done using:

pthread_mutex_unlock(&add_entry_mutex)

This implementation definitely guaruntees correctness as the entire critical section (the add entry function) was locked as a whole
This means that each individual thread will all have to execute the add entry function at different times after they have obtained the lock, 
so there is no possibility for a race condition.

### Performance
```shell
./hash-table-tester -t num_threads -s num_entries_per_thread
```
Version 1 is a little slower than the base version. As we will have locking overhead in Version 1 where in the base version,
we don't lock anything and just run 1 thread the whole time. 
In Version 1, the multi-threading is pretty much nullified as every time that the hash table adds an entry 
(which is going to be most of the program), only 1 thread can run it at a time. 
Because of this fact alone, Version 1 would pretty much be at the same speed as base. 
Then ON TOP OF THIS, we have overhead which comes from locking and unlocking the mutex every time the add entry function is called, 
making it slower than the base version.

This can be seen in a test run using ./hash-table-tester -t 4 -s 50000:

Hash table base: 251,467 usec
  - 0 missing
Hash table v1: 599,103 usec
  - 0 missing
Hash table v2: 82,966 usec
  - 0 missing

Where v1's performance is more than double of that of the base implementation.

## Second Implementation
Each hash_table_entry struct now holds a:

pthread_mutex_t lock, which is initialized when each hash_table_entry is allocated to the hash table.

In the `hash_table_v2_add_entry` function, I locked each individual hash_table_entry's lock, 
instead of the locking the whole add_entry function.
This makes it so the only time that another thread would have to wait for a lock 
would be if it was going to add an entry to the exact same bucket.

Right after hash_table_entry is obtained by get_hash_table_entry() and the corresponding list_head is grabbed,
the hash_table_entry is then locked by:

pthread_mutex_lock(&hash_table_entry->lock);

Now, that specific hash_table_entry can only be modified with the thread that has its lock until it has completely updated the entry.
After the entry is updated, its lock is then unlocked by the process that held it.

The lock was then unlocked at the either after an existing list_entry has been updated (which would immmediately return) 
or at end of the function, after the new list_entry was added to the bucket. Just like V1.
This was done using:

pthread_mutex_unlock(&hash_table_entry->lock);



### Performance
```shell
./hash-table-tester -t num_threads -s num_entries_per_thread
```
Version 2's performance is much faster than base (and V1 inherently). This is because of the face mentioned previously that 
V2's locks have a much more fine granularity, as each individual (of the 4096) hash_table_entry has its own lock, so a thread will only have to wait to add an entry to a bucket if another thread happens to have the lock for that exact same bucket. So with a proper hash function, each thread would only have at max an n-1/4096 (n => number of threads) of adding a new entry to a bucket which has its lock already held by another thread. This is MUCH better than V1, as only 1 thread could add an entry to the whole table at a time. Becuase there is a very high chance of each thread being able to add an entry without waiting for the lock, it is much closer to an execution time of base/threads, as each thread can do work in parallel. However, there is still a slight chance of waiting for a lock, AND there is added overhead for locking and unlocking, so time is closer to base/(threads-1). 

Test runs using 

1. ./hash-table-tester -t 4 -s 50000 | Avg time improvement over base:

Generation: 34,568 usec
Hash table base: 224,447 usec
  - 0 missing
Hash table v1: 575,395 usec
  - 0 missing
Hash table v2: 78,805 usec
  - 0 missing

Hash table base: 239,314 usec
  - 0 missing
Hash table v1: 727,495 usec
  - 0 missing
Hash table v2: 81,615 usec

Generation: 34,758 usec
Hash table base: 208,349 usec
  - 0 missing
Hash table v1: 502,346 usec
  - 0 missing
Hash table v2: 68,088 usec
  - 0 missing

## Cleaning up
```shell
make clean
```

In order to clean up locks, in V1 and V2, pthread_mutex_destroy was used either before the loop (V1) or during the loop that cleans up the individual hash_table_entry s (V2).