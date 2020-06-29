---
title: pppd源码详解一 （插件加载）
type: linuxc
description: 本文对pppd源码插件加载流程进行解读
date: 2020-06-22 19:10:42
---

本文使用 `pppd-2.4.5`在`CentOS-7.5.1804`进行测试.

## 获取rpm包中源码

```
wget http://vault.centos.org/7.5.1804/os/Source/SPackages/ppp-2.4.5-33.el7.src.rpm
rpm -ivh ppp-2.4.5-33.el7.src.rpm
ls ./rpmbuild/SOURCE/ppp-2.4.5.tar.gz
```

## 说明

解读pppd的源码，重点想知道如下信息：

1. pppd 如何加载插件？
2. pppd 如何发起 ppp的 discovery连接？
3. pppd 对各种 ppp协议的处理逻辑？
4. pppd 如何创建虚端口
   
## main 函数解读

```c
int
main(argc, argv)
    int argc;
    char *argv[];
{
    int i, t;
    char *p;
    struct passwd *pw;
    struct protent *protp;
    char numbuf[16];

    link_stats_valid = 0;
    new_phase(PHASE_INITIALIZE);

    script_env = NULL;

    /* Initialize syslog facilities */
    reopen_log();

    if (gethostname(hostname, MAXNAMELEN) < 0 ) {
	option_error("Couldn't get hostname: %m");
	exit(1);
    }
    hostname[MAXNAMELEN-1] = 0;

    /* make sure we don't create world or group writable files. */
    umask(umask(0777) | 022);

    uid = getuid();
    privileged = uid == 0;
    slprintf(numbuf, sizeof(numbuf), "%d", uid);
    script_setenv("ORIG_UID", numbuf, 0);

    ngroups = getgroups(NGROUPS_MAX, groups);

    /*
     * Initialize magic number generator now so that protocols may
     * use magic numbers in initialization.
     */
    magic_init();

    /*
     * Initialize each protocol.各种协议初始化
     */
    for (i = 0; (protp = protocols[i]) != NULL; ++i)
        (*protp->init)(0);

    /*
     * Initialize the default channel.
     */
    tty_init();

    progname = *argv;

    /*
     * Parse, in order, the system options file, the user's options file,
     * and the command line arguments.
     * 分别从三个地方获取参数，文件、用户、命令行，然后解析
     */
    if (!options_from_file(_PATH_SYSOPTIONS, !privileged, 0, 1)
	|| !options_from_user()
	|| !parse_args(argc-1, argv+1))
	exit(EXIT_OPTION_ERROR);
    devnam_fixed = 1;		/* can no longer change device name */

    /*
     * Work out the device name, if it hasn't already been specified,
     * and parse the tty's options file.
     */
    if (the_channel->process_extra_options)
	(*the_channel->process_extra_options)();

    if (debug)
	setlogmask(LOG_UPTO(LOG_DEBUG));

    /*
     * Check that we are running as root.
     */
    if (geteuid() != 0) {
	option_error("must be root to run %s, since it is not setuid-root",
		     argv[0]);
	exit(EXIT_NOT_ROOT);
    }

    if (!ppp_available()) {
	option_error("%s", no_ppp_msg);
	exit(EXIT_NO_KERNEL_SUPPORT);
    }

    /*
     * Check that the options given are valid and consistent.
     * 执行每个协议的 check_options 函数
     */
    check_options();
    if (!sys_check_options())
	exit(EXIT_OPTION_ERROR);
    auth_check_options();
#ifdef HAVE_MULTILINK
    mp_check_options();
#endif
    for (i = 0; (protp = protocols[i]) != NULL; ++i)
	if (protp->check_options != NULL)
	    (*protp->check_options)();
    if (the_channel->check_options)
	(*the_channel->check_options)();


    if (dump_options || dryrun) {
	init_pr_log(NULL, LOG_INFO);
	print_options(pr_log, NULL);
	end_pr_log();
    }

    if (dryrun)
	die(0);

    /* Make sure fds 0, 1, 2 are open to somewhere. */
    fd_devnull = open(_PATH_DEVNULL, O_RDWR);
    if (fd_devnull < 0)
	fatal("Couldn't open %s: %m", _PATH_DEVNULL);
    while (fd_devnull <= 2) {
	i = dup(fd_devnull);
	if (i < 0)
	    fatal("Critical shortage of file descriptors: dup failed: %m");
	fd_devnull = i;
    }

    /*
     * Initialize system-dependent stuff.
     */
    sys_init();

#ifdef USE_TDB      /* 使用宏 隔开的语句其实都不用看，用宏隔开，说明可有可无*/
    pppdb = tdb_open(_PATH_PPPDB, 0, 0, O_RDWR|O_CREAT, 0644);
    if (pppdb != NULL) {
	slprintf(db_key, sizeof(db_key), "pppd%d", getpid());
	update_db_entry();
    } else {
	warn("Warning: couldn't open ppp database %s", _PATH_PPPDB);
	if (multilink) {
	    warn("Warning: disabling multilink");
	    multilink = 0;
	}
    }
#endif

    /*
     * Detach ourselves from the terminal, if required,
     * and identify who is running us.
     */
    if (!nodetach && !updetach)
	detach();
    p = getlogin();
    if (p == NULL) {
	pw = getpwuid(uid);
	if (pw != NULL && pw->pw_name != NULL)
	    p = pw->pw_name;
	else
	    p = "(unknown)";
    }
    syslog(LOG_NOTICE, "pppd %s started by %s, uid %d", VERSION, p, uid);
    script_setenv("PPPLOGNAME", p, 0);

    if (devnam[0])
	script_setenv("DEVICE", devnam, 1);
    slprintf(numbuf, sizeof(numbuf), "%d", getpid());
    script_setenv("PPPD_PID", numbuf, 1);

    setup_signals();

    create_linkpidfile(getpid());

    waiting = 0;

    /*
     * If we're doing dial-on-demand, set up the interface now.
     * 这个条件想成立，除非显示在命令行指定
     */
    if (demand) {
	/*
	 * Open the loopback channel and set it up to be the ppp interface.
	 */
	fd_loop = open_ppp_loopback();  /*创建PPP虚设备*/
	set_ifunit(1);
	/*
	 * Configure the interface and mark it up, etc.
	 */
	demand_conf();
    }

    do_callback = 0;
    for (;;) {

	bundle_eof = 0;
	bundle_terminating = 0;
	listen_time = 0;
	need_holdoff = 1;
	devfd = -1;
	status = EXIT_OK;
	++unsuccess;
	doing_callback = do_callback;
	do_callback = 0;

	if (demand && !doing_callback) {
	    /*
	     * Don't do anything until we see some activity.
	     */
	    new_phase(PHASE_DORMANT);
	    demand_unblock();
	    add_fd(fd_loop);
	    for (;;) {
		handle_events();
		if (asked_to_quit)
		    break;
		if (get_loop_output())
		    break;
	    }
	    remove_fd(fd_loop);
	    if (asked_to_quit)
		break;

	    /*
	     * Now we want to bring up the link.
	     */
	    demand_block();
	    info("Starting link");
	}

	gettimeofday(&start_time, NULL);
	script_unsetenv("CONNECT_TIME");
	script_unsetenv("BYTES_SENT");
	script_unsetenv("BYTES_RCVD");

	lcp_open(0);		/* Start protocol */
    /* start_link 调用 rp-pppoe 插件，发起pppoe的 discovery连接，这个很重要*/
	start_link(0);
    /* 下面开始，就已经是ppp的 session 阶段了*/
	while (phase != PHASE_DEAD) {
	    handle_events();    /* 监控各个fd，一旦就绪，就处理*/
	    get_input();        /*  这个函数 处理 sesseion 的输入*/
	    if (kill_link)
		lcp_close(0, "User request");
	    if (asked_to_quit) {
		bundle_terminating = 1;
		if (phase == PHASE_MASTER)
		    mp_bundle_terminated();
	    }
	    if (open_ccp_flag) {
		if (phase == PHASE_NETWORK || phase == PHASE_RUNNING) {
		    ccp_fsm[0].flags = OPT_RESTART; /* clears OPT_SILENT */
		    (*ccp_protent.open)(0);
		}
	    }
	}
	/* restore FSMs to original state */
	lcp_close(0, "");

	if (!persist || asked_to_quit || (maxfail > 0 && unsuccess >= maxfail))
	    break;

	if (demand)
	    demand_discard();
	t = need_holdoff? holdoff: 0;
	if (holdoff_hook)
	    t = (*holdoff_hook)();
	if (t > 0) {
	    new_phase(PHASE_HOLDOFF);
	    TIMEOUT(holdoff_end, NULL, t);
	    do {
		handle_events();
		if (kill_link)
		    new_phase(PHASE_DORMANT); /* allow signal to end holdoff */
	    } while (phase == PHASE_HOLDOFF);
	    if (!persist)
		break;
	}
    }

    /* Wait for scripts to finish */
    reap_kids();
    if (n_children > 0) {
	if (child_wait > 0)
	    TIMEOUT(childwait_end, NULL, child_wait);
	if (debug) {
	    struct subprocess *chp;
	    dbglog("Waiting for %d child processes...", n_children);
	    for (chp = children; chp != NULL; chp = chp->next)
		dbglog("  script %s, pid %d", chp->prog, chp->pid);
	}
	while (n_children > 0 && !childwait_done) {
	    handle_events();
	    if (kill_link && !childwait_done)
		childwait_end(NULL);
	}
    }

    die(status);
    return 0;
}
```

