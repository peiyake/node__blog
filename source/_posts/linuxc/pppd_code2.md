---
title: pppd源码详解二 --（discovery 连接）
type: linuxc
description: 本文对pppd源码（discovery 连接）流程进行解读
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

## pppd 如何发起 ppp的 discovery连接？

照旧 `main-> start_link`

## start_link

```
void start_link(unit)
    int unit;
{
    char *msg;

    new_phase(PHASE_SERIALCONN);

    hungup = 0;
    devfd = the_channel->connect();     /* the_channel 的值 在 rp-pppoe插件加载的时候已经被初始化为，插件的pppoe_channel,（pppd/plugins/rp-pppoe/plugin.c）如下：
                                        
                                        struct channel pppoe_channel = {
                                            .options = Options,
                                            .process_extra_options = &PPPOEDeviceOptions,
                                            .check_options = pppoe_check_options,
                                            .connect = &PPPOEConnectDevice,
                                            .disconnect = &PPPOEDisconnectDevice,
                                            .establish_ppp = &generic_establish_ppp,
                                            .disestablish_ppp = &generic_disestablish_ppp,
                                            .send_config = NULL,
                                            .recv_config = &PPPOERecvConfig,
                                            .close = NULL,
                                            .cleanup = NULL
                                        };                                  
                                        */
    msg = "Connect script failed";
    if (devfd < 0)
	goto fail;

    /* set up the serial device as a ppp interface */
    /*
     * N.B. we used to do tdb_writelock/tdb_writeunlock around this
     * (from establish_ppp to set_ifunit).  However, we won't be
     * doing the set_ifunit in multilink mode, which is the only time
     * we need the atomicity that the tdb_writelock/tdb_writeunlock
     * gives us.  Thus we don't need the tdb_writelock/tdb_writeunlock.
     */
    fd_ppp = the_channel->establish_ppp(devfd);
    msg = "ppp establishment failed";
    if (fd_ppp < 0) {
	status = EXIT_FATAL_ERROR;
	goto disconnect;
    }

    if (!demand && ifunit >= 0)
	set_ifunit(1);

    /*
     * Start opening the connection and wait for
     * incoming events (reply, timeout, etc.).
     */
    if (ifunit >= 0)
	notice("Connect: %s <--> %s", ifname, ppp_devnam);
    else
	notice("Starting negotiation on %s", ppp_devnam);
    add_fd(fd_ppp);

    status = EXIT_NEGOTIATION_FAILED;
    new_phase(PHASE_ESTABLISH);

    lcp_lowerup(0);
    return;

 disconnect:
    new_phase(PHASE_DISCONNECT);
    if (the_channel->disconnect)
	the_channel->disconnect();

 fail:
    new_phase(PHASE_DEAD);
    if (the_channel->cleanup)
	(*the_channel->cleanup)();
}
```

## the_channel->connect()

```c
static int
PPPOEConnectDevice(void)
{
    struct sockaddr_pppox sp;

    strlcpy(ppp_devnam, devnam, sizeof(ppp_devnam));
    if (existingSession) {
	unsigned int mac[ETH_ALEN];
	int i, ses;
	if (sscanf(existingSession, "%d:%x:%x:%x:%x:%x:%x",
		   &ses, &mac[0], &mac[1], &mac[2],
		   &mac[3], &mac[4], &mac[5]) != 7) {
	    fatal("Illegal value for rp_pppoe_sess option");
	}
	conn->session = htons(ses);
	for (i=0; i<ETH_ALEN; i++) {
	    conn->peerEth[i] = (unsigned char) mac[i];
	}
    } else {
	discovery(conn);    /*重点就是这个 discovery 函数， 这个函数执行成功，那么conn->discoveryState 设置成 STATE_SESSION， discovery就结束，进入 session阶段了*/
	if (conn->discoveryState != STATE_SESSION) {
	    error("Unable to complete PPPoE Discovery");
	    return -1;
	}
    }

    /* Set PPPoE session-number for further consumption */
    ppp_session_number = ntohs(conn->session);

    /* Make the session socket */
    conn->sessionSocket = socket(AF_PPPOX, SOCK_STREAM, PX_PROTO_OE);
    if (conn->sessionSocket < 0) {
	error("Failed to create PPPoE socket: %m");
	goto errout;
    }
    sp.sa_family = AF_PPPOX;
    sp.sa_protocol = PX_PROTO_OE;
    sp.sa_addr.pppoe.sid = conn->session;
    memcpy(sp.sa_addr.pppoe.dev, conn->ifName, IFNAMSIZ);
    memcpy(sp.sa_addr.pppoe.remote, conn->peerEth, ETH_ALEN);

    /* Set remote_number for ServPoET */
    sprintf(remote_number, "%02X:%02X:%02X:%02X:%02X:%02X",
	    (unsigned) conn->peerEth[0],
	    (unsigned) conn->peerEth[1],
	    (unsigned) conn->peerEth[2],
	    (unsigned) conn->peerEth[3],
	    (unsigned) conn->peerEth[4],
	    (unsigned) conn->peerEth[5]);

    warn("Connected to %02X:%02X:%02X:%02X:%02X:%02X via interface %s",
	 (unsigned) conn->peerEth[0],
	 (unsigned) conn->peerEth[1],
	 (unsigned) conn->peerEth[2],
	 (unsigned) conn->peerEth[3],
	 (unsigned) conn->peerEth[4],
	 (unsigned) conn->peerEth[5],
	 conn->ifName);

    script_setenv("MACREMOTE", remote_number, 0);

    if (connect(conn->sessionSocket, (struct sockaddr *) &sp,
		sizeof(struct sockaddr_pppox)) < 0) {
	error("Failed to connect PPPoE socket: %d %m", errno);
	close(conn->sessionSocket);
	goto errout;
    }

    warn("PPPOEConnectDevice ok, session[%d]!",conn->session);
    return conn->sessionSocket;

 errout:
    if (conn->discoverySocket >= 0) {
	sendPADT(conn, NULL);
	close(conn->discoverySocket);
	conn->discoverySocket = -1;
    }
    return -1;
}
```