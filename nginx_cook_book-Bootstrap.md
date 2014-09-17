#Nginx源码分析之启动过程

nginx的启动过程代码主要分布在src/core以及src/os/unix目录下。启动流程的函数调用序列：main(src/core/nginx.c)→ngx_init_cycle(src/core/ngx_cycle.c)→ngx_master_process_cycle(src/os/)。nginx的启动过程就是围绕着这三个函数进行的。

main函数的处理过程总体上可以概括如下：

1. 简单初始化一些数据接结构和模块：ngx_debug_init、ngx_strerror_init、ngx_time_init、ngx_regex_init、ngx_log_init、ngx_ssl_init等
2. 获取并处理命令参数：ngx_get_options。
3. 初始化ngx_cycle_t结构体变量init_cycle，设置其log、pool字段等。
	<pre>
	ngx_memzero(&init_cycle, sizeof(ngx_cycle_t));
    init_cycle.log = log;
    ngx_cycle = &init_cycle;

    init_cycle.pool = ngx_create_pool(1024, log);
    if (init_cycle.pool == NULL) {
        return 1;
    }
	</pre>
4. 保存参数，设置全局变量：ngx_argc ngx_os_argv ngx_argv ngx_environ
5. 处理控制台命令行参数，设置init_cycle的字段。这些字段包括：conf_prefix、prefix（-p prefix）、conf_file(-c filename)、conf_param(-g directives)。此外，还设置init_cyle.log.log_level=NGX_LOG_INFO。
6. 调用ngx_os_init来设置一些和操作系统相关的全局变量：ngx_page_size、ngx_cacheline_size、ngx_ncpu、ngx_max_sockets、ngx_inherited_nonblocking。
7. 调用ngx_crc32_table_init初始化ngx_crc32_table_short。用于后续做crc校验。
8. 调用ngx_add_inherited_sockets(&init_cycle)→ngx_set_inherited_sockets，初始化init_cycle的listening字段（一个ngx_listening_t的数组）。
9. 对所有模块进行计数
	<pre>
	ngx_max_module = 0;
    for (i = 0; ngx_modules[i]; i++) {
        ngx_modules[i]->index = ngx_max_module++;
    }
	</pre>
10. 调用ngx_init_cycle进行模块的初始化，当解析配置文件错误时，退出程序。这个函数传入init_cycle然后返回一个新的ngx_cycle_t。
11. 调用ngx_signal_process、ngx_init_signals处理信号。
12. 在daemon模式下，调用ngx_daemon以守护进程的方式运行。这里可以在./configure的时候加入参数--with-debug，并在nginx.conf中配置:
	<pre>
	master_process  off; # 简化调试 此指令不得用于生产环境
	daemon          off; # 简化调试 此指令可以用到生产环境
	</pre>
可以取消守护进程模式以及master线程模型。
13. 调用ngx_create_pidfile创建pid文件，把master进程的pid保存在里面。
14. 根据进程模式来分别调用相应的函数
	<pre>
	if (ngx_process == NGX_PROCESS_SINGLE) {
        ngx_single_process_cycle(cycle);

    } else {
        ngx_master_process_cycle(cycle);
    }
	</pre>
	多进程的情况下，调用ngx_master_process_cycle。单进程的情况下调用ngx_single_process_cycle完成最后的启动工作。
	
整个启动过程中一个关键的变量init_cycle，其数据结构ngx_cycle_t如下所示：

<pre>
struct ngx_cycle_s {
    void                  ****conf_ctx;
    ngx_pool_t               *pool;

    ngx_log_t                *log;
    ngx_log_t                 new_log;

    ngx_connection_t        **files;
    ngx_connection_t         *free_connections;
    ngx_uint_t                free_connection_n;

    ngx_queue_t               reusable_connections_queue;

    ngx_array_t               listening;
    ngx_array_t               paths;
    ngx_list_t                open_files;
    ngx_list_t                shared_memory;

    ngx_uint_t                connection_n;
    ngx_uint_t                files_n;

    ngx_connection_t         *connections;
    ngx_event_t              *read_events;
    ngx_event_t              *write_events;

    ngx_cycle_t              *old_cycle;

    ngx_str_t                 conf_file;
    ngx_str_t                 conf_param;
    ngx_str_t                 conf_prefix;
    ngx_str_t                 prefix;
    ngx_str_t                 lock_file;
    ngx_str_t                 hostname;
};
</pre>

它保存了一次启动过程需要的一些资源。

ngx_init_cycle函数的处理过程如下：

1. 调用ngx_timezone_update()、ngx_timeofday、ngx_time_update()来更新时区、时间等，做时间校准，用来创建定时器等。
2. 创建pool,赋给一个新的cycle（ngx_cycle_t）。这个新的cycle的一些字段从旧的cycle传递过来，比如：log,conf_prefix,prefix,conf_file,conf_param。
	<pre>
	cycle->conf_prefix.len = old_cycle->conf_prefix.len;
    cycle->conf_prefix.data = ngx_pstrdup(pool, &old_cycle->conf_prefix);
    if (cycle->conf_prefix.data == NULL) {
        ngx_destroy_pool(pool);
        return NULL;
    }

    cycle->prefix.len = old_cycle->prefix.len;
    cycle->prefix.data = ngx_pstrdup(pool, &old_cycle->prefix);
    if (cycle->prefix.data == NULL) {
        ngx_destroy_pool(pool);
        return NULL;
    }

    cycle->conf_file.len = old_cycle->conf_file.len;
    cycle->conf_file.data = ngx_pnalloc(pool, old_cycle->conf_file.len + 1);
    if (cycle->conf_file.data == NULL) {
        ngx_destroy_pool(pool);
        return NULL;
    }
    ngx_cpystrn(cycle->conf_file.data, old_cycle->conf_file.data,
                old_cycle->conf_file.len + 1);

    cycle->conf_param.len = old_cycle->conf_param.len;
    cycle->conf_param.data = ngx_pstrdup(pool, &old_cycle->conf_param);
    if (cycle->conf_param.data == NULL) {
        ngx_destroy_pool(pool);
        return NULL;
    }
	</pre>