## pppd 如何加载插件？

1. 举个例子，我这里有个已经成功拨号的环境，查看pppd进程运行信息如下：

![pppd进程](images/pppoe.png)

如上图： `/usr/sbin/pppd plugin /usr/lib64/pppd/2.4.5/rp-pppoe.so ... `, pppd 后面 + 字段 `plugin` 再 + `插件绝对路径` 这样就调用了 `rp-pppoe.so` 动态库，
那么对照代码，如何实现呢？

2. 代码解读

首先还是，`main` 函数 调用 `parse_args(argc-1, argv+1)` ,一切玄机都在这个 命令行解析里面


* `parse_args(argc-1, argv+1)`

```c
int
parse_args(argc, argv)
    int argc;
    char **argv;
{
    char *arg;
    option_t *opt;
    int n;

    privileged_option = privileged;
    option_source = "command line";
    option_priority = OPRIO_CMDLINE;
    while (argc > 0) {
	arg = *argv++;
	--argc;
	opt = find_option(arg);     /*玄机都在这个 find_option 函数中*/
	if (opt == NULL) {
	    option_error("unrecognized option '%s'", arg);
	    usage();
	    return 0;
	}
	n = n_arguments(opt);
	if (argc < n) {
	    option_error("too few parameters for option %s", arg);
	    return 0;
	}
	if (!process_option(opt, arg, argv))
	    return 0;
	argc -= n;
	argv += n;
    }
    return 1;
}
```

