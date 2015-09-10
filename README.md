# Troubleshooting Java Applications

This document contains useful commands and code snippets for troubleshooting Java applications.

The following assumptions are made.

* The `$pid` variable contains the process ID (PID) to the Java process in question
* `$JAVA_HOME` is set to a JDK installation
* You are logged in as the same user as the Java process you are troubleshooting is running as
* You are on a Linux machine (most stuff works on *NIX but is untested)

## General

##### View running Java processes

    $JAVA_HOME/bin/jps -mlvV

## Thread Dumps


##### Dump Thread Stacks

    $JAVA_HOME/bin/jstack $pid
    

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
    
Show only live objects (*NB: this triggers a full GC*):

    $JAVA_HOME/bin/jmap -histo:live $pid

##### Full heap dump

Generate a full heap dump (*NB: this will freeze the JVM for the full duration*).

    $JAVA_HOME/bin/jmap -dump:format=b,file=/tmp/heap_dump_$pid.bin $pid



## CPU Usage

##### View CPU usage by thread

Java threads are mapped to native threads on Linux. Use `top -H` to show threads in top.

    top -p $pid -H 
    
The PIDs of the threads can be converted to hexadecimal and matched to the nid=<id> entries in a thread dump. The following command can be used to convert PIDs to hexadecimal on the fly.

    top -p $pid -H -d 3 -b | sed -r 's/^([0-9]+) (.*)/printf "%x  \2" \1/e'
    
Top is used in batch mode which makes it append the output to the console.
