# How the mem$d used the libevent? #

---

## Introduction ##

  * he tree was showed that how libevent was used in mem$d, also it is a scenario that how to use libevent in multi-thread, in mem$d, it uses main thread and worker threads mode, each thread has its own event base, doing its event loop to check the cared event.
  * ach worker thread has a CQ, main thread will push task into CQ and send a notification to it, so the worker thread will get this notification, then start to process the task in CQ.
  * he main thread use the round robin to select the work thread to do the task.


## Details ##

```
/* initialize main thread libevent instance */
main_base = event_init();

/* start up worker threads if MT mode */
thread_init(settings.num_threads, main_base); 

    /* the type of threads[i] is LIBEVENT_THREAD */
    -- setup_thread(&threads[i]); 

        /* initial current thread own's event base */
        -- me->base = event_init(); 

        /* setup the notification event, the main thread will send a 
         * message to current thread, tell it the task queue has task, 
         * note that the event is persist event (don't remove from 
         * register queue and fdset)
         */
        -- event_set(&me->notify_event, me->notify_receive_fd, 
           EV_READ | EV_PERSIST, thread_libevent_process, me);

        /* associate this event to event base */
        -- event_base_set(me->base, &me->notify_event); 

        /* register the event */
        -- event_add(&me->notify_event, 0) 
        
    /* create the worker pthread */
    -- create_worker(worker_libevent, &threads[i]); 
        -- pthread_create(&thread, &attr, func, arg)
            /* current worker pthread's event loop */
            -- event_base_loop(me->base, 0); 

server_socket(settings.port, tcp_transport, portnumber_file)

    -- socket/bind/listen

    /* prepare the data structure struct conn 
     *which representing a connection into memcached
     */
    -- isten_conn_add = conn_new(sfd, conn_listening, 
       EV_READ | EV_PERSIST, 1, transport, main_base)))
 
        /* initial event structure and set the event 
         * base, event_flags is EV_READ | EV_PERSIST 
         */
        -- event_set(&c->event, sfd, event_flags, event_handler, (void *)c); 
            -- drive_machine(c);

                /* when state is conn_listening, ony called from the main thread */
                -- dispatch_conn_new(sfd, conn_new_cmd, EV_READ | EV_PERSIST,
                   DATA_BUFFER_SIZE, tcp_transport); 

                    /* round robin select the worker thread, then 
                     * push the item into its CQ 
                     */
                    -- cq_push(thread->new_conn_queue, item); 

                    /* write one bit to notify the worker thread 
                     * which is doing event loop 
                     */
                    -- write(thread->notify_send_fd, "", 1); 
        -- event_base_set(base, &c->event);

        /* register the listening fd in main base */
        -- event_add(&c->event, 0);
        
/* enter the main event loop */
event_base_loop(main_base, 0); 
```