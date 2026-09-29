---
title: Shell Lab
description: A job-control Unix shell — signals, process groups, and the fork/addjob race.
---

# Shell Lab · Building a Shell

<p class="article-meta">Processes & signals <span class="dot">·</span> Keywords: fork, signals, process groups, reaping <span class="dot">·</span> <a href="https://github.com/LeeXSean/csapp-labs/blob/main/Shell_Lab/shlab-handout/tsh.c">tsh.c</a></p>

!!! success "Verified locally"
    All **16 traces** match `tshref` (PID-normalized) — `sdriver.pl -s ./tsh` vs `-s ./tshref`.

`tsh` is a small but complete Unix shell: it launches programs, tracks them as **jobs**, forwards keyboard signals, and supports `fg` / `bg` / `jobs` / `quit`. The code is short. The hard part is that it is **concurrent** — a `SIGCHLD` can fire between any two instructions, and the correctness of the whole thing rests on controlling exactly *when* that is allowed to happen.

!!! abstract "The assignment"
    Fill in seven functions on top of a provided read/eval loop and job-list library:

    - `eval` — parse a line; run a built-in, or `fork`/`execve` an external command;
    - `builtin_cmd`, `do_bgfg` — the built-ins `quit`, `jobs`, `fg`, `bg`;
    - `waitfg` — block until the foreground job leaves the foreground;
    - `sigchld_handler`, `sigint_handler`, `sigtstp_handler` — reap children and forward Ctrl-C / Ctrl-Z.

    Every job is one of three states, and at most **one** job is in the foreground at a time:

## Job states

``` text
   (at most one job is FG at any moment)

   FG  -- Ctrl-Z (SIGTSTP) -->  ST
   ST  -- fg -->  FG
   ST  -- bg -->  BG
   BG  -- fg -->  FG

   - a new command starts a job as FG, or as BG if it ends in '&'
   - when any job exits or is killed, it leaves the job list
```

<p class="code-source">Source: <a href="https://github.com/LeeXSean/csapp-labs/blob/main/Shell_Lab/shlab-handout/tsh.c#L24-L26"><code>tsh.c</code> · FG / BG / ST L24–L26</a></p>

Only a foreground job blocks the prompt; background and stopped jobs live on in the job list until they finish.

---

## eval: fork, exec, and the masking dance { data-toc-label="eval" }

For an external command, `eval` forks a child and `execve`s the program. The subtlety is entirely in the **signal mask** around those calls:

``` c
void eval(char *cmdline) {
    char *argv[MAXARGS]; int bg; pid_t pid; struct job_t *job_pos;
    sigset_t mask_all, mask_one, prev_one;

    bg = parseline(cmdline, argv);
    if (argv[0] == NULL)
        return;

    sigfillset(&mask_all); sigemptyset(&mask_one); sigaddset(&mask_one, SIGCHLD);

    if (!builtin_cmd(argv)) {
        sigprocmask(SIG_BLOCK, &mask_one, &prev_one);
        if ((pid = fork()) == 0) {
            setpgid(0,0); sigprocmask(SIG_SETMASK, &prev_one, NULL);
            if (execve(argv[0], argv, environ) < 0) {
                printf("%s: Command not found\n", argv[0]);
                exit(1);
            }
        }

        sigprocmask(SIG_BLOCK, &mask_all, NULL);
        if (!bg) {
            addjob(jobs, pid, FG, cmdline); waitfg(pid);
        } else {
            addjob(jobs, pid, BG, cmdline);
            job_pos = getjobpid(jobs, pid);
            printf("[%d] (%d) %s", job_pos->jid, job_pos->pid, job_pos->cmdline);
        }
        sigprocmask(SIG_SETMASK, &prev_one, NULL);
    }

    return;
}
```

<p class="code-source">Source: <a href="https://github.com/LeeXSean/csapp-labs/blob/main/Shell_Lab/shlab-handout/tsh.c#L170-L212"><code>tsh.c</code> L170–L212</a> (reformatted to fit)</p>

`SIGCHLD` is blocked **before** the `fork`, with the previous mask saved in `prev_one`, so from that point until the final restore the parent cannot be interrupted by a dying child. The child immediately puts itself in its **own process group** with `setpgid(0,0)`. That is what makes `kill(-pid, ...)` target exactly this job, so Ctrl-C at the keyboard hits the foreground job's group instead of the shell. Before `execve`, the child restores the original mask, so the new program starts with a clean, unblocked signal state.

The parent blocks **all** signals while it touches the shared job list, mirroring the handler, so the two can never corrupt `jobs` concurrently. The two paths reach the final restore differently. A background job restores `prev_one` immediately after `addjob` and the job report. A foreground job first enters `waitfg`, where `sigsuspend` temporarily opens the controlled window for `SIGCHLD`, and only reaches the restore after that job has left the foreground. Either way `SIGCHLD` stays blocked across the entire `fork` to `addjob` window.

