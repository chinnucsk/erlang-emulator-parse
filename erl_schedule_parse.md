erl_schedule_parse
=================

__Authors:__[`lucas`](mailto:564985699@qq.com).

__HomePage:__[`Erlang Emulator Parse`](http://lucas564985699.github.io/erlang-emulator-parse).

__Data:__2013-09-10.

Introduce
---------------

This articl helps you to know erlang parameters better, your Erlang runtime system runs more effciently in production environment.

Some basic flags and emulator flags will not introduce here for example -boot, -config, -cookie, I will introduce some flags that can affect runtime system, we can get a better and stronger runtime system accorrding to config them.

Text
---------------

* What is SMP?
	+ introduce

	SMP (Symmetrical Multi Processor) ,SMP has multiple processors, if you don't open SMP, only one processor works, so the efficient is slow. Erlang runtime system open SMP default, we can open or close SMP by flag `-smp [enable|auto|disable]`. <br />

	+ instrustion
```javascript
		   -smp enable and -smp starts the Erlang runtime system with SMP sup‐
           port  enabled.  This may fail if no runtime system with SMP support
           is available. -smp auto starts the Erlang runtime system  with  SMP
           support  enabled  if it is available and more than one logical pro‐
           cessor are detected. -smp disable starts a runtime  system  without
           SMP support. By default -smp auto will be used unless a conflicting
           parameter has been passed, then -smp disable  will  be  used.  Cur‐
           rently only the -hybrid parameter conflicts with -smp auto.

           NOTE:  The runtime system with SMP support will not be available on
           all supported platforms. See also the +S flag.
```
	+ config
		config:<br />
			- compile: `./configure --enable-smp-support`
			- close smp emulator: `./configure --disable-smp-support`
			- erl start: `-smp enable -smp disable`
* What is Scheduler?
	+ introduce
	In erlang runtime syterm, to ensure the effiect, when SMP is enabled, multiple processes will handle the work, so Scheduler comes out, it used to dispatch the tasks to make sure the tasks can be handled by cpu balance.
	+ scheduler struct <br />

```javascript
struct ErtsSchedulerData_ {
    /*
     * Keep X registers first (so we get as many low
     * numbered registers as possible in the same cache
     * line).
     */
    Eterm* x_reg_array;     /* X registers */
    FloatDef* f_reg_array;  /* Floating point registers. */

#ifdef ERTS_SMP
    ethr_tid tid;       /* Thread id */
    struct erl_bits_state erl_bits_state; /* erl_bits.c state */
    void *match_pseudo_process; /* erl_db_util.c:db_prog_match() */
    Process *free_process;
    ErtsThrPrgrData thr_progress_data;
#endif
#if !HEAP_ON_C_STACK
    Eterm tmp_heap[TMP_HEAP_SIZE];
    int num_tmp_heap_used;
    Eterm beam_emu_tmp_heap[BEAM_EMU_TMP_HEAP_SIZE];
    Eterm cmp_tmp_heap[CMP_TMP_HEAP_SIZE];
    Eterm erl_arith_tmp_heap[ERL_ARITH_TMP_HEAP_SIZE];
#endif
    ErtsSchedulerSleepInfo *ssi;
    Process *current_process;
    Uint no;            /* Scheduler number */
    struct port *current_port;
    ErtsRunQueue *run_queue;
    int virtual_reds;
    int cpu_id;         /* >= 0 when bound */
    ErtsAuxWorkData aux_work_data;
    ErtsAtomCacheMap atom_cache_map;

    ErtsSchedAllocData alloc_data;

    Uint64 reductions;
    ErtsSchedWallTime sched_wall_time;

#ifdef ERTS_DO_VERIFY_UNUSED_TEMP_ALLOC
    erts_alloc_verify_func_t verify_unused_temp_alloc;
    Allctr_t *verify_unused_temp_alloc_data;
#endif
};
```	

	The struct includes many parameters, They are used to store some values  used in procedure, for example current_process is a point to store the erlang   runtime main process pointer.  

* What is Run queue?
	+ introduce <br>
	Run queue is a queue for stroring tasks unhandled in erlang runtime system,
when before R13, only one run queue ,so schedulers will share the same run queue, so we need the lock to avoid errors, obviousely the effient is low for the many schedulers rob the same run queue. But after that, every scheduler will own one run queue, so they don't need to rob resources, it will improves the efficient,of course the precondition is SMP is enable.
	+ struct ErtsRunQueue_
```javascript
struct ErtsRunQueue_ {
    int ix;
    erts_smp_atomic32_t info_flags;

    erts_smp_mtx_t mtx;
    erts_smp_cnd_t cnd;

    ErtsSchedulerData *scheduler;
    int waiting; /* < 0 in sys schedule; > 0 on cnd variable */
    int woken;
    Uint32 flags;
    int check_balance_reds;
    int full_reds_history_sum;
    int full_reds_history[ERTS_FULL_REDS_HISTORY_SIZE];
    int out_of_work_count;
    int max_len;
    int len;
    int wakeup_other;
    int wakeup_other_reds;
    int halt_in_progress;

    struct {
    int len;
    ErtsProcList *pending_exiters;
    Uint context_switches;
    Uint reductions;

    ErtsRunQueueInfo prio_info[ERTS_NO_PROC_PRIO_LEVELS];

    /* We use the same prio queue for low and
       normal prio processes */
    ErtsRunPrioQueue prio[ERTS_NO_PROC_PRIO_LEVELS-1];
    } procs;

    struct {
    ErtsMiscOpList *start;
    ErtsMiscOpList *end;
    ErtsRunQueue *evac_runq;
    } misc;

    struct {
    ErtsRunQueueInfo info;
    struct port *start;
    struct port *end;
    } ports;
};
```
	The struct stores the information about balance flag(To show whether run queque is balance), wakeup reds(Scheduler can be sleep, so wakeup reds to show wheter to be waked up), Procs(The process  to run work), ports(which to port to run work) and so on.	

```javascript
typedef struct {
    Process* first;
    Process* last;
} ErtsRunPrioQueue;
	
	The struct stores two pointers to point the last and next queue task.

```javascript
typedef struct {
    int len;
    int max_len;
    int reds;
    struct {
    struct {
        int this;
        int other;
    } limit;
    ErtsRunQueue *runq;
    } migrate;
} ErtsRunQueueInfo;
```
	The struct stores the information of current run queue.

* Scheduler Strategy
	+ introduce
	When erlang runtime system runs with SMP enable, will produce the schdulers(number is default current system cpu cores) when you not set erl paras `+s`, every scheduler will have a corresponding run queue for consuming tasks, when one scheduler has many tasks and another has few, so scheduler can balance it and migrate somes works to the scheduler with few tasks.
	Also some Schedulers are working and some Schedulers are sleeping, and when some sepical conditions it can be waked up ,you can `+swt` to tell emulator when to wake them. 
	You can tell the emulator how many schedulers will start and how many schedulers will work by paras `erl +s N:M`, the paras i will introduce it below in detail.
	Erlang system has 8 reader groups, When the amount of schedulers is less than or equal to  the  reader groups  limit,  each  scheduler  has its own reader group. If some schedulers shared one reader group will degrade the read operation s and performance.	

* erl start parameters about scheduler and cpu
	+ `+rq ReaderGroupsLimit`
		- introduce
```javascript
		   Limits the amount of reader groups used by read/write  locks  opti‐
           mized  for read operations in the Erlang runtime system. By default
           the reader groups limit equals 8.

           When the amount of schedulers is less than or equal to  the  reader
           groups  limit,  each  scheduler  has its own reader group. When the
           amount of schedulers is larger than the reader groups limit, sched‐
           ulers  share reader groups. Shared reader groups degrades read lock
           and read unlock performance while a large amount of  reader  groups
           degrades write lock performance, so the limit is a tradeoff between
           performance for read operations and performance  for  write  opera‐
           tions.  Each  reader  group  currently  consumes  64  byte  in each
           read/write lock. Also note  that  a  runtime  system  using  shared
           reader  groups  benefits from binding schedulers to logical proces‐
           sors, since the reader groups are distributed better between sched‐
           ulers.
```
		- instrustions
			This parmeters will affect the read perfomance ,but normally we can ignore it for the erlang runtime system start 8 reader groups.
	+ `+S Schedulers:SchedulerOnline`
		- introduce
```javascript
           Sets  the  amount  of  scheduler  threads  to  create and scheduler
           threads to set online when SMP  support  has  been  enabled.  Valid
           range  for  both values are 1-1024. If the Erlang runtime system is
           able to determine the amount of logical processors  configured  and
           logical  processors  available,  Schedulers will default to logical
           processors configured, and SchedulersOnline will default to logical
           processors  available;  otherwise,  the  default  values will be 1.
           Schedulers may be omitted  if  :SchedulerOnline  is  not  and  vice
           versa.  The  amount of schedulers online can be changed at run time
           via erlang:system_flag(schedulers_online, SchedulersOnline).

           This flag will be ignored if the emulator doesn't have SMP  support
           enabled (see the -smp flag).

         +sFlag Value:
           Scheduling specific flags.

           +sbt BindType:
             Set scheduler bind type. Currently valid BindTypes:

             u:
               unbound  -  Schedulers will not be bound to logical processors,
               i.e., the operating system decides where the scheduler  threads
               execute, and when to migrate them. This is the default.

             ns:
               no_spread - Schedulers with close scheduler identifiers will be
               bound as close as possible in hardware.

             ts:
               thread_spread - Thread refers to hardware threads (e.g. Intel's
               hyper-threads). Schedulers with low scheduler identifiers, will
               be bound to the first hardware thread of each core, then sched‐
               ulers  with  higher  scheduler identifiers will be bound to the
               second hardware thread of each core, etc.

             ps:
               processor_spread   -   Schedulers   will   be    spread    like
               thread_spread, but also over physical processor chips.

             s:
               spread - Schedulers will be spread as much as possible.

             nnts:
               no_node_thread_spread  -  Like  thread_spread,  but if multiple
               NUMA (Non-Uniform Memory Access) nodes exists, schedulers  will
               be  spread over one NUMA node at a time, i.e., all logical pro‐
               cessors of one  NUMA  node  will  be  bound  to  schedulers  in
               sequence.

			nnps:
               no_node_processor_spread - Like processor_spread, but if multi‐
               ple NUMA nodes exists, schedulers will be spread over one  NUMA
               node  at  a time, i.e., all logical processors of one NUMA node
               will be bound to schedulers in sequence.

             tnnps:
               thread_no_node_processor_spread    -    A    combination     of
               thread_spread, and no_node_processor_spread. Schedulers will be
               spread over hardware threads across NUMA nodes, but  schedulers
               will only be spread over processors internally in one NUMA node
               at a time.

             db:
               default_bind - Binds schedulers the default way. Currently  the
               default  is thread_no_node_processor_spread (which might change
               in the future).

             Binding of schedulers is currently only supported on newer Linux,
             Solaris, FreeBSD, and Windows systems.

             If  no  CPU topology is available when the +sbt flag is processed
             and BindType is any other type than u, the  runtime  system  will
             fail  to  start. CPU topology can be defined using the +sct flag.
             Note that the +sct flag may have to be  passed  before  the  +sbt
             flag  on the command line (in case no CPU topology has been auto‐
             matically detected).

             The runtime system will by default not bind schedulers to logical
             processors.

             NOTE:  If  the Erlang runtime system is the only operating system
             process that binds threads to logical processors,  this  improves
             the  performance of the runtime system. However, if other operat‐
             ing system processes (as for example another Erlang runtime  sys‐
             tem)  also  bind  threads to logical processors, there might be a
             performance penalty  instead.  In  some  cases  this  performance
             penalty  might be severe. If this is the case, you are advised to
             not bind the schedulers.

             How schedulers are bound matters. For example, in situations when
             there  are  fewer  running  processes than schedulers online, the
             runtime system tries to migrate processes to schedulers with  low
             scheduler  identifiers.  The  more the schedulers are spread over
             the hardware, the more resources will be available to the runtime
             system in such situations.

			 NOTE:  If  a scheduler fails to bind, this will often be silently
             ignored. This since it isn't always possible to verify valid log‐
             ical  processor  identifiers. If an error is reported, it will be
             reported to the error_logger. If you  want  to  verify  that  the
             schedulers  actually  have  bound  as requested, call erlang:sys‐
             tem_info(scheduler_bindings).
```
		- instrustion
			`+S` rule the number of schedulers total and working schedulers,	`+sbt` rule the bind type of scheduler, in production environment, we use the default value to start schedulers same with cpu cores, except we use it for other purpose ,for example testing codes in lower system config.
		- code
```javascript
    case 'S' : /* Was handled in early_init() just read past it */
        (void) get_arg(argv[i]+2, argv[i+1], &i);
        break;

```
	+ `scl true|false`
		- introduce
```javascript
			 Enable or disable scheduler compaction of load. By default sched‐
             uler  compaction of load is enabled. When enabled, load balancing
             will strive for a load distribution which causes as  many  sched‐
             uler threads as possible to be fully loaded (i.e., not run out of
             work). This is accomplished by migrating load (e.g. runnable pro‐
             cesses)  into  a  smaller  set of schedulers when schedulers fre‐
             quently run out of work. When disabled, the frequency with  which
             schedulers  run out of work will not be taken into account by the
             load balancing logic.
```
		- instruction
			`scl` rules whether erlang runtime system can balance tasks when one or more schedulers are busy and others are free, default is of course balance.

	+ `sct CpuTopology`
		- introduce
```javascript
             * <Id> = integer(); when 0 =< <Id> =< 65535

             * <IdRange> = <Id>-<Id>

             * <IdOrIdRange> = <Id> | <IdRange>

             * <IdList> = <IdOrIdRange>,<IdOrIdRange> | <IdOrIdRange>

             * <LogicalIds> = L<IdList>

             * <ThreadIds> = T<IdList> | t<IdList>

             * <CoreIds> = C<IdList> | c<IdList>

             * <ProcessorIds> = P<IdList> | p<IdList>

             * <NodeIds> = N<IdList> | n<IdList>

             * <IdDefs>       =       <LogicalIds><ThreadIds><CoreIds><Proces‐
               sorIds><NodeIds>         |         <LogicalIds><ThreadIds><Cor‐
               eIds><NodeIds><ProcessorIds>

             * CpuTopology = <IdDefs>:<IdDefs> | <IdDefs>
```
		- introduce
			A faked CPU topology that does not reflect how the real CPU  topology looks like is likely to decrease the performance of the runtime system.
			When we want to test or other purpose want to have special cpu core and special scheduler to test system perfomance, we can use it.
		- example
			`erl +sct L0-3c0-3 +sbt db +S3:2` start 4 cores has id for 0-3 with 3 schedulers ,2 of them works, schedule bind type is default bind.
	+ `swt very_low|low|medium|high|very_high`
		- introduce
```javascript
			Set scheduler wakeup threshold. Default is medium. The  threshold
             determines  when  to  wake  up sleeping schedulers when more work
             than can be handled by currently awake schedulers  exist.  A  low
             threshold  will  cause earlier wakeups, and a high threshold will
             cause later wakeups. Early wakeups will distribute work over mul‐
             tiple schedulers faster, but work will more easily bounce between
             schedulers.
```
		- instruction
			Simply say, let the system knows when to wake up the sleeping schedulers.
		- code
```javascript
	case 's' : {
        char *estr;
        int res;
        char *sub_param = argv[i]+2;
        if (has_prefix("bt", sub_param)) {
        arg = get_arg(sub_param+2, argv[i+1], &i);
        res = erts_init_scheduler_bind_type_string(arg);
        if (res != ERTS_INIT_SCHED_BIND_TYPE_SUCCESS) {
            switch (res) {
            case ERTS_INIT_SCHED_BIND_TYPE_NOT_SUPPORTED:
            estr = "not supported";
            break;
            case ERTS_INIT_SCHED_BIND_TYPE_ERROR_NO_CPU_TOPOLOGY:
            estr = "no cpu topology available";
            break;
            case ERTS_INIT_SCHED_BIND_TYPE_ERROR_NO_BAD_TYPE:
            estr = "invalid type";
            break;
            default:
            estr = "undefined error";
            break;
            }
            erts_fprintf(stderr,
                 "setting scheduler bind type '%s' failed: %s\n",
                 arg,
                 estr);
            erts_usage();
        }
        }
        else if (has_prefix("bwt", sub_param)) {
        arg = get_arg(sub_param+3, argv[i+1], &i);
        if (erts_sched_set_busy_wait_threshold(arg) != 0) {
            erts_fprintf(stderr, "bad scheduler busy wait threshold: %s\n",
                 arg);
            erts_usage();
        }
        VERBOSE(DEBUG_SYSTEM,
            ("scheduler wakup threshold: %s\n", arg));
        }
        else if (has_prefix("cl", sub_param)) {
        arg = get_arg(sub_param+2, argv[i+1], &i);
        if (sys_strcmp("true", arg) == 0)
            erts_sched_compact_load = 1;
        else if (sys_strcmp("false", arg) == 0)
            erts_sched_compact_load = 0;
        else {
				 erts_fprintf(stderr,
                 "bad scheduler compact load value '%s'\n",
                 arg);
            erts_usage();
        }
        }
        else if (has_prefix("ct", sub_param)) {
        arg = get_arg(sub_param+2, argv[i+1], &i);
        res = erts_init_cpu_topology_string(arg);
        if (res != ERTS_INIT_CPU_TOPOLOGY_OK) {
            switch (res) {
            case ERTS_INIT_CPU_TOPOLOGY_INVALID_ID:
            estr = "invalid identifier";
            break;
            case ERTS_INIT_CPU_TOPOLOGY_INVALID_ID_RANGE:
            estr = "invalid identifier range";
            break;
            case ERTS_INIT_CPU_TOPOLOGY_INVALID_HIERARCHY:
            estr = "invalid hierarchy";
            break;
            case ERTS_INIT_CPU_TOPOLOGY_INVALID_ID_TYPE:
            estr = "invalid identifier type";
            break;
            case ERTS_INIT_CPU_TOPOLOGY_INVALID_NODES:
            estr = "invalid nodes declaration";
            break;
            case ERTS_INIT_CPU_TOPOLOGY_MISSING_LID:
            estr = "missing logical identifier";
            break;
            case ERTS_INIT_CPU_TOPOLOGY_NOT_UNIQUE_LIDS:
            estr = "not unique logical identifiers";
            break;
            case ERTS_INIT_CPU_TOPOLOGY_NOT_UNIQUE_ENTITIES:
            estr = "not unique entities";
            break;
            case ERTS_INIT_CPU_TOPOLOGY_MISSING:
            estr = "missing cpu topology";
            break;
            default:
            estr = "undefined error";
            break;
            }
            erts_fprintf(stderr,
                 "bad cpu topology '%s': %s\n",
                 arg,
                 estr);
            erts_usage();
        }
        }
 	 else if (sys_strcmp("nsp", sub_param) == 0)
        erts_use_sender_punish = 0;
        else if (sys_strcmp("wt", sub_param) == 0) {
        arg = get_arg(sub_param+2, argv[i+1], &i);
        if (erts_sched_set_wakeup_other_thresold(arg) != 0) {
            erts_fprintf(stderr, "scheduler wakeup threshold: %s\n",
                 arg);
            erts_usage();
        }
        VERBOSE(DEBUG_SYSTEM,
            ("scheduler wakeup threshold: %s\n", arg));
        }
        else if (sys_strcmp("ws", sub_param) == 0) {
        arg = get_arg(sub_param+2, argv[i+1], &i);
        if (erts_sched_set_wakeup_other_type(arg) != 0) {
            erts_fprintf(stderr, "scheduler wakeup strategy: %s\n",
                 arg);
            erts_usage();
        }
        VERBOSE(DEBUG_SYSTEM,
            ("scheduler wakeup threshold: %s\n", arg));
        }
        else if (has_prefix("ss", sub_param)) {
        /* suggested stack size (Kilo Words) for scheduler threads */
        arg = get_arg(sub_param+2, argv[i+1], &i);
        erts_sched_thread_suggested_stack_size = atoi(arg);

        if ((erts_sched_thread_suggested_stack_size
             < ERTS_SCHED_THREAD_MIN_STACK_SIZE)
            || (erts_sched_thread_suggested_stack_size >
            ERTS_SCHED_THREAD_MAX_STACK_SIZE)) {
            erts_fprintf(stderr, "bad stack size for scheduler threads %s\n",
                 arg);
            erts_usage();
        }
        VERBOSE(DEBUG_SYSTEM,
            ("suggested scheduler thread stack size %d kilo words\n",
             erts_sched_thread_suggested_stack_size));
        }
        else {
        erts_fprintf(stderr, "bad scheduling option %s\n", argv[i]);
        erts_usage();
        }
        break;
    }

	The Codes show how erlang runtime system handles the args before starting scheduler thread.
```

* How Scheduler work?
	In file(`erts/emulator/beam/erl_init.c`), function erl_start is the entry to start erlang system, `erts_start_schedulers();` do first init the scheduler, do some work about creating scheduler thread and init internal parameters and the corresponding main erlang process parameters.
```javascript
void erl_start(int args, char **argv)
{
...
	erl_init(ncpu);

    load_preloaded();

    erts_initialized = 1;

    erl_first_process_otp("otp_ring0", NULL, 0, boot_argc, boot_argv);

#ifdef ERTS_SMP
    erts_start_schedulers();
    /* Let system specific code decide what to do with the main thread... */

    erts_sys_main_thread(); /* May or may not return! */
#else
    erts_thr_set_main_status(1, 1);

...
}
```
	Next, when all erlang runtime syetem init over, we will got to main method, `process_main();`, this function will check if scheduler thread if started or will start again, then will `goto do_schedule1`, run `schedule(c_p, reds_used);`, schedule return a process c_p from run queue, c_p will store as a variable in process_main , then will go into loop switch, execute the basic operations of current process,
when switch case is MFA, will go to `do_schedule` choose another process and excute it , then loop.

	When there is other task works, the task will add in run queue, the scheduler will balance when checking the run queue flags, `schedule` just pick out the process from run queue and excute it, then it do a loop.	

```javascript
void process_main(void)
{
...

do_schedule:
    reds_used = REDS_IN(c_p) - FCALLS;
 do_schedule1:
    PROCESS_MAIN_CHK_LOCKS(c_p);
    ERTS_SMP_UNREQ_PROC_MAIN_LOCK(c_p);
#if HALFWORD_HEAP
    ASSERT(erts_get_scheduler_data()->num_tmp_heap_used == 0);
#endif
    ERTS_VERIFY_UNUSED_TEMP_ALLOC(c_p);
    c_p = schedule(c_p, reds_used);
    ERTS_VERIFY_UNUSED_TEMP_ALLOC(c_p);
#ifdef DEBUG
    pid = c_p->id; /* Save for debugging purpouses */
#endif

...
}
```

	First , let us see the funcion erl_start_schedulers()
```javascript
void
erts_start_schedulers(void)
{
    int res = 0;
    Uint actual = 0;
    Uint wanted = erts_no_schedulers;
    Uint wanted_no_schedulers = erts_no_schedulers;
    ethr_thr_opts opts = ETHR_THR_OPTS_DEFAULT_INITER;

    opts.detached = 1;
    opts.suggested_stack_size = erts_sched_thread_suggested_stack_size;

    if (wanted < 1)
    wanted = 1;
    if (wanted > ERTS_MAX_NO_OF_SCHEDULERS) {
    wanted = ERTS_MAX_NO_OF_SCHEDULERS;
    res = ENOTSUP;
    }

    while (actual < wanted) {
    ErtsSchedulerData *esdp = ERTS_SCHEDULER_IX(actual);
    actual++;
    ASSERT(actual == esdp->no);
 res = ethr_thr_create(&esdp->tid,sched_thread_func,(void*)esdp,&opts);
    if (res != 0) {
        actual--;
        break;
    }
    }

    erts_no_schedulers = actual;

    ERTS_THR_MEMORY_BARRIER;

    res = ethr_thr_create(&aux_tid, aux_thread, NULL, &opts);
    if (res != 0)
    erl_exit(1, "Failed to create aux thread\n");

    if (actual < 1)
    erl_exit(1,
         "Failed to create any scheduler-threads: %s (%d)\n",
         erl_errno_id(res),
     res);
    if (res != 0) {
    erts_dsprintf_buf_t *dsbufp = erts_create_logger_dsbuf();
    ASSERT(actual != wanted_no_schedulers);
    erts_dsprintf(dsbufp,
              "Failed to create %beu scheduler-threads (%s:%d); "
              "only %beu scheduler-thread%s created.\n",
              wanted_no_schedulers, erl_errno_id(res), res,
              actual, actual == 1 ? " was" : "s were");
    erts_send_error_to_logger_nogl(dsbufp);
    }
}
```
	In funtion erts_start_schedulers, first do some init about some value, wanted stores the parameters system default define(erts_no_schedulers), wanted_no_schedulers stores system default define(erts_no_schedulers), actual init 0, next varable (want actual wanted_no_schedulers) change with the parameters we pass to system and system actual conditon. 
	 Some other opts about we defined when erl start, next, init a variable esdp(is a ErtsSchedulerData) for using later, then init  esdp with `ethr_thr_create(&aux_tid, aux_thread, NULL, &opts)`, judge if actual<wanted, or will cause error, judge if actual<1 ,or will cause error. this two judges mainly to verify whether the number of scheduers we set if valid.
	Last judge the return of ethr_thr_create, if res != 0, means create scheduler thread failed, if success, our init over.

Next ,let go into ethr_thr_create to see how to create a thread.
```javascript
int
ethr_thr_create(ethr_tid *tid, void * (*func)(void *), void *arg,
        ethr_thr_opts *opts)
{
    ethr_thr_wrap_data__ twd;
    pthread_attr_t attr;
    int res, dres;
    int use_stack_size = (opts && opts->suggested_stack_size >= 0
              ? opts->suggested_stack_size
              : -1 /* Use system default */);

#ifdef ETHR_MODIFIED_DEFAULT_STACK_SIZE
    if (use_stack_size < 0)
    use_stack_size = ETHR_MODIFIED_DEFAULT_STACK_SIZE;
#endif

#if ETHR_XCHK
    if (ethr_not_completely_inited__) {
    ETHR_ASSERT(0);
    return EACCES;
    }
    if (!tid || !func) {
    ETHR_ASSERT(0);
    return EINVAL;
    }
#endif

    ethr_atomic32_init(&twd.result, (ethr_sint32_t) -1);
    twd.tse = ethr_get_ts_event();
    twd.thr_func = func;
    twd.arg = arg;

    res = pthread_attr_init(&attr);
    if (res != 0)
    return res;

    /* Error cleanup needed after this point */

    /* Schedule child thread in system scope (if possible) ... */
    res = pthread_attr_setscope(&attr, PTHREAD_SCOPE_SYSTEM);
    if (res != 0 && res != ENOTSUP)
    goto error;

    if (use_stack_size >= 0) {
    size_t suggested_stack_size = (size_t) use_stack_size;
    size_t stack_size;
#ifdef ETHR_DEBUG
    suggested_stack_size /= 2; /* Make sure we got margin */
#endif
    if (suggested_stack_size < ethr_min_stack_size__)
        stack_size = ETHR_KW2B(ethr_min_stack_size__);
    else if (suggested_stack_size > ethr_max_stack_size__)
        stack_size = ETHR_KW2B(ethr_max_stack_size__);
    else
        stack_size = ETHR_PAGE_ALIGN(ETHR_KW2B(suggested_stack_size));
    (void) pthread_attr_setstacksize(&attr, stack_size);
    }

#ifdef ETHR_STACK_GUARD_SIZE
    (void) pthread_attr_setguardsize(&attr, ETHR_STACK_GUARD_SIZE);
#endif

    /* Detached or joinable... */
    res = pthread_attr_setdetachstate(&attr,
                      (opts && opts->detached
                       ? PTHREAD_CREATE_DETACHED
                       : PTHREAD_CREATE_JOINABLE));
    if (res != 0)
    goto error;

    /* Call prepare func if it exist */
    if (ethr_thr_prepare_func__)
    twd.prep_func_res = ethr_thr_prepare_func__();
    else
    twd.prep_func_res = NULL;

    res = pthread_create((pthread_t *) tid, &attr, thr_wrapper, (void*) &twd);

    if (res == 0) {
    int spin_count = child_wait_spin_count;

    /* Wait for child to initialize... */
    while (1) {
        ethr_sint32_t result;
        ethr_event_reset(&twd.tse->event);

        result = ethr_atomic32_read(&twd.result);
        if (result == 0)
        break;

        if (result > 0) {
        res = (int) result;
        goto error;
        }

        res = ethr_event_swait(&twd.tse->event, spin_count);
        if (res != 0 && res != EINTR)
        goto error;
        spin_count = 0;
    }
    }

    /* Cleanup... */

 error:
    dres = pthread_attr_destroy(&attr);
    if (res == 0)
    res = dres;
    if (ethr_thr_parent_func__)
    ethr_thr_parent_func__(twd.prep_func_res);
    return res;
}
```
	Fucntion ethr_thr_create mainly to create a scheduler thread, first define `pthread_attr attr`(for stroring thread attrs and informaitons), use_stack_size(if we define in erl start, it will be value we set when the value are checked is valid else it will default value).
	Call `pthread_attr_init(&attr)` to init the attrs in thread, if !=0, init failed else init success.
	Call `pthread_attr_setscope()` to Schedule child thread in system scope, if !=0, set error else set success.
	Call `pthread_attr_setdetachstate()` to set thread detach property.
	Call `pthread_create((pthread_t *) tid, &attr, thr_wrapper, (void*) &twd);` , `thr_wrapper` is used to create event, set the value of twd.
	Last whil loop, check the value of twd.result to initialize the child unitl to 0.
	Now the init funcion over.

```javascript
Process *schedule(Process *p, int calls)
{
    ErtsRunQueue *rq;
    ErtsRunPrioQueue *rpq;
    erts_aint_t dt;
    ErtsSchedulerData *esdp;
    int context_reds;
    int fcalls;
    int input_reductions;
    int actual_reds;
    int reds;
...

if (!p) {   /* NULL in the very first schedule() call */
    esdp = erts_get_scheduler_data();
    rq = erts_get_runq_current(esdp);
    ASSERT(esdp);
    fcalls = (int) erts_smp_atomic32_read_acqb(&function_calls);
    actual_reds = reds = 0;
    erts_smp_runq_lock(rq);
    } else {
#ifdef ERTS_SMP
    ERTS_SMP_CHK_HAVE_ONLY_MAIN_PROC_LOCK(p);
    esdp = p->scheduler_data;
    ASSERT(esdp->current_process == p
           || esdp->free_process == p);
#else
    esdp = erts_scheduler_data;
    ASSERT(esdp->current_process == p);
#endif
    reds = actual_reds = calls - esdp->virtual_reds;
    if (reds < ERTS_PROC_MIN_CONTEXT_SWITCH_REDS_COST)
        reds = ERTS_PROC_MIN_CONTEXT_SWITCH_REDS_COST;
    esdp->virtual_reds = 0;

    fcalls = (int) erts_smp_atomic32_add_read_acqb(&function_calls, reds);
    ASSERT(esdp && esdp == erts_get_scheduler_data());
...

check_activities_to_run: {

#ifdef ERTS_SMP

    if (rq->check_balance_reds <= 0)
        check_balance(rq);

    ERTS_SMP_LC_ASSERT(!erts_thr_progress_is_blocking());
    ERTS_SMP_LC_ASSERT(erts_smp_lc_runq_is_locked(rq));

    if (rq->flags & ERTS_RUNQ_FLGS_IMMIGRATE_QMASK)
        immigrate(rq);

 continue_check_activities_to_run:

    if (rq->flags & (ERTS_RUNQ_FLG_CHK_CPU_BIND
             | ERTS_RUNQ_FLG_SUSPENDED)) {
        if (rq->flags & ERTS_RUNQ_FLG_SUSPENDED) {
        ASSERT(erts_smp_atomic32_read_nob(&esdp->ssi->flags)
               & ERTS_SSI_FLG_SUSPENDED);
        suspend_scheduler(esdp);
        }
        if (rq->flags & ERTS_RUNQ_FLG_CHK_CPU_BIND)
        erts_sched_check_cpu_bind(esdp);
    }
...

  if (rq->ports.info.len) {
        int have_outstanding_io;
        have_outstanding_io = erts_port_task_execute(rq, &esdp->current_port);
        if ((have_outstanding_io && fcalls > 2*input_reductions)
        || rq->halt_in_progress) {
        /*
         * If we have performed more than 2*INPUT_REDUCTIONS since
         * last call to erl_sys_schedule() and we still haven't
         * handled all I/O tasks we stop running processes and
         * focus completely on ports.
         *
         * One could argue that this is a strange behavior. The
         * reason for doing it this way is that it is similar
         * to the behavior before port tasks were introduced.
         * We don't want to change the behavior too much, at
         * least not at the time of writing. This behavior
         * might change in the future.
         *
         * /rickard
         */
        goto check_activities_to_run;
        }
    }

    /*
     * Find a new process to run.
     */
 pick_next_process:

    ERTS_DBG_CHK_PROCS_RUNQ(rq);

      switch (rq->flags & ERTS_RUNQ_FLGS_PROCS_QMASK) {
    case MAX_BIT:
    case MAX_BIT|HIGH_BIT:
    case MAX_BIT|NORMAL_BIT:
    case MAX_BIT|LOW_BIT:
...

}
```
	Function has parameter process, this is the main erlang system process, this funcion update run queue and scheduler data in process, in erlang system , process is the main line , structs including all main datas, all operations are done in it. 
	Function schedule is used to schedule tasks,first define (run queue, prio queue, scheduler data).
	Then clean up after the process being scheduled out.
	Next define some jump block `check_activites_to_run`,`continue_check_activities_to_run`, check the rq's flags to udpate esdp, if `rq->flags & ERTS_RUNQ_FLGS_IMMIGRATE_QMASK`,will immigrate run queue rq; if `(rq->flags & (ERTS_RUNQ_FLG_CHK_CPU_BIND| ERTS_RUNQ_FLG_SUSPENDED))` will suspend_scheduler;if `rq->flags & ERTS_RUNQ_FLG_CHK_CPU_BIND` will check scheduler cpu bind of esdp; these all operations is used to check cureent rq status to find out whether to run rq tasks or go on waiting.
	When find a port to run, will excute `erts_port_task_execute(rq, &esdp->current_port)`, to do the task, another question If we have performed more than 2*INPUT_REDUCTIONS since last call to erl_sys_schedule() and we still haven't handled all I/O tasks we stop running processes and focus completely on ports.
	When find a new process, check the flag or rq, and accorrding the flags to redefine the (prio run queque)rpq's pointer to point to hte next process and previosus process. Then it will choose the procoss out of the queuqe.

Next, let's see `erts_port_task_execute`
```javascript
int
erts_port_task_execute(ErtsRunQueue *runq, Port **curr_port_pp)
{
    int port_was_enqueued = 0;
    Port *pp;
    ErtsPortTaskQueue *ptqp;
    ErtsPortTask *ptp;
    int res = 0;
    int reds = ERTS_PORT_REDS_EXECUTE;
    erts_aint_t io_tasks_executed = 0;
    int fpe_was_unmasked;

    ERTS_SMP_LC_ASSERT(erts_smp_lc_runq_is_locked(runq));

    ERTS_PT_CHK_PORTQ(runq);

    pp = pop_port(runq);
    if (!pp) {
    res = 0;
    goto done;
    }
...
...
```
	Schedule aims to pick out a execute process, when find a new port ,will execute port task instantly(switch current port), if not found, will go to pick process.
	When port tasks come in, scheduler will schedule the tasks use `erl_port_task_schedue()` and sequence them.
	`erts_port_task_execute`:Run all scheduled tasks for the first port in run queue. If new tasks appear while running reschedule port (free task is an exception; it is always handled instantly). erts_port_task_execute() is called by scheduler threads between scheduleing of processes. Sched lock should be held by caller.
