
# The ultimate golden release upgrade guide

## Community effort to write an upgrade guide for dCache 7.2

How to get from dCache 6.2 to dCache 7.2

## Executive summary

### The highlights in 7.2 compared with 6.2

-  quota system 
-  support for file labels
-  a dedicated scheduling strategy for bring-online requests in SRM-Manager
-  extended attribute support with http and nfs


## Breaking changes

### Configuration properties
For File system statistics added functionality to cache total files and total space used on DB backend. 
The following two properties has been added:


| property  | new value |
|:----------|-------:|
pnfsmanager.fs-stat-cache.time | 3600
pnfsmanager.fs-stat-cache.time.unit | SECONDS

### New output format in NFS admin interface

The admin interface for NFS doors has been updated to remove redundant information from the output of the `show clients` command:

```
    [dcache-lab000] (NFS-dcache-lab007@core-dcache-lab007) admin > show clients
        /11.19.15.23:967:Linux NFSv4.1 ani:v4.1
            5f4ccad3000300010000000000000001 max slot: 15/0
    
    [dcache-lab000] (NFS-dcache-lab007@core-dcache-lab007) admin >
```

The argument of `kill client` accepts the client's session id. For instance, to kill the client from example above:

```
[dcache-lab000] (NFS-dcache-lab007@core-dcache-lab007) admin > kill client 5f4ccad3000300010000000000000001
```
## Runtime environment

### Java flight recorder

When debugging an issue on a running system often we need to collect jvm performance stats with ‘Java flight recorder’. Starting from release 7.2 the Java flight recorder attach listener is enabled by default. Site admins can collect and provide developers with additional information when high CPU load or memory consuption is observed as:

jcmd <pid> JFR.start duration=60s filename=/tmp/dcache.jfr,

    Please note, that jcmd command is a part of java-11-openjdk-devel package (on RHEL and clones)

### Handling of OutOfMemoryError

Depending which thread have received OutOfMemoryError the JVM might or might not exit. In a later cache, dCache might remain in an unpredictable state, where a component might be exposed as functional when its not.

With 7.2 we have update the java options to include ExitOnOutOfMemoryError, which forces JVM to exit when an OOM is detected.

    NOTE: There are several situations when jvm generates an OutOfMemoryError. The ExitOnOutOfMemoryError option works ONLY when allocation in heap space fails.


## New services