### The race this prevents

Why block `SIGCHLD` before `fork` rather than after `addjob`? Because a short-lived child can **die before the parent gets scheduled again**. If `SIGCHLD` were deliverable then, the handler would run `deletejob(pid)` for a job that `addjob` has not yet inserted — the delete is a no-op, and the job leaks into the list forever.

Blocking `SIGCHLD` until after `addjob` forces the only correct order — *insert, then reap*:

``` text
   parent (tsh)                              child / kernel
   -------------                             --------------
   block SIGCHLD
   fork()  ---------------------> child: setpgid; restore mask
                                  execve; exit -> SIGCHLD pending
   SIGCHLD remains blocked                              |
   block all signals; addjob(pid)                       |
   allow SIGCHLD only after insertion:                  |
     FG: waitfg -> sigsuspend(empty)                    |
     BG: restore previous mask                          |
        <--------------- deliver SIGCHLD ---------------+
   sigchld_handler -> deletejob  (job already registered: no leak)
```

<p class="code-source">Source: <a href="https://github.com/LeeXSean/csapp-labs/blob/main/Shell_Lab/shlab-handout/tsh.c#L186-L207"><code>tsh.c</code> · eval race window L186–L207</a></p>

## waitfg: sigsuspend, not a spin loop { data-toc-label="waitfg" }

A foreground job must block the prompt until it exits or is stopped. The tempting `while (fg) ;` busy-loop burns a core; `while (fg) pause();` has a fatal race (the child can exit between the `fg` test and `pause`, and then `pause` sleeps forever). The correct primitive is `sigsuspend`, which **atomically** installs a mask and waits:

``` c
void waitfg(pid_t pid) {
    struct job_t *job_cur; sigset_t mask_all;

    sigemptyset(&mask_all);
    while ((job_cur = getjobpid(jobs, pid)) != NULL && job_cur->state == FG) {
        sigsuspend(&mask_all);
    }

    return;
}
```

<p class="code-source">Source: <a href="https://github.com/LeeXSean/csapp-labs/blob/main/Shell_Lab/shlab-handout/tsh.c#L351-L362"><code>tsh.c</code> L351–L362</a> (reformatted to fit)</p>

An **empty** mask means that while suspended, *all* signals, including `SIGCHLD`, are unblocked. `sigsuspend` installs that mask, sleeps until any signal arrives, then restores the previous all-blocked mask as one atomic step. When `SIGCHLD` reaps the foreground child, the handler flips its state, the loop condition fails, and `waitfg` returns.

Recall that `eval` had blocked everything before calling `waitfg`, so this empty-mask `sigsuspend` is precisely the controlled window in which the foreground child's `SIGCHLD` is allowed through.

## Reaping children: the SIGCHLD handler { data-toc-label="SIGCHLD handler" }

One `SIGCHLD` may stand for **several** children, because standard signals do not queue. So the handler reaps in a loop with `WNOHANG | WUNTRACED`, draining every child that is ready without ever blocking:

``` c
void sigchld_handler(int sig) {
    int olderrno = errno;
    sigset_t mask_all, prev_all; pid_t pid; int jid; struct job_t *job_cur; int status;

    sigfillset(&mask_all);
    while ((pid = waitpid(-1, &status, WUNTRACED | WNOHANG)) > 0) {
        sigprocmask(SIG_BLOCK, &mask_all, &prev_all);
        if (WIFEXITED(status)) {
            deletejob(jobs, pid);
        } else if (WIFSIGNALED(status)) {
            jid = pid2jid(pid);
            sio_puts("Job [");
            sio_putl((long)jid);
            sio_puts("] (");
            sio_putl((long)pid);
            sio_puts(") terminated by signal ");
            sio_putl((long)WTERMSIG(status));
            sio_puts("\n");
            deletejob(jobs, pid);
        } else if (WIFSTOPPED(status)) {
            jid = pid2jid(pid);
            sio_puts("Job [");
            sio_putl((long)jid);
            sio_puts("] (");
            sio_putl((long)pid);
            sio_puts(") stopped by signal ");
            sio_putl((long)WSTOPSIG(status));
            sio_puts("\n");
            job_cur = getjobpid(jobs, pid);
            job_cur->state = ST;
        }
        sigprocmask(SIG_SETMASK, &prev_all, NULL);
    }

    errno = olderrno;
    return;
}
```

<p class="code-source">Source: <a href="https://github.com/LeeXSean/csapp-labs/blob/main/Shell_Lab/shlab-handout/tsh.c#L375-L418"><code>tsh.c</code> L375–L418</a> (reformatted to fit)</p>

A handler can run at any point in `main`, so it **saves and restores `errno`**. Otherwise a `waitpid` here could silently clobber the `errno` that some interrupted library call was about to read.

