# Modified xv6-riscv

## Getting Started

1. Clone the repository and change directory into it.

    ```sh
    git clone git@github.com:SwethaVipparla/modified-xv6-riscv.git
    cd modified-xv6-riscv
    ```

2. Run the shell

    The shell supports Round Robin, First Come First Serve, and Priority Based Scheuling.

    Round Robin is the default implemented scheduling.

    To use Round Robin scheduling, run the following command.  

    ```sh
    make qemu
    ```

    To use First Come First Serve or Priority Based scheduling, run the following command, where `<choice>` can be of the value `FCFS` or `PBS`.

    ```sh
    make qemu SCHEDULER=<choice>
    ```

## Specifications

### 1. Syscall Tracing

1. Added `$U/_strace` to UPROGS to Makefile.

2. Added a new variable `mask` in `kernel/proc.h` and initialised it in `fork.c` function in `kernel/proc.c` to copy the trace mask from the parent to the child process.

    ```c
    np->mask = p->mask;
    ```

3. Implemented a `sys_trace()` function in `kernel/sysproc.c`.  
   This function implements the new system call to assign the value for a new variable `mask`.

   ```c
    uint64
    sys_trace()
    {
        int mask;

        if(argint(0, &mask) < 0)
        return -1;
    
        myproc()-> mask = mask;
        return 0;
    }
    ```

4. Modified `syscall()` function in `kernel/syscall().c` to print the trace output and `syscall_names` and `syscall_num` arrays to help with this.

    ```c
    void syscall(void)
   {
     struct proc *p = myproc();
     int num = p->trapframe->a7;

     int len = syscallnum[num];
     int arr[len];
     
     for (int i = 0; i < len; i++)
       arr[i] = argraw(i);

     if(num > 0 && num < NELEM(syscalls) && syscalls[num]) 
     {
       p->trapframe->a0 = syscalls[num]();
       if(p->mask & 1 << num)
       {
         printf("%d: syscall %s (", p->pid, syscall_names[num]);

         for (int i = 0; i < len; i++)
           printf("%d ", arr[i]);

         printf("\b) -> %d\n", argraw(0));
       }
     }
     else 
     {
       printf("%d %s: unknown sys call %d\n", p->pid, p->name, num);
       p->trapframe->a0 = -1;
     }
   }
   ```

5. Created a user program in `user/strace.c` to generate the user-space stubs for the system call.

    ```c
    int main(int argc, char *argv[]) 
    {
       int i;
       char *nargv[MAXARG];

       if(argc < 3 || (argv[1][0] < '0' || argv[1][0] > '9'))
       {
           fprintf(2, "Usage: %s mask command\n", argv[0]);
           exit(1);
       }

       if (trace(atoi(argv[1])) < 0) 
       {
           fprintf(2, "%s: trace failed\n", argv[0]);
           exit(1);
       }

       for(i = 2; i < argc && i < MAXARG; i++)
       	nargv[i-2] = argv[i];

       exec(nargv[0], nargv);
       exit(0);
    }
    ```

6. Added a stub to `user/usys.pl`, which causes the `Makefile` to invoke the Perl script `user/usys.pl` and produce `user/usys.S`, and a syscall number to `kernel/syscall.h`.

### 2. Scheduling

The default scheduler of xv6-ricv is Round Robin. I have implemented two other schedulers, which are First Come First Serve and Priority Based.

#### 2.1) FCFS

FCFS selects the process with the lease creation time, which is the tick number corresponding to the time the process was created. The process is executed until it is terminated.

1. Modified the Makefile to support `SCHEDULER` macro for the compilation of the specified scheduling algorithm. Here

    ```makefile
    ifndef SCHEDULER
        SCHEDULER:=RR
    endif

    CFLAGS+="-D$(SCHEDULER)"
    ```

2. Added a variable `timeOfCreation` to `struct proc` in `kernel/proc.h`.

3. Initialised `timeOfCreation` to 0 in `allocproc()` function in `kernel/proc.c`.

4. Implemented scheduling functionality in `scheduler()` function in `kernel/proc.c`, where the process with the least `timeOfCreation` is selected from all the available processes.

    ```c
     #ifdef FCFS

      struct proc* firstProcess = 0;

      for (p = proc; p < &proc[NPROC]; p++)
      {
        acquire(&p->lock);
        if (p->state == RUNNABLE)
        {
          if (!firstProcess || p->timeOfCreation < firstProcess->timeOfCreation)
            {
                if (firstProcess)
                    release(&firstProcess->lock);

                firstProcess = p;
                continue;
            }
        }
        release(&p->lock);
      }

      if (firstProcess)
      {
        firstProcess->state = RUNNING;
        
        c->proc = firstProcess;
        swtch(&c->context, &firstProcess->context);

        c->proc = 0;
        release(&firstProcess->lock);
      }

    #else
    ```

5. Disables `yield()` in `kernel/trap.c` in order to prevent preemption of the process after the clock interrupts in FCFS.

#### 2.2) PBS

