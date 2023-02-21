# Starve-Free-Readers-Writers-Problem
A starve free solution for the classical Readers-Writers Problem

The Readers and Writers Problem is one of the classical Inter-Process Communication(IPC) problems of Computer Science. This problem is used to model the way a database is accessed simultaneously by multiple processes.
Several Readers can be allowed to concurrently read files from the database(inclusive access), whereas only a single writer should be allowed to change the records(exclusive access), and during this time no other process should be given access to the database. Hence, the critical region in this problem is the lines of code where a process is writing into the database.
The naive solution to the problem requires us to keep either the readers or writers at higher priority than the others, this can be achieved as follows:
* Keeping the Readers at higher priority: The writer is not allowed to enter the critical region unless there is no reader who wishes to access the database.
* Keeping the Writers at higher priority: No reader who requests to access the database after the writer's request would be allowed to complete its request until the Writer makes the necessary changes to the database. Hence, the Writer waits for all the current readers to complete their job and then enters the critical region.  

We will first look at the classical solution to this problem which gurantees Mutual Exclusion but does not prevent/avoid starvation. We then present a starve free solution to the above problem for both the cases, viz. when the Reader is at priority and when the Writer is at priority.
