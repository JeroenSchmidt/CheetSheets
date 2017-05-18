# Hadoop Setup 

[TOC]

## Binary File for windows

### HIVE error - writing rights

You can run everything as Admin then this error wont appear OR you can fix it as follows:

[stack over flow](http://stackoverflow.com/questions/34196302/the-root-scratch-dir-tmp-hive-on-hdfs-should-be-writable-current-permissions)

```shell
D:\winutils\bin\winutils.exe chmod 777 D:\tmp\hive
```

**TLDR** - you need to change the user rights within hadoop for the temp hive file in the windows directory

### Doesn't recognise "''C:\Program

Maven & Hadoop can't interpret spaces, this means that the `ENVIROMENT VARIABLES` that have path directories with spaces in them will cause errors 

**Solution**

- Use Windows shortened name (say 'PROGRA~1' for 'Program Files') or
- My preferred method - create a folder called C:\Java which your JAVA_HOME can target. Put a [symlink](https://technet.microsoft.com/en-us/library/Cc753194.aspx) in C:\Java to point to your actual Java installation. This also makes things easier if you have multiple versions of Java on your machine.

[Further reading](https://github.com/karthikj1/Hadoop-2.7.1-Windows-64-binaries)

 