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

### How tail calls work
- The kernel program must have different XDP programs in different sections using `SEC("xdp1")`. 
- The user program has to make a call to the function `load_bpf_file`.
- When this call is made, an array `map_fd` is filled with file descriptors of maps in the order it is defined in the kernel program. An array `prog_fd` is also filled with file descriptors of programs in the order it is defined in the kernel program.
- File descriptors from the `prog_fd` array can be appropriately put into maps using their descriptors from the `map_fd` array
- For example - in this application the file descriptor of the second program (index 1) is put into key 78 (anything) into the first map defined (index 0). 
- Tail call using `bpf_tail_call` is made to the key used in the map (78 in the example)

### Kernel program
The kernel XDP code is available [here](https://github.com/PriyankaSelvan/xdp-tailcall/blob/master/kern/tail_kern.c) and the Makefile to compile it is avilable [here](https://github.com/PriyankaSelvan/xdp-tailcall/blob/master/kern/Makefile). 

The Makefile just makes sure required header files are accessible. The Makefile compiles the kernel code write using clang. This is due to the fact that only clang provides an option of specifying a bpf target required for XDP.

The kernel code contains
- Map definition of jump table
- 2 XDP programs
- Tail call from one program to another

#### Map definition
```
struct bpf_map_def SEC("maps") jmp_table1 = {
	.type = BPF_MAP_TYPE_PROG_ARRAY,
	.key_size = sizeof(u32),
	.value_size = sizeof(u32),
	.max_entries = 100,
};
```
This is the definition of the map that is used as the jump table for the tail call. The user program is expected to insert a value of the file descriptor of the second program at some key (can be anything - same key must be used in kernel and user). The map must be of type `BPF_MAP_TYPE_PROG_ARRAY` to enable tail calls. 

#### Tail call
```
u32 key = 78;
bpf_tail_call(ctx, &jmp_table1, key);
```
Making a tail call using key 78 in map `jmp_table1`. Key 78 at the same map is populated with the file descriptor of the second section in the user program.

#### Debug
In order to print debugs from the kernel XDP program, _printk_ must be used. A standard way of using this in most XDP sample programs is defining a macro _bpf_debug_ as follows. 
```
#define bpf_debug(fmt, ...)                     \
        ({                          \
            char ____fmt[] = fmt;               \
            bpf_trace_printk(____fmt, sizeof(____fmt),  \
                     ##__VA_ARGS__);            \
        })
```

This can be used like a formatted output statement anywhere in the kernel XDP program.
```
bpf_debug("XDP: Killroy was here! %d\n", 42);
```

In order to view these debugs as traffic arrives at the interface, the following steps need to be taken. 
First, set the debug level to show all debugs. _7_ means all levels. 
```
sudo sh -c 'echo 7 > /proc/sys/kernel/printk'
```
Viewing the debugs. Needs to be run as root. Run `sudo su` before the following. 
```
tc exec bpf dbg
```

### User program
The user program contains the following
- Call to `load_bpf_file`
- Adding file descriptor to map
- Loading XDP kernel program to interface

#### Call to `load_bpf_file`
```
if(bpf_code = load_bpf_file(filename))
	{
		printf("\n error in bpf load");
		return 1;
	}
```
When this call is made, an array `map_fd` is filled with file descriptors of maps in the order it is defined in the kernel program(file descriptor of `jmp_table1` at index 0 and file descriptor `jmp_table2` at index 1). An array `prog_fd` is also filled with file descriptors of programs in the order it is defined in the kernel program(file descriptor of `xdp/0` in index 0 and file descriptor of `xdp/1` in index 1).

Make sure that the filename of the kernel object file is correct. The example code has it hardcoded. 

#### Adding tail call file descriptor to map
```
void jmp_table_add_prog(int map_fd_id, int key, int prog_fd_id)
{
	int jmp_fd = map_fd[map_fd_id];
	int tail_fd = prog_fd[prog_fd_id];

	if(tail_fd == 0)
	{
		printf("\n invalid program - not loaded ?");
		return;
	}

	int err;

	err = bpf_map_update_elem(jmp_fd, &key, &tail_fd, 0);
	if(err)
	{
		printf("\n could not update element");
		return;
	}

	struct bpf_prog_info info = {};
	uint32_t info_len = sizeof(info);
	int value;

	err = bpf_obj_get_info_by_fd(tail_fd, &info, &info_len);
	assert(!err);
	err = bpf_map_lookup_elem(jmp_fd, &key, &value);
	assert(!err);
	assert(value == info.id);

	return;
}
```
This function takes as inputs the `map_fd` index of the jump table, the key and the `prog_fd` index of the tail call. The asserts at the end of the function tests the valid updation of element in the map. Here, the file descriptor of the second program (index 1) is put into the file descriptor of the first map (index 0) at key 78. 

#### Loading XDP program to interface
```
int ifindex;
	ifindex = if_nametoindex(iface);
	if(ifindex == 0)
	{
		printf("\n interface name unknown");
		return 1;
	}
  
 if(bpf_set_link_xdp_fd(ifindex, prog_fd[0], 0) < 0)
	{
		printf("\n link set failed");
		return 1;
	}
```
These pieces of code load the kernel program to the interface. Make sure that the interface name is correct. The example code has this hardcoded. 

```
sleep(50);
```
The sleep exists in the program because the tail call will not be successful if the user program finishes execution. 

### Program procedure
The way to run this setup is as follows
- Compile kernel program
- Compile user program
- `sudo su` to be in root
- Run user program
- Send traffic to interface from another node
- `tc exec bpf dbg` to view kernel debugs that shows tail call working (Look above to enable kernel debugs)

This concludes the procedure to perform tail calls in XDP. Earlier, functions did not exist in XDP hence the provision for tail calls. Currently, XDP programs do allow functions. Nevertheless, tail calls can still be used because it still is much faster than a function call.

Other articles related to XDP are as follows
### [XDP Setup](https://priyankaselvan.github.io/eXpress-Data-Path--Setup/)
### [Using XDP Maps](https://priyankaselvan.github.io/eXpress-Data-Path--Maps)
### [Modifying packets using XDP](https://priyankaselvan.github.io/eXpress-Data-Path--Modifying-Packets/)
