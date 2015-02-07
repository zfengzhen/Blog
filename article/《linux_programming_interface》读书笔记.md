# 《Linux Programming Interface》读书笔记  
**作者: fergus (zfengzhen@gmail.com)**    

## 目录 

- [4 file I/O:the universal I/O model](https://github.com/zfengzhen/Blog/blob/master/%E3%80%8Alinux_programming_interface%E3%80%8B%E8%AF%BB%E4%B9%A6%E7%AC%94%E8%AE%B0.md#4-file-iothe-universal-io-model)
- [5 file I/O:  further details](https://github.com/zfengzhen/Blog/blob/master/%E3%80%8Alinux_programming_interface%E3%80%8B%E8%AF%BB%E4%B9%A6%E7%AC%94%E8%AE%B0.md#5-file-io--further-details)
- [6 processes](https://github.com/zfengzhen/Blog/blob/master/%E3%80%8Alinux_programming_interface%E3%80%8B%E8%AF%BB%E4%B9%A6%E7%AC%94%E8%AE%B0.md#6-processes)
- [10 time](https://github.com/zfengzhen/Blog/blob/master/%E3%80%8Alinux_programming_interface%E3%80%8B%E8%AF%BB%E4%B9%A6%E7%AC%94%E8%AE%B0.md#10-time)
- [12 system and process information](https://github.com/zfengzhen/Blog/blob/master/%E3%80%8Alinux_programming_interface%E3%80%8B%E8%AF%BB%E4%B9%A6%E7%AC%94%E8%AE%B0.md#12-system-and-process-information)
- [13 file I/O buffering](https://github.com/zfengzhen/Blog/blob/master/%E3%80%8Alinux_programming_interface%E3%80%8B%E8%AF%BB%E4%B9%A6%E7%AC%94%E8%AE%B0.md#13-file-io-buffering)
- [14 file systems](https://github.com/zfengzhen/Blog/blob/master/%E3%80%8Alinux_programming_interface%E3%80%8B%E8%AF%BB%E4%B9%A6%E7%AC%94%E8%AE%B0.md#14-file-systems)
- [19 monitoring file events](https://github.com/zfengzhen/Blog/blob/master/%E3%80%8Alinux_programming_interface%E3%80%8B%E8%AF%BB%E4%B9%A6%E7%AC%94%E8%AE%B0.md#19-monitoring-file-events)
- [23 process creation](https://github.com/zfengzhen/Blog/blob/master/%E3%80%8Alinux_programming_interface%E3%80%8B%E8%AF%BB%E4%B9%A6%E7%AC%94%E8%AE%B0.md#23-process-creation)
- [24 process termination](https://github.com/zfengzhen/Blog/blob/master/%E3%80%8Alinux_programming_interface%E3%80%8B%E8%AF%BB%E4%B9%A6%E7%AC%94%E8%AE%B0.md#24-process-termination)
- [25 monitoring child process](https://github.com/zfengzhen/Blog/blob/master/%E3%80%8Alinux_programming_interface%E3%80%8B%E8%AF%BB%E4%B9%A6%E7%AC%94%E8%AE%B0.md#25-monitoring-child-process)
- [34 Process Groups, Sessions, and Job Control](https://github.com/zfengzhen/Blog/blob/master/%E3%80%8Alinux_programming_interface%E3%80%8B%E8%AF%BB%E4%B9%A6%E7%AC%94%E8%AE%B0.md#34-process-groups-sessions-and-job-control)
- [41 fundamentals of shared libraries](https://github.com/zfengzhen/Blog/blob/master/%E3%80%8Alinux_programming_interface%E3%80%8B%E8%AF%BB%E4%B9%A6%E7%AC%94%E8%AE%B0.md#41-fundamentals-of-shared-libraries)
- [51 Introduction to POSIX IPC](https://github.com/zfengzhen/Blog/blob/master/%E3%80%8Alinux_programming_interface%E3%80%8B%E8%AF%BB%E4%B9%A6%E7%AC%94%E8%AE%B0.md#51-introduction-to-posix-ipc)
- [52 POSIX Message Queues](https://github.com/zfengzhen/Blog/blob/master/%E3%80%8Alinux_programming_interface%E3%80%8B%E8%AF%BB%E4%B9%A6%E7%AC%94%E8%AE%B0.md#52-posix-message-queues)
- [53 POSIX Semaphores](https://github.com/zfengzhen/Blog/blob/master/%E3%80%8Alinux_programming_interface%E3%80%8B%E8%AF%BB%E4%B9%A6%E7%AC%94%E8%AE%B0.md#53-posix-semaphores)
- [54 POSIX Shared Memory](https://github.com/zfengzhen/Blog/blob/master/%E3%80%8Alinux_programming_interface%E3%80%8B%E8%AF%BB%E4%B9%A6%E7%AC%94%E8%AE%B0.md#54-posix-shared-memory)
- [55 File Locking](https://github.com/zfengzhen/Blog/blob/master/%E3%80%8Alinux_programming_interface%E3%80%8B%E8%AF%BB%E4%B9%A6%E7%AC%94%E8%AE%B0.md#55-file-locking)
- [56 Sockets: Introduction](https://github.com/zfengzhen/Blog/blob/master/%E3%80%8Alinux_programming_interface%E3%80%8B%E8%AF%BB%E4%B9%A6%E7%AC%94%E8%AE%B0.md#56-sockets-introduction)
- [59 Sockets: Internet Domains](https://github.com/zfengzhen/Blog/blob/master/%E3%80%8Alinux_programming_interface%E3%80%8B%E8%AF%BB%E4%B9%A6%E7%AC%94%E8%AE%B0.md#59-sockets-internet-domains)
- [63 Alternative I/O Models](https://github.com/zfengzhen/Blog/blob/master/%E3%80%8Alinux_programming_interface%E3%80%8B%E8%AF%BB%E4%B9%A6%E7%AC%94%E8%AE%B0.md#63-alternative-io-models)

## 4 file I/O:the universal I/O model
![](https://github.com/zfengzhen/Blog/blob/master/img/lpi_4_1.png)  
![](https://github.com/zfengzhen/Blog/blob/master/img/lpi_4_2.png)  
![](https://github.com/zfengzhen/Blog/blob/master/img/lpi_4_3.png)  

## 5 file I/O:  further details
![](https://github.com/zfengzhen/Blog/blob/master/img/lpi_5_1.png)  
![](https://github.com/zfengzhen/Blog/blob/master/img/lpi_5_2.png)  

## 6 processes
![](https://github.com/zfengzhen/Blog/blob/master/img/lpi_6_1.png)  
![](https://github.com/zfengzhen/Blog/blob/master/img/lpi_6_2.png)  
![](https://github.com/zfengzhen/Blog/blob/master/img/lpi_6_3.png)  

## 10 time
![](https://github.com/zfengzhen/Blog/blob/master/img/lpi_10_1.png)  
![](https://github.com/zfengzhen/Blog/blob/master/img/lpi_10_2.png)   

## 12 system and process information
![](https://github.com/zfengzhen/Blog/blob/master/img/lpi_12_1.png)  

## 13 file I/O buffering
![](https://github.com/zfengzhen/Blog/blob/master/img/lpi_13_1.png)  
![](https://github.com/zfengzhen/Blog/blob/master/img/lpi_13_2.png)  
![](https://github.com/zfengzhen/Blog/blob/master/img/lpi_13_3.png)  

## 14 file systems
![](https://github.com/zfengzhen/Blog/blob/master/img/lpi_13_1.png)  
![](https://github.com/zfengzhen/Blog/blob/master/img/lpi_14_2.png)  
![](https://github.com/zfengzhen/Blog/blob/master/img/lpi_14_3.png)  
![](https://github.com/zfengzhen/Blog/blob/master/img/lpi_14_4.png)  
![](https://github.com/zfengzhen/Blog/blob/master/img/lpi_14_5.png)  
![](https://github.com/zfengzhen/Blog/blob/master/img/lpi_14_6.png)

## 19 monitoring file events
![](https://github.com/zfengzhen/Blog/blob/master/img/lpi_19_1.png)    
![](https://github.com/zfengzhen/Blog/blob/master/img/lpi_19_2.png)    

## 23 process creation
![](https://github.com/zfengzhen/Blog/blob/master/img/lpi_23_1.png)    

## 24 process termination
![](https://github.com/zfengzhen/Blog/blob/master/img/lpi_24_1.png)    

## 25 monitoring child process
![](https://github.com/zfengzhen/Blog/blob/master/img/lpi_25_1.png)    
![](https://github.com/zfengzhen/Blog/blob/master/img/lpi_25_2.png)   

## 34 Process Groups, Sessions, and Job Control
![](https://github.com/zfengzhen/Blog/blob/master/img/lpi_34_1.png)    
![](https://github.com/zfengzhen/Blog/blob/master/img/lpi_34_2.png)   

## 41 fundamentals of shared libraries
![](https://github.com/zfengzhen/Blog/blob/master/img/lpi_41_1.png)    
![](https://github.com/zfengzhen/Blog/blob/master/img/lpi_41_2.png)    
![](https://github.com/zfengzhen/Blog/blob/master/img/lpi_41_3.png)    

## 51 Introduction to POSIX IPC
![](https://github.com/zfengzhen/Blog/blob/master/img/lpi_51_1.png)   

## 52 POSIX Message Queues
![](https://github.com/zfengzhen/Blog/blob/master/img/lpi_52_1.png)   

## 53 POSIX Semaphores
![](https://github.com/zfengzhen/Blog/blob/master/img/lpi_53_1.png)  

## 54 POSIX Shared Memory
![](https://github.com/zfengzhen/Blog/blob/master/img/lpi_54_1.png)  

## 55 File Locking
![](https://github.com/zfengzhen/Blog/blob/master/img/lpi_55_1.png)  

## 56 Sockets: Introduction
![](https://github.com/zfengzhen/Blog/blob/master/img/lpi_56_1.png)  

## 59 Sockets: Internet Domains
![](https://github.com/zfengzhen/Blog/blob/master/img/lpi_59_1.png)   

## 63 Alternative I/O Models
![](https://github.com/zfengzhen/Blog/blob/master/img/lpi_63_1.png)   

