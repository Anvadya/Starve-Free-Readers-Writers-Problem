# Starve-Free-Readers-Writers-Problem
A starve free solution for the classical Readers-Writers Problem

The Readers and Writers Problem is one of the classical Inter-Process Communication(IPC) problems of Computer Science. This problem is used to model the way a database is accessed simultaneously by multiple processes.

Several Readers can be allowed to concurrently read files from the database(inclusive access), whereas only a single writer should be allowed to change the records(exclusive access), and during this time no other process should be given access to the database. Hence, the critical region in this problem is the lines of code where a process is writing into the database.

The naive solution to the problem requires us to keep the readers at higher priority than the others, we will start by looking at this approach. This approach is succesful in achieving mutual exclusion but suffers from a major deficit, the writer might starve till the end of time. 

We discuss a slightly modified approach to the naive solution which solves this deficit and also achieves mutual exlusion. We procees to look at slight modifications of this approach to improve the execution speed of the program and we end by looking at how Linux solves this issue (it uses a special synchronisation primitive, the 'readers/writers semaphore'.

Let us havev a brief look at semaphores before diving into the solution.

#Semaphores
