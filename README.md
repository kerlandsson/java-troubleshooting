# Troubleshooting Java Applications

This document contains useful commands and code snippets for troubleshooting (Java) server applications in production.

The following assumptions are made.

* The `$pid` variable contains the process ID (PID) to the Java process in question
* `$JAVA_HOME` is set to a JDK installation
* You are using Oracle's HotSpot JVM
* You are logged in as the same user as the Java process you are troubleshooting is running as
* You are on a Linux machine (most stuff works on *NIX but is untested)

The commands are generally safe to run in production unless otherwise stated.

## General

##### View running Java processes

    $JAVA_HOME/bin/jps -mlvV


## Thread Dumps


##### Dump Thread Stacks

Performance impact is usually minimal, although all threads must reach a safepoint to be able to dump

    $JAVA_HOME/bin/jstack $pid
    
##### Dump several times

Take a thread dump every 5 seconds and save to a timestamped file

    while true;do echo -n ".";$JAVA_HOME/bin/jstack $pid > /tmp/dump_${pid}_$(date '+%H_%M_%S').txt; sleep 5;done
    
##### Dump when CPU usage is high

Take a thread dump every 5 seconds whenever the CPU usage on the whole machine is over 50%.

```
while true
    do echo -n .
    if [ "$(mpstat 1 1|grep '^Average' |awk '{print $3}'|cut -d '.' -f 1)" -gt "50" ];then
        echo dumping
        $JAVA_HOME/bin/jstack $pid > /tmp/dump_${pid}_$(date '+%H_%M_%S').txt
        sleep 5
    fi
done
```
    
    
## Garbage Collector

##### Heap usage and accumulated GC times

Show heap usage by generation and GC duration in seconds accumulated since the JVM started.

    $JAVA_HOME/bin/jstat -gc $pid 3s
    
If you prefer percentages instead of bytes:

    $JAVA_HOME/bin/jstat -gcutil $pid 3s
    
##### Garbage collection cause

Print the cause of the current and the last GC.

    $JAVA_HOME/bin/jstat -gccause $pid 3s


## Heap

##### Object histogram

Show a class histogram with class names, number of instances and total size on the heap.

    $JAVA_HOME/bin/jmap -histo $pid
    
Show only live objects (**NB: this triggers a full GC**):

    $JAVA_HOME/bin/jmap -histo:live $pid
    
Tip: This is a way to explicitly trigger a full GC even though explicit GC has been disabled on the target JVM.


##### Full heap dump

Generate a full heap dump (**NB: this will freeze the JVM for the full duration**).

    $JAVA_HOME/bin/jmap -dump:format=b,file=/tmp/heap_dump_$pid.bin $pid



## CPU Usage

##### View CPU usage by thread

Java threads are mapped to native threads on Linux. Use `top -H` to show threads in top.

    top -p $pid -H 
    
The PIDs of the threads can be converted to hexadecimal and matched to the nid=<id> entries in a thread dump. The following command can be used to convert PIDs to hexadecimal on the fly.

    top -p $pid -H -d 3 -b | sed -r 's/^ *([0-9]+) (.*)/printf "%x  \2" \1/e'
    
Top is used in batch mode which makes it append the output to the console.

##### Dump threads and CPU usage by thread

Dump both thread stacks and CPU usage by thread with PIDs hex-converted to the same file. 

    export fn=/tmp/dump_${pid}_$(date +"%H_%M_%S"); $JAVA_HOME/bin/jstack $pid > $fn && top -p $pid -H -d 2 -b -n 2 | sed -r 's/^ *([0-9]+) (.*)/printf "%x  \2" \1/e' >> $fn
    

## JVM Internals

##### JIT compiler statistics

Show number of total compile tasks and information about the most recent task (size, type and method).

    $JAVA_HOME/bin/jstat -printcompilation $pid 3s
    
##### View lots of performance counters

    $JAVA_HOME/bin/jcmd $pid PerfCounter.print
    
Uses a lowercase `c` (`Perfcounter.print`) on older JVMs.

##### View system properties

     $JAVA_HOME/bin/jcmd $pid VM.system_properties
     
##### View JVM flags

    $JAVA_HOME/bin/jcmd $pid VM.flags
    
    
## Network

##### Who is listening?

Show all listen ports:

    ss -ln

Show which process is listening on port 8989 (requires root):

    ss -lnp "sport = :8989"
     
     
##### What is sent/received?

Show 4 packets as ascii that is sent or received on port 10003 on interface `bond0`.

    tcpdump -A -p -c 4 -i bond0 'port 10003'
    
`tcpdump` is not ubiquitous.
    
##### View network utilization by process

Network utilization on interface `bond0`.

    nethogs bond0
    
`nethogs` is not ubiquitous.

    
## Disk

##### View disk utilization

    iostat -x 2
    
##### Show open files

    lsof -p $pid
    
##### Show free disk space

    df -h
    
##### What is using the disk?

    du -hs <path> 
     
     
## Memory

##### Show used/available RAM

    free -h
    
##### Show processes sorted by memory usage

    top -ba -n 1
    
    
## Other

##### Run a command when something specific is logged

    tail -n0 -F <file> |grep --line-buffered <regex> | <cmd>
    
##### Count total number of threads on the machine
    
    ps -eL|wc -l
    