还有一些字段会首先判断old_cycle中是否存在，如果存在，则申请同样大小的空间，并初始化。这些字段如下：
	<pre>
	n = old_cycle->paths.nelts ? old_cycle->paths.nelts : 10;

	//paths
    cycle->paths.elts = ngx_pcalloc(pool, n * sizeof(ngx_path_t *));
    if (cycle->paths.elts == NULL) {
        ngx_destroy_pool(pool);
        return NULL;
    }

    cycle->paths.nelts = 0;
    cycle->paths.size = sizeof(ngx_path_t *);
    cycle->paths.nalloc = n;
    cycle->paths.pool = pool;


    //open_files
    if (old_cycle->open_files.part.nelts) {
        n = old_cycle->open_files.part.nelts;
        for (part = old_cycle->open_files.part.next; part; part = part->next) {
            n += part->nelts;
        }

    } else {
        n = 20;
    }
    
    //shared_memory
    if (old_cycle->shared_memory.part.nelts) {
        n = old_cycle->shared_memory.part.nelts;
        for (part = old_cycle->shared_memory.part.next; part; part = part->next)
        {
            n += part->nelts;
        }

    } else {
        n = 1;
    }
    
    //listening
    n = old_cycle->listening.nelts ? old_cycle->listening.nelts : 10;

    cycle->listening.elts = ngx_pcalloc(pool, n * sizeof(ngx_listening_t));
    if (cycle->listening.elts == NULL) {
        ngx_destroy_pool(pool);
        return NULL;
    }

    cycle->listening.nelts = 0;
    cycle->listening.size = sizeof(ngx_listening_t);
    cycle->listening.nalloc = n;
    cycle->listening.pool = pool;
	</pre>
	此外，new_log.log_level重新赋值的为NGX_LOG_ERR；old_cycle为传递进来的cycle；hostname为gethostname;初始化resuable_connection_queue。

	这里有一个关键变量的初始化：conf_ctx。初始化为ngx_max_module个void *指针。说明其实所有模块的配置结构的指针。

3. 调用所有模块的crea_conf，返回的配置结构指针放到conf_ctx数组中，索引为ngx_modules[i]->index。
4. 从命令行和配置文件读取配置更新到conf_ctx中。ngx_conf_param是读取命令行中的指令，ngx_conf_parse是把读取配置文件。ngx_conf_param最后也是通过调用ngx_cong_parse来读取配置的。ngx_conf_parse函数中有一个for循环，每次都调用ngx_conf_read_token取得一个配置指令，然后调用ngx_conf_handler来处理这条指令。ngx_conf_handler每次会遍历所有模块的指令集，查找这条配置指令并分析其合法性，如果正确则创建配置结构并把指针加入到cycle.conf_ctx中。
	遍历指令集的过程首先是遍历所有的核心类模块，若是 event类的指令，则会遍历到ngx_events_module，这个模块是属于核心类的，其钩子set又会嵌套调用ngx_conf_parse去遍历所有的event类模块，同样的，若是http类指令，则会遍历到ngx_http_module，该模块的钩子set进一步遍历所有的http类模块，mail类指令会遍历到ngx_mail_module，该模块的钩子进一步遍历到所有的mail类模块。要特别注意的是：这三个遍历过程中会在适当的时机调用event类模块、http类模块和mail类模块的创建配置和初始化配置的钩子。从这里可以看出，event、http、mail三类模块的钩子是配置中的指令驱动的。
5. 调用core module的init_conf。
6. 读取核心模块ngx_core_module的配置结构，调用ngx_create_pidfile创建pid文件。
	<pre>
	ccf = (ngx_core_conf_t *) ngx_get_conf(cycle->conf_ctx, ngx_core_module);
	</pre>
	
	这里代码中有一句注释：
	<pre>
     we do not create the pid file in the first ngx_init_cycle() call
     because we need to write the demonized process pid
	</pre>
	当不是第一次初始化cycles时才会调用ngx_create_pidfile写入pid。
7. 调用ngx_test_lockfile,ngx_create_paths并打开error_log文件复制给cycle->new_log.file。
8. 遍历cycle的open_files.part.elts，打开每一个文件。open_files填充的文件数据是读取配置文件时写入的。
9. 创建共享内存。这里和对open_files类似。先预分配空间，再填充数据。
10. 处理listening sockets，遍历cycle->listening数组与old_cycle->listenning进行比较，设置cycle->listening的一些状态信息，调用ngx_open_listening_sockets启动所有监听socket，循环调用socket、bind、listen完成服务端监听监听socket的启动。并调用ngx_configure_listening_sockets配置监听socket,根据ngx_listening_t中的状态信息设置socket的读写缓存和TCP_DEFER_ACCEPT。
11. 调用每个module的init_module。
12. 关闭或者删除一些残留在old_cycle中的资源，首先释放不用的共性内存，关闭不使用的监听socket，再关闭不适用的打开文件。最后把old_cycle放入ngx_old_cycles。最后再设定一个定时器，定期回调ngx_cleaner_event清理ngx_old_cycles。周期设置为30000ms。