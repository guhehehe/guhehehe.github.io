---
title: "On A Frustrating Debugging Experience"
tags:
  - story telling
---

Sometimes, seemingly hard bugs can have surprisingly simple and stupid root.

Recently I had been working on a Spark job that keeps failing. The job was just some normal data loading and processing,
few hundreds of GBs of load, nothing fancy. It was running on a 3 nodes cluster running Debian linux, 12 cores
each, with 35G RAM for two of them and 25G for the thrid. I setup the cluster so the worker can use 30G on the two
machines with larger RAM and 20G on the smaller one. I also set the executor memory to 10G so there would be 3
executors running on each of the 35G machines in parallel, and 2 on the 25G machine. The symptom was quite
straightforward: after running for 20-30 minutes, two of the workers just crashed.

The first thing is always to check the log. I checked the worker log and the application log, they seemed to be fine,
no error messages at all, even when the worker crashed. I started to suspect that there were memory leak and the OS
killed the application because of it. I hooked the machines with VisualVM, periodically get the heap dump and check
the memory usage after each GC, and the memory footprint seemed to be perfect. A bit of `vmstat` also showed that the
system wasn't being crazy.

So now I had ruled out the application exception and memory leak, and started to believe that OS was doing some nasty
job. I checked dmesg and bunch of other system logs, but there was nothing! Ok, maybe the system won't log anything
under some special circumstances, for example, the job exceeded some system limit. `ulimit -a` proved me wrong. Then I
ran the job for another serveral times, and noticed a seemingly promising clue which was proven to be misleading later
on: every time, the worker crashed right after the job used up the free memory and started to claim memory from the
page cache. Maybe the OS won't release the page cache? This is quite unlikely, but given the clue I found it sounded
like a good justification. I asked an OS expert friend, and he said OS might lock the page if it is dirty. So I ran the
job again(remember it takes the job 20-30 minutes to reproduce the bug), monitored the dirty pages and they were at a
level of 1000K max during the run, couldn't be more normal.

At this point I really ran out of ideas. So I just flushed the page cache with `echo 3 > /proc/sys/vm/drop_caches` and
hoped it would work automagically, but you are right, it crashed on me again. So far I was getting a bit frustrating so
I decided to perform a surgery to the worker. I attached `strace` to the running processes and threads, and found out
that the worker was killed by `SIGKILL`. Now who the hell killed my worker?  Googled around a little bit and found that
auditd or system tap can be used to trace the issuer of a signal. But instead of installing them and keep debugging, I
checked the system logs again. I don't even know why I was doing that, probably just couldn't believe that there was
nothing logged for such events. I went trough the syslog from the beginning to the end, then from the end to the
beginning, then from a random line to another random line... Wait!  "...CRON:...kill-stale-jvms...", this line was
super suspicious!

I opened the cron job, it was a script that kills the dangling JVMs created by Mapreduce jobs, it
uses a series of `ps` and `grep` command to find the Mapreduce JVM processes, then pipe them through a `sed` command to
extract the application id of the job, then use the job id to find the pid and kill the process. Here is the bash code
with irrelevant parts removed:

```sh
JVMS_COMMAND="ps -ef |grep -v grep |grep java |\
grep -v NodeManager |grep -v DataNode |grep -v '/bin/bash'"

function list_processes() {
    eval "$JVMS_COMMAND" |\
    sed 's/^.*\(application_[0-9]*_[0-9]*\)\/.*$/\1/' |sort
}

function find_pid() {
    eval "$JVMS_COMMAND" |grep "$1" |awk '{print $2}'
}

for PROCESS in $(list_processes); do
    PID=$(find_pid $PROCESS)
    COMMAND="kill -9 $PID"
done
```

As you can see, the code lists a bunch of JVM processes, matches some pattern in each line of the outputs and replace
the whole line with the first occurrence. But `sed` by default prints everything, even when the match fails. If the
match fails, it will print the raw unmatched line as is. In my case, the line of Spark worker JVM gets printed and
fed to the for loop, which breaks the line into words, and eventually the pid of the Spark worker gets found and
killed. To fix it, just silent `sed` to make it not print anything on match fail:
`sed -n 's/^.*\(application_[0-9]*_[0-9]*\)\/.*$/\1/p'`

So lessons learnt from this. I was so convinced that if a program fails, it must be either the program itself, or the
operating system, even when there was nothing logged in the system logs, which I should've pay enough attention about
what it means. OSes are mature and extensively tested in production, it would be super rare that events such as process
being killed is not logged if never, so just trust the log, if no log shows that it's the OS did it, then it didn't
do it. Also, when I start to believe that the OS killed my job and didn't find any clue in the logs, I should've  gone
directly to figure out who killed the process, that's the right track to take.
