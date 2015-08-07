# Catch the different places between select.c and poll.c #

---

## Introduction ##

Understand the design of the "struct selectop", "struct pollop" and "struct epollop", select and op need manage the fds in userspace, so we need alloc memory for them, but epoll manage the fds in kernel, but we need handle the active fds.

  * elect have 4 fd\_sets, 'event\_xxset\_in' were used to register the interested fds, 'event\_xxset\_out' were used by select, because after select return, it will eliminate the unactive fds, in order to avoid this, we need copy the fdset 'in' to fdset 'out' before calling select each time.

  * oll can manage many fds at the same times, so we can't malloc a large array which array size is largest fd (not reason for memory, reason for loop times?), so the 'struct pollfd' in event\_set are compact, nfds means current number of pollfd. In order to find the pollfd/event quickly, it uses "idxplus1\_by\_fd" to do the map, this design can reduce the loop times when deal with fds.

  * poll only need manage events which will fill by the kernel, all of epoll\_event in events are active, so we can check them one by one easily. Also i will store the event in userspace mapping use fd.

## Details ##

```
struct selectop {
    int event_fds;        // highest fd in fdset	

    // event_fdsz * 8 means the array size of event_r_by_fd/event_w_by_fd	
    int event_fdsz;

    // the user can register the r/w at here
    fd_set *event_readset_in; 
    fd_set *event_writeset_in;

    /* select will use these copy fds from in, because 
     * select will eliminate the unready fd
     */
    fd_set *event_readset_out; 
    fd_set *event_writeset_out;

    // store the struct event, one fd, one struct event
    struct event **event_r_by_fd; 
    struct event **event_w_by_fd;
};
```

```
struct pollop {
    int event_count;                 // highest number alloc	
    int nfds;                        // size of event_*    
    int fd_count;                    // Size of idxplus1_by_fd 
    struct pollfd *event_set;

    struct event **event_r_back;
    struct event **event_w_back;

    /* Index into event_set by fd; we add 1 so 
     * that 0 (which is easy to memset) can mean "no entry."
     * This array is used to map fd to "struct pollfd"
     */
    int *idxplus1_by_fd; 
};
```

```
struct evepoll {
	struct event *evread;
	struct event *evwrite;
};

struct epollop {
	struct evepoll *fds;
	int nfds;
	struct epoll_event *events;
	int nevents;
	int epfd;
};
```