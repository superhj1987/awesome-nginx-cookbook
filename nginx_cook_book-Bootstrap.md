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
7. 调用ngx_crc32_table_init初始化ngx_crc32_table_short。用于计算hash key等。
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