`WNOHANG` makes `waitpid` return `0` immediately when no more children are ready, so the shell is never blocked; `WUNTRACED` also reports children that just **stopped**, not only those that exited. The loop keeps going until every ready child is handled. Every job-list mutation is guarded by blocking all signals, mirroring `eval`.

`printf` is **not** async-signal-safe. The handler reports through the provided `sio_*` routines instead, which are thin wrappers over `write(2)` and safe to call from a signal handler.

The three `waitpid` outcomes map cleanly onto the job model: **exited** or **killed** → `deletejob`; **stopped** → mark `ST` and keep it.

## Forwarding the keyboard: SIGINT & SIGTSTP { data-toc-label="SIGINT / SIGTSTP" }

The shell itself receives Ctrl-C and Ctrl-Z. Its job is not to die, but to **relay** them to the foreground job's entire process group:

``` c
void sigint_handler(int sig) {
    int olderrno = errno;
    pid_t pid;

    if ((pid = fgpid(jobs)) > 0) { kill(-pid, SIGINT); }

    errno = olderrno;
    return;
}
```

<p class="code-source">Source: <a href="https://github.com/LeeXSean/csapp-labs/blob/main/Shell_Lab/shlab-handout/tsh.c#L425-L436"><code>tsh.c</code> L425–L436</a> (reformatted to fit)</p>

The **negative** pid means "send to the whole process group `pid`." Because every job got its own group back in `eval` (`setpgid`), this reaches the foreground job and its children, and no one else. `sigtstp_handler` is identical but sends `SIGTSTP`. If there is no foreground job, both handlers do nothing.

## do_bgfg: resuming stopped jobs { data-toc-label="do_bgfg" }

`bg %n` and `fg %n` (or by PID) resume a stopped job by sending `SIGCONT` to its group; the only difference is whether the shell then waits:

``` c
void do_bgfg(char **argv) {
    char *ptr; pid_t pid; int jid; struct job_t *job_cur;

    ptr = argv[1];
    if (ptr == NULL) {
        printf("%s command requires PID or %sjobid argument\n", argv[0], "%");
        return;
    }
    if (*ptr == '%') {
        ptr++;
        if ((jid = atoi(ptr)) == 0) {
            printf("%s: argument must be a PID or %sjobid\n", argv[0], "%");
            return;
        }
        if ((job_cur = getjobjid(jobs, jid)) == NULL) {
            printf("%s: No such job\n", argv[1]);
            return;
        }
    } else {
        if ((pid = atoi(ptr)) == 0) {
            printf("%s: argument must be a PID or %sjobid\n", argv[0], "%");
            return;
        }
        if ((job_cur = getjobpid(jobs, pid)) == NULL) {
            printf("(%d): No such process\n", pid);
            return;
        }
    }

    if (!strcmp(argv[0], "bg")) {
        if (job_cur->state == ST) {
            job_cur->state = BG;
            kill(-(job_cur->pid), SIGCONT);
            printf("[%d] (%d) %s", job_cur->jid, job_cur->pid, job_cur->cmdline);
        }
    } else if (!strcmp(argv[0], "fg")) {
        if (job_cur->state == BG || job_cur->state ==ST) {
            job_cur->state = FG;
            kill(-(job_cur->pid), SIGCONT); waitfg(job_cur->pid);
        }
    }

    return;
}
```

<p class="code-source">Source: <a href="https://github.com/LeeXSean/csapp-labs/blob/main/Shell_Lab/shlab-handout/tsh.c#L295-L346"><code>tsh.c</code> L295–L346</a> (reformatted to fit)</p>

`do_bgfg` parses the two argument forms: a job id introduced by `%`, looked up with `getjobjid`, or a bare PID, looked up with `getjobpid`. Both forms report an error and return when the argument is missing, unparsable, or names no job. From there `bg` restarts a **stopped** job in the background and prints its job line, while `fg` moves a background or stopped job to the foreground, sends `SIGCONT` to its process group, and blocks in `waitfg` until the job leaves the foreground again.

!!! note "The three rules a signal handler must obey"
    Everything above follows from three constraints that make signal code correct rather than merely working-on-my-machine:

    1. **Save and restore `errno`** — a handler must be transparent to the code it interrupts.
    2. **Call only async-signal-safe functions** — `write`/`sio_*`, never `printf`/`malloc`.
    3. **Block signals around shared-state changes** — the job-list *mutations* that race (`addjob` in `eval`, the handler's `deletejob` and state updates) run under a full signal mask, so `main` and the handler can't corrupt the list at the same instant. The limit of this implementation is elsewhere: `do_bgfg` runs its lookup and its `job_cur->state` write without masking, so a `SIGCHLD` arriving at that moment is not excluded.
