## XDP
Express Data Path is a programmable fast packet processor in the kernel. Details about XDP can be found [here](https://dl.acm.org/citation.cfm?id=3281443), and [here](https://developers.redhat.com/blog/2018/12/06/achieving-high-performance-low-latency-networking-with-xdp-part-1/). This article contains the steps to setup a development environment for XDP.

## Required for this article
### [XDP Setup](https://priyankaselvan.github.io/eXpress-Data-Path--Setup/)

## Other Articles
### [Using XDP Maps](https://priyankaselvan.github.io/eXpress-Data-Path--Maps)
### [Modifying packets using XDP](https://priyankaselvan.github.io/eXpress-Data-Path--Modifying-Packets/)

## XDP Tail Calls
Tail calls are a mechanism that allows one XDP program to call another, without returning back to the old program. Such a call has minimal overhead as unlike function calls, it is implemented as a long jump, reusing the same stack frame.

## XDP application
A simple XDP application that contains a kernel and a user program has been written to illustrate the use of XDP tail calls. The repository can be found [here](https://github.com/PriyankaSelvan/xdp-tailcall). The rest of this article, explains the different parts of code required to perform an XDP tail call.

