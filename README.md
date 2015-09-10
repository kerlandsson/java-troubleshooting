# Troubleshooting Java Applications

This document contains useful commands and code snippets for troubleshooting Java applications in production.

The following assumptions are made.

* The `$pid` variable contains the process ID (PID) to the Java process in question
* `$JAVA_HOME` is set to a JDK installation
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

    for i in `seq 10`;do echo "dump $i";$JAVA_HOME/bin/jstack $pid > /tmp/dump_${pid}_$(date '+%H_%M_%S').txt; sleep 5;done
    
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

##### Full heap dump

Generate a full heap dump (**NB: this will freeze the JVM for the full duration**).

    $JAVA_HOME/bin/jmap -dump:format=b,file=/tmp/heap_dump_$pid.bin $pid



## CPU Usage

##### View CPU usage by thread

Java threads are mapped to native threads on Linux. Use `top -H` to show threads in top.

    top -p $pid -H 
    
The PIDs of the threads can be converted to hexadecimal and matched to the nid=<id> entries in a thread dump. The following command can be used to convert PIDs to hexadecimal on the fly.

    top -p $pid -H -d 3 -b | sed -r 's/^([0-9]+) (.*)/printf "%x  \2" \1/e'
    
Top is used in batch mode which makes it append the output to the console.

##### Dump threads and CPU usage by thread

Dump both thread stacks and CPU usage by thread with PIDs hex-converted to the same file. 

    export fn=/tmp/dump_${pid}_$(date +"%H_%M_%S"); $JAVA_HOME/
    bin/jstack $pid > $fn && top -p $pid -H -d 2 -b -n 2 | sed -r 's/^([0-9]+) (.*)/printf "%x  \2" \1/e' >> $fn
    

## Other

##### JIT compiler statistics

Show number of total compile tasks and information about the most recent task (size, type and method).

    $JAVA_HOME/bin/jstat -printcompilation $pid 3s
    
    