* `find_option(arg)`

```c
static option_t *
find_option(name)
    const char *name;
{
	option_t *opt;
	struct option_list *list;
	int i, dowild;

	for (dowild = 0; dowild <= 1; ++dowild) {
		for (opt = general_options; opt->name != NULL; ++opt)
			if (match_option(name, opt, dowild))
				return opt;
		for (opt = auth_options; opt->name != NULL; ++opt)
			if (match_option(name, opt, dowild))
				return opt;
		for (list = extra_options; list != NULL; list = list->next)
			for (opt = list->options; opt->name != NULL; ++opt)
				if (match_option(name, opt, dowild))
					return opt;
		for (opt = the_channel->options; opt->name != NULL; ++opt)
			if (match_option(name, opt, dowild))
				return opt;
		for (i = 0; protocols[i] != NULL; ++i)
			if ((opt = protocols[i]->options) != NULL)
				for (; opt->name != NULL; ++opt)
					if (match_option(name, opt, dowild))
						return opt;
	}
	return NULL;
}
```

* 全局变量 `general_options`

```c
/*
 * Valid arguments.
 */
option_t general_options[] = {
    { "debug", o_int, &debug,
      "Increase debugging level", OPT_INC | OPT_NOARG | 1 },
    ...
    ...
    ...

#ifdef PLUGIN
    { "plugin", o_special, (void *)loadplugin,      /* 如果参数匹配 plugin, 那么调用函数 loadplugin 加载插件*/
      "Load a plug-in module into pppd", OPT_PRIV | OPT_A2LIST },
#endif

#ifdef MAXOCTETS
    { "maxoctets", o_int, &maxoctets,
      "Set connection traffic limit",
      OPT_PRIO | OPT_LLIMIT | OPT_NOINCR | OPT_ZEROINF },
    { "mo", o_int, &maxoctets,
      "Set connection traffic limit",
      OPT_ALIAS | OPT_PRIO | OPT_LLIMIT | OPT_NOINCR | OPT_ZEROINF },
    { "mo-direction", o_special, setmodir,
      "Set direction for limit traffic (sum,in,out,max)" },
    { "mo-timeout", o_int, &maxoctets_timeout,
      "Check for traffic limit every N seconds", OPT_PRIO | OPT_LLIMIT | 1 },
#endif

    { NULL }
};
```

* `loadplugin` 相信到这里，你就明白这个插件是如何加载的了
  
```c
static int
loadplugin(argv)
    char **argv;
{
    char *arg = *argv;
    void *handle;
    const char *err;
    void (*init) __P((void));
    char *path = arg;
    const char *vers;

    if (strchr(arg, '/') == 0) {
	const char *base = _PATH_PLUGIN;
	int l = strlen(base) + strlen(arg) + 2;
	path = malloc(l);
	if (path == 0)
	    novm("plugin file path");
	strlcpy(path, base, l);
	strlcat(path, "/", l);
	strlcat(path, arg, l);
    }
    handle = dlopen(path, RTLD_GLOBAL | RTLD_NOW);
    if (handle == 0) {
	err = dlerror();
	if (err != 0)
	    option_error("%s", err);
	option_error("Couldn't load plugin %s", arg);
	goto err;
    }
    init = (void (*)(void))dlsym(handle, "plugin_init");
    if (init == 0) {
	option_error("%s has no initialization entry point", arg);
	goto errclose;
    }
    vers = (const char *) dlsym(handle, "pppd_version");
    if (vers == 0) {
	warn("Warning: plugin %s has no version information", arg);
    } else if (strcmp(vers, VERSION) != 0) {
	option_error("Plugin %s is for pppd version %s, this is %s",
		     arg, vers, VERSION);
	goto errclose;
    }
    info("Plugin %s loaded.", arg);
    (*init)();
    return 1;

 errclose:
    dlclose(handle);
 err:
    if (path != arg)
	free(path);
    return 0;
}
```
