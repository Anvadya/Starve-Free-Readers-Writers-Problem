# Starve-Free-Readers-Writers-Problem
A starve free solution for the classical Readers-Writers Problem

The Readers and Writers Problem is one of the classical Inter-Process Communication(IPC) problems of Computer Science. This problem is used to model the way a database is accessed simultaneously by multiple processes.

Several Readers can be allowed to concurrently read files from the database(inclusive access), whereas only a single writer should be allowed to change the records(exclusive access), and during this time no other process should be given access to the database. Hence, the critical region in this problem is the lines of code where a process is writing into the database.

The naive solution to the problem requires us to keep the readers at higher priority than the others, we will start by looking at this approach. This approach is succesful in achieving mutual exclusion but suffers from a major deficit, the writer might starve till the end of time. 

We discuss a slightly modified approach to the naive solution which solves this deficit and also achieves mutual exlusion. We procees to look at slight modifications of this approach to improve the execution speed of the program and we end by looking at how Linux solves this issue (it uses a special synchronisation primitive, the 'readers/writers semaphore'.

Let us havev a brief look at semaphores before diving into the solution.

# Semaphores

`Semaphore` is another mechanism which is used for achieving thread and/or process synchronisation. `Spinnlocks` are useful when the lock has to be acquired for a short time, semaphores should be used whenever we wish to acquire the lock for an extended period of time. 

There are two types of semaphores broadly:
* `Mutex or Binary Semaphore` : Can have only two values viz. 0 and 1
* `Counting Semaphore` : Can have any non-negative value

Let us look at how semaphores are implemented in Linux.

```C
struct semaphore{
  raw_spinlock_t lock;
  unsigned int count;
  struct list_head wait_list;
};
```

The struct defines three things:
* `lock`: Conventionally, all modern architectures provide an implementation of spinlock which forms the basis of every type of synchronisation primitive. (NB: raw_spinlock_t does NOT mean the same as spinlock)
* `count`: This variable stores the count of the semaphore
* `wait_list`: This stores a list of processes waiting to acquire the lock

The Linux kernel provides the following API to manipulate `semaphores`:
```
void down(struct semaphore *sem);
void up(struct semaphore *sem);
int  down_interruptible(struct semaphore *sem);
int  down_killable(struct semaphore *sem);
int  down_trylock(struct semaphore *sem);
int  down_timeout(struct semaphore *sem, long jiffies);
```

We will only look at the working of the `down` and `up` functions here, as they are the only methods we would be using in our discussion.

The `down` function is used to acquiring the semaphore and the `up` function is used to release the semaphore.

Linux implements the down function as follows(I have modified the code to increase readability):
```C
void down(struct semaphore *sem)
{
        unsigned long flags;
        //This variable is passed to functions and stores the relevant flags

        raw_spin_lock_irqsave(&sem->lock, flags);
        //As mentioned above, raw_spin_lock is used to talk to hardware for implementing any synchronisation primitive
        //We are forced to use this method to gurantee atomicity, any other approach cannot gurantee that the process will not 
        //be preempted by a hardware interrupt
        
        //The below code tests whether the semaphore can be acquired or not by checking the value of the count variable
        if (likely(sem->count > 0))
                sem->count--;
                //This is the case when the semaphore can be acquires, i.e. its count is a positive number
        else
                __down(sem);
                //This is called when the semaphore cannot be acquired at the given moment
                //The __down(sem) function sends the process to sleep state and adds it to the list of waiting processes
                
        raw_spin_unlock_irqrestore(&sem->lock, flags);
        //This function ends the critical region
        //Up until this point from raw_spin_lock_irqsave atomicity is guranteed
        //We briefly acquire a spinlock, as this is a very short segment of code, spinlocks can be used to protect it
        //This is crucial to gurantee the correctness of a semaphore
}
```