PBS is a non-preemptive Priority Based scheduler that chooses the process with the highest priority of execution. When two or more processes have the same priority, the number of times the process has been scheduled is used to determine the priority.
In case the tie still remains, the start-time of the processes are used to break the tie, with the processes having a lower start-time being assigned a higher priority.

1. Added variables `staticPriority`, `runTime`, `startTime`, `numScheduled`, and `sleepTime` to `struct proc` in `kernel/proc.h`.

2. Initialised the above variables with default values in `allocproc()` function in `kernel/proc.c`. 

    ```c
    p->numScheduled = 0;
    p->staticPriority = 60;
    p->runTime = 0;
    p->startTime = 0;
    p->sleepTime = 0;
    ```

3. Added the scheduling functionality for PBS.

    ```md
    niceness = Int ((Ticks spent in (sleeping) state / Ticks spent in (running + sleeping) state) * 10)
    ```

    ```md
    Dynamic Priority = max(0, min(Static Priority - niceness + 5, 100))
    ```

    ```c
    #ifdef PBS

      struct proc* process = 0;
      int dp = 101;
      for (p = proc; p < &proc[NPROC]; p++)
      {
        acquire(&p->lock);

        int niceness = 5;

        if (p->numScheduled)
        {
          if (p->sleepTime + p->runTime != 0)
            niceness = (p->sleepTime / (p->sleepTime + p->runTime)) * 10;
          else
            niceness = 5;
        }

        int val = p->staticPriority - niceness + 5;
        int tmp = val < 100? val : 100;
        int processDp = 0 > tmp? 0 : tmp;

        int flag1 = (dp == processDp && p->numScheduled < process->numScheduled);
        int flag2 = (dp == processDp && p->numScheduled == process->numScheduled && p->timeOfCreation < process->timeOfCreation);

        if (p->state == RUNNABLE)
        {
          if(!process || dp > processDp || flag1 || flag2)
          {
            if (process)
              release(&process->lock);

            process = p;
            dp = processDp;
            continue;
          }
        }
        release(&p->lock);
      }

      if (process)
      {
        process->numScheduled++;
        process->startTime = ticks;
        process->state = RUNNING;
        process->runTime = 0;
        process->sleepTime = 0;
        c->proc = process;
        swtch(&c->context, &process->context);
        c->proc = 0;
        release(&process->lock);
      }

    #endif
    ```

4. Added `set_priority()` function in `kernel/proc.c`.

   ```c
   int set_priority(int priority, int pid)
   {
       struct proc *p;

       for(p = proc; p < &proc[NPROC]; p++)
       {
         acquire(&p->lock);
         
         if(p->pid == pid)
         {
           int val = p->staticPriority;
           p->staticPriority = priority;

           p->runTime = 0;
           p->sleepTime = 0;

           release(&p->lock);

           if (val > priority)
               yield();
           return val;
         }
         release(&p->lock);
       }
       return -1;
   }
   ```

5. Created a user program `user/setpriority.c`.

   ```c
   int main(int argc, char *argv[])
    {
        int priority, pid;
        if(argc != 3) 
        {
            fprintf(2,"Wrong number of arguments\n");
        exit(1);
        }

        priority = atoi(argv[1]);
        pid = atoi(argv[2]);

        if (priority < 0 || priority > 100)
        {
            fprintf(2,"Invalid: Priority should range from 0 to 100\n");
            exit(1);
        }
        set_priority(priority, pid);
        exit(1);
    }

6. Added `set_priority()` system call in `kernel/sysproc.c`.

    ```c
    uint64
   sys_set_priority()
   {
     int pid, priority;
     if(argint(0, &priority) < 0)
       return -1;
     if(argint(1, &pid) < 0)
       return -1;

     return set_priority(priority, pid);
   }
   ```

#### 2.3) MLFQ

In MLFQ, if a process voluntarily relinquishes CPU control, it is removed from the queue.
When it is ready to rejoin, it is added to the end of the same queue.

So, if a process, just relinquishes control before its time slice gets over, then it will relinquish control. By this, it will always remain in the top priority and keep getting executed in the main queue.


### 3) Procdump

1. Added a variable `totalRunTime` to calculate the total run time of the process in `kernel/proc.h` and a variable `endTime` in `kernel/proc.c`  which is initialized once the process' state is changed to Zombie.

2. Modified `procdump()` function in `kernel/proc.c` to reflect the new output.

    ```c
    #ifdef PBS
        printf("PID   Priority\tState\t  rtime\t wtime\tnrun\n");
    #else
    #ifdef MLFQ
        printf("PID   Priority\tState\t  rtime\t wtime\tnrun\n\tq0\tq1\tq2\tq3\tq4");
     #else
        printf("PID   Priority\tState\t  rtime\t wtime\tnrun\n");
    #endif
    #endif
     ```

## Benchmark Testing

A new file `user/schedulertest.c` was added to test the implemented schedulers.

On running the command on 3 CPUs, the following were observed:

### Round Robin

Average rtime 21,  wtime 124

### First Come First Serve

Average rtime 57,  wtime 73

### Priority Based Scheduling

Average rtime 26,  wtime 109
