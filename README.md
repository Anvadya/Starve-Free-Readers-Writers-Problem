# Starve-Free-Readers-Writers-Problem
A starve free solution for the classical Readers-Writers Problem

The Readers and Writers Problem is one of the classical Inter-Process Communication(IPC) problems of Computer Science. This problem is used to model the way a database is accessed simultaneously by multiple processes.

Several Readers can be allowed to concurrently read files from the database(inclusive access), whereas only a single writer should be allowed to change the records(exclusive access), and during this time no other process should be given access to the database. Hence, the critical region in this problem is the lines of code where a process is writing into the database.

The naive solution to the problem requires us to keep the readers at higher priority than the others, we will start by looking at this approach. This approach is succesful in achieving mutual exclusion but suffers from a major deficit, the writer might starve till the end of time. 

We discuss a slightly modified approach to the naive solution which solves this deficit and also achieves mutual exlusion. We procees to look at slight modifications of this approach to improve the execution speed of the program and we end by looking at how Linux solves this issue (it uses a special synchronisation primitive, the 'readers/writers semaphore'.

Let us havev a brief look at `semaphores` before diving into the solution.

# Semaphores

`Semaphore` is another mechanism which is used for achieving thread and/or process synchronisation. `Spinnlocks` are useful when the lock has to be acquired for a short time, semaphores should be used whenever we wish to acquire the lock for an extended period of time. 

There are two types of semaphores broadly:
* `Mutex or Binary Semaphore` : Can have only two values viz. 0 and 1
* `Counting Semaphore` : Can have any non-negative value

Let us look at how `semaphores` are implemented in Linux.

```C
struct semaphore{
  raw_spinlock_t lock;
  unsigned int count;
  struct list_head wait_list;
};
```

The struct defines three things:
* `lock`: Conventionally, all modern architectures provide an implementation of `spinlock` which forms the basis of every type of synchronisation primitive. (NB: raw_spinlock_t does NOT mean the same as spinlock)
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
We acquire a `spinlock` at the start of the procedure to ensure the atomicity of the entire process. Interrupts are disabled and the processor might disable the memory bus in case of a multicore system. The `if-else` clause checks whether the lock can be acquired, if it can be, we reduce the count of the semaphore and acquire the semaphore; if not, then another process `__down` is called to send the process to sleep state and add the process to the list of waiting processes.

The `up` function's code is very similar to the code of the `down` function presented above.
```C
void up(struct semaphore *sem)
{
        unsigned long flags;
        //performs the same role as down, this is sent to the raw_spin_lock functions, in case there is any
        //error in acquiring the spinlock, the appropriate flags are set in the flags variable

        raw_spin_lock_irqsave(&sem->lock, flags);
        //Acquiring the spinlock to gurantee atomicity
        
        if (likely(list_empty(&sem->wait_list)))
                //Checking if there is no process who is waiting on this semaphore
                //If none is waiting then we increment the count of the semaphore
                sem->count++;
        else
                //If there are processes sleeping on the semaphore, then we must call the __up function
                //The __up function will wake a process who is sleeping on the semaphore
                __up(sem);
                
        raw_spin_unlock_irqrestore(&sem->lock, flags);
        //Leaving the spinlock
}
```
`Semaphores` have to be initialised at the time of creation.

# The Naive solution:
In our solution, we define 3 functions:
```
enterReader(pID)
exitReader(pID)
enterWriter(pID)
```
We would be using two `Mutex` here,
* `read_mutex`: This Mutex would ensure only one Reader reads and changes the value of the variable count.
* `write_mutex`: This Mutex would ensure that no Reader can enter the critical section once the Writer enters it.
A shared variable `rCount` would store the numbers of readers currently in their critical section.

Initialisation of the mutex:
```
read_mutex  = new Mutex(1);
write_mutex = new Mutex(1);
```

Let us analyse the codes for the reader:

The reader needs to call enterReader function before entering its critical region
```C
void enterReader(pid reader){
  wait(read_mutex, reader);
  //Ensuring only one reader enters this section at once
  
  ++rCount; //Incrementing the count of readers inside the Critical Section
  
  if(rCount == 1) wait(write_mutex, reader);
  //Ensuring the writer does not enter after any reader has entered
  //Done only once to avoid unncecessary system calls which reduce the performance
  
  signal(read_mutex);
  //Freeing the read_mutex so other processes can do their work
  
  return;
}
```

The reader needs to call exitReader function after executing its critical region
```C
void exitReader(pid reader){
  wait(read_mutex, reader);
  //Ensuring only one reader enters this section at once
  
  --rCount;
  //Reducing the number of readers in their critical region(as one process has left it)
  
  signal(read_mutex);
  //Freeing the read_mutex so other processes can do their work
  
  if(count == 0) signal(write_mutex);
  //This statement would allow the writer to enter its critical region
  //This statement is after the signal(read_mutex) to ensure that readers at a higher priority
  //Consider the sequence RWRR, where all WRR come at once (R represents a Reader, W represents a Writer)
  //If signal was inside the locked region, the Writer would run after the first R even though 2 more R were waiting on it
  //As the signal is outside, the Writer can enter its critical region only if there are no readers ready to enter the critical region
    
  return;
}
```

The Writer needs to call this function before entering
```C
void enterWriter(pid writer){
  wait(write_mutex, writer);
  //If a process can enter here, this implies that count was equal to 0
  //Hence, it is safe to enter the critical region now
  //(No other reader or writer can enter the critical section now)
  
  /* 
    CRITICAL REGION
  */
  
  signal(write_mutex);
  //Freeing the write_mutex so that other readers/writers may run
  
  return;
}
```

We can clearly see a major drawback in this solution, let us assume that Readers keep on coming, the Writer will have to wait until there is no reader in the critical region but readers are given entry as soon as they reach, i.e. they are kept at priority (this is done because having multiple readers simultaneously reading the records makes efficient use of parallellism and thus increases the performance of the system) the Writer might never get a chance to enter the critical region or it enters the critical section after a long period of time which might be an unwanted consequence of this solution.

# Starve Free Solution

The solution is largely the same. We define the same 3 functions as above, we introduce a new `entry_mutex` whose value is initialised to 1.

The reader needs to call enterReader function before entering its critical region
```C
void enterReader(pid reader){
  wait(entry_mutex, reader);
  
  wait(read_mutex, reader);
  //acquiring the read_mutex for guaranteeing mutual exclusion 
  
  ++rCount; //Incrementing the count of readers inside the Critical Section
  if(rCount == 1) wait(write_mutex, reader);
  //The above statement acquires the write_mutex to prevent writers from enetring while readers are still working
  //in their critical sections
  
  signal(read_mutex);
  
  signal(entry_mutex);
  
  return;
}
```

The reader needs to call exitReader function after executing its critical region
```C
void exitReader(pid reader){
  wait(read_mutex, reader);
  
  --rCount; 
  //Reducing the number of readers in their critical region(as one process has left it)
  
  signal(read_mutex);
  //Freeing the read_mutex so other processes can do their work
  
  if(count == 0) signal(write_mutex);
  
  return;
}
```

The Writer needs to call this function before entering
```C
void enterWriter(pid writer){
  wait(entry_mutex, writer);
  
  wait(write_mutex, writer);
  //If a process can enter here, this implies that count was equal to 0
  //Hence, it is safe to enter the critical region now
  //(No other reader or writer can enter the critical section now)
  
  signal(entry_mutex);
  
  /* 
    CRITICAL REGION
  */
  
  signal(write_mutex);
  //Freeing the write_mutex so that other readers/writers may run
  
  return;
}
```

Let us spend some time in analysing why this particular variation can deal with stravation. Assume that there are 4 readers and 1 writer, they come in the sequence RRWRR (here `R` represents a Reader and `W` represents a Writer).
When Reader 1 comes in, it gets gold of the entry_mutex, sets the `rCount` to 1 and blocks the `write_mutex`. Reader 1 then enters its critical region. Reader 2 does something similar (except blocking the `write_mutex` as the `rCount` has been set to 2 now).
When the Writer comes in, it succesfully gets hold of the `entry_mutex` but gets blocked on the `write_mutex` (as it has been blocked by Reader 1). Hence, writer 1 is now waiting on the `write_mutex`.
Now Reader 3 comes in and tries to acquire `entry_mutex`, this request is denied as the Writer currently holds the `mutex`, so Reader 3 is now waiting on the `entry_mutex`. Reader 4 meets the exact same fate as Reader 3 and waits on the `entry_mutex`.
After some time, Reader 1 and 2 exit their critical regions and call the `exitReader` procedure. The last reader to exit unlocks the `write_mutex`, which in turn wakes up the writer who proceeds to enter its critical region. Before entering the critical region, the writer leaves the `entry_mutex` thereby waking Reader 3, but after incrementing `rCount` to 1, Reader 3 fails to acquire the `write_mutex` as it is currently in posession of the Writer, thereby ensuring Mutual Exclusion between the Readers and Writers. The remaining readers can resume their work once the Writer exits the critical region and unlocks the `write_mutex`.

# Correctness of the Solution
Any starve-free solution must meet three criterias, viz. _Mutual Exclusion_, _Bounded Waiting_ and _Progress_. Let us look at each of them:
## Mutual Exclusion:
The `entry_mutex` and `write_mutex` ensure Mutual Exclusion. The `entry_mutex` ensures that only 1 reader is reading/writing the rCount variable at a time, thereby avoiding race conditions. The `write_mutex` ensures that the writer should not enter its critical region while readers are still working in their critical regions and also ensures that while the readers do not enter their critical region while the writer is working in its critical region.
## Bounded Waiting:
The `entry_mutex` ensures that all the readers who come after the writer will enter their critical regions only after the writer finishes its work. Therefore, there is a guarantee that writers will not wait till eternity before entering their critical regions. Similar arguments also ensure that no Reader needs to wait endlessly before entering its critical region.
## Progress:
The structure of the code ensures that no deadlock can exist and therefore progress is guaranteed.

# Optimisations
Accessing a `semaphore` is a long task as the Operating System has to be called by using the `trap` instruction. This means that a significant overhead is associated with every `semaphore` operation. The code presented above is correct but one might argue that it is not very efficient as the Reader needs to access two semaphores (`entry_mutex` and `read_mutex`) every time it needs to enter the critical region. In a scenario where multiple readers might need to access their critical region (for example, in a server's database) this might prove to be detrimental to the database's performance. 

There exists a way to refactor the above code such that it uses only one `semaphore` for every reader who wishes to enter its critical region, thereby greatly increasing performance. The crux of the idea is to absorb the `read_mutex` into the `entry_mutex` (notice that these two are used consecutively in the `enterReader` function) and using an additional `out_mutex` semaphore. Some additional variables have to introduced, but the overall overhead of accessing variables is almost negligible in comparison to that of `semaphore`. The exact details of the same can be found in the References (present at the bottom).

# R/W Semaphores

The `Linux kernel` provides a synchronisation primitive known as the `Readers/Writers semaphore`. We will have a brief look at the internal organisation of this `semaphore` and then look at the API it provides to the user.

Internally, the `R/W semaphore` also uses `spinlock` (remember that all synchronisation primitives in Linux internally use `raw_spinlock`). Theoretically, the functioning is same as we discussed above, the `semaphore` is provided to increase the ease with which softwares can be written. Since errors in writing the code for `semaphores` are difficult to catch and often lead to unpredictable behaviour, the use of a pre-written library makes the development process more convenient.

The Linux kernel provides following primary [API](https://en.wikipedia.org/wiki/Application_programming_interface) to manipulate `reader/writer semaphores`:

* `void down_read(struct rw_semaphore *sem)` - lock for reading;
* `int down_read_trylock(struct rw_semaphore *sem)` - try lock for reading;
* `void down_write(struct rw_semaphore *sem)` - lock for writing;
* `int down_write_trylock(struct rw_semaphore *sem)` - try lock for writing;
* `void up_read(struct rw_semaphore *sem)` - release a read lock;
* `void up_write(struct rw_semaphore *sem)` - release a write lock;

Hence now the reader just need to call the `down_read` function before entering the critical section and call `up_read` after finishing its work in the critical section. Similarly, the writer needs to call the `down_write` function before entering its critical section and call the `up_write` function after exiting the critical section. The `down_read_trylock` returns whether the lock is available or not, in case the lock is unavailable the process might try doing some other work rather than going to a wait state on the `semaphore`, `down_write_trylock` achieves the same thing for the writers. 

These functions are provided as a part of the system call library provided by the OS.
