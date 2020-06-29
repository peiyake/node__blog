---
title: pppd源码详解三 ---（ppp协议处理）
type: linuxc
description: 本文对pppd源码（ppp协议处理）流程进行解读
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

## PPP协议处理

有哪些ppp协议呢？

```
/*
 * Protocol field values.
 */
#define PPP_IP		0x21	/* Internet Protocol */
#define PPP_AT		0x29	/* AppleTalk Protocol */
#define PPP_IPX		0x2b	/* IPX protocol */
#define	PPP_VJC_COMP	0x2d	/* VJ compressed TCP */
#define	PPP_VJC_UNCOMP	0x2f	/* VJ uncompressed TCP */
#define PPP_MP		0x3d	/* Multilink protocol */
#define PPP_IPV6	0x57	/* Internet Protocol Version 6 */
#define PPP_COMPFRAG	0xfb	/* fragment compressed below bundle */
#define PPP_COMP	0xfd	/* compressed packet */
#define PPP_IPCP	0x8021	/* IP Control Protocol */
#define PPP_ATCP	0x8029	/* AppleTalk Control Protocol */
#define PPP_IPXCP	0x802b	/* IPX Control Protocol */
#define PPP_IPV6CP	0x8057	/* IPv6 Control Protocol */
#define PPP_CCPFRAG	0x80fb	/* CCP at link level (below MP bundle) */
#define PPP_CCP		0x80fd	/* Compression Control Protocol */
#define PPP_ECPFRAG	0x8055	/* ECP at link level (below MP bundle) */
#define PPP_ECP		0x8053	/* Encryption Control Protocol */
#define PPP_LCP		0xc021	/* Link Control Protocol */
#define PPP_PAP		0xc023	/* Password Authentication Protocol */
#define PPP_LQR		0xc025	/* Link Quality Report protocol */
#define PPP_CHAP	0xc223	/* Cryptographic Handshake Auth. Protocol */
#define PPP_CBCP	0xc029	/* Callback Control Protocol */
```

这里我们介绍一下，`PPP_IPCP` ，这个协议用来为ppp客户端分配网络配置（IP/dns等）

### 代码流程

`main` -> `get_input()`

* `get_input`

需要注意的是，这里收到的ppp报文，是不包含 ppp协议的头的前4个字节的，下图：

![ppp协议](images/pppcap.png)

```c
static void
get_input()
{
    int len, i;
    u_char *p;
    u_short protocol;
    struct protent *protp;

    p = inpacket_buf;	/* point to beginning of packet buffer */

    len = read_packet(inpacket_buf);
    if (len < 0)
	return;

    if (len == 0) {
	if (bundle_eof && multilink_master) {
	    notice("Last channel has disconnected");
	    mp_bundle_terminated();
	    return;
	}
	notice("Modem hangup");
	hungup = 1;
	status = EXIT_HANGUP;
	lcp_lowerdown(0);	/* serial link is no longer available */
	link_terminated(0);
	return;
    }

    if (len < PPP_HDRLEN) {
	dbglog("received short packet:%.*B", len, p);
	return;
    }

    dump_packet("rcvd", p, len);
    if (snoop_recv_hook) snoop_recv_hook(p, len);

    p += 2;				/* Skip address and control, 参照上图，就知道为什么这里 + 2 了， */
    GETSHORT(protocol, p);  /* 获取到协议ID*/
    len -= PPP_HDRLEN;

    /*
     * Toss all non-LCP packets unless LCP is OPEN.
     */
    if (protocol != PPP_LCP && lcp_fsm[0].state != OPENED) {
	dbglog("Discarded non-LCP packet when LCP not open");
	return;
    }

    /*
     * Until we get past the authentication phase, toss all packets
     * except LCP, LQR and authentication packets.
     */
    if (phase <= PHASE_AUTHENTICATE
	&& !(protocol == PPP_LCP || protocol == PPP_LQR
	     || protocol == PPP_PAP || protocol == PPP_CHAP ||
		protocol == PPP_EAP)) {
	dbglog("discarding proto 0x%x in phase %d",
		   protocol, phase);
	return;
    }

    /*
     * Upcall the proper protocol input routine.
     */

     /*重点就是这个for循环，这里一次调用了每个协议的的input函数
     */
    for (i = 0; (protp = protocols[i]) != NULL; ++i) {
	if (protp->protocol == protocol && protp->enabled_flag) {
	    (*protp->input)(0, p, len);
	    return;
	}
        if (protocol == (protp->protocol & ~0x8000) && protp->enabled_flag
	    && protp->datainput != NULL) {
	    (*protp->datainput)(0, p, len);
	    return;
	}
    }

    if (debug) {
	const char *pname = protocol_name(protocol);
	if (pname != NULL)
	    warn("Unsupported protocol '%s' (0x%x) received", pname, protocol);
	else
	    warn("Unsupported protocol 0x%x received", protocol);
    }
    lcp_sprotrej(0, p - PPP_HDRLEN, len + PPP_HDRLEN);
}
```

```
struct protent *protocols[] = {
    &lcp_protent,
    &pap_protent,
    &chap_protent,
#ifdef CBCP_SUPPORT
    &cbcp_protent,
#endif
    &ipcp_protent,
#ifdef INET6
    &ipv6cp_protent,
#endif
    &ccp_protent,
    &ecp_protent,
#ifdef IPX_CHANGE
    &ipxcp_protent,
#endif
#ifdef AT_CHANGE
    &atcp_protent,
#endif
    &eap_protent,
    NULL
};

struct protent ipcp_protent = {
    PPP_IPCP,
    ipcp_init,
    ipcp_input,
    ipcp_protrej,
    ipcp_lowerup,
    ipcp_lowerdown,
    ipcp_open,
    ipcp_close,
    ipcp_printpkt,
    NULL,
    1,
    "IPCP",
    "IP",
    ipcp_option_list,
    ip_check_options,
    ip_demand_conf,
    ip_active_pkt
};
```

* `ipcp_input`

```c
static void
ipcp_input(unit, p, len)
    int unit;
    u_char *p;
    int len;
{
    fsm_input(&ipcp_fsm[unit], p, len);
}

```

* `fsm_input`

根据连接状态机，分阶段处理

```c
void
fsm_input(f, inpacket, l)
    fsm *f;
    u_char *inpacket;
    int l;
{
    u_char *inp;
    u_char code, id;
    int len;

    /*
     * Parse header (code, id and length).
     * If packet too short, drop it.
     */
    inp = inpacket;
    if (l < HEADERLEN) {
	FSMDEBUG(("fsm_input(%x): Rcvd short header.", f->protocol));
	return;
    }
    GETCHAR(code, inp);
    GETCHAR(id, inp);
    GETSHORT(len, inp);
    if (len < HEADERLEN) {
	FSMDEBUG(("fsm_input(%x): Rcvd illegal length.", f->protocol));
	return;
    }
    if (len > l) {
	FSMDEBUG(("fsm_input(%x): Rcvd short packet.", f->protocol));
	return;
    }
    len -= HEADERLEN;		/* subtract header length */

    if( f->state == INITIAL || f->state == STARTING ){
	FSMDEBUG(("fsm_input(%x): Rcvd packet in state %d.",
		  f->protocol, f->state));
	return;
    }

    /*
     * Action depends on code.
     */
    switch (code) {
    case CONFREQ:
	fsm_rconfreq(f, id, inp, len);
	break;
    
    case CONFACK:
	fsm_rconfack(f, id, inp, len);
	break;
    
    case CONFNAK:
    case CONFREJ:
	fsm_rconfnakrej(f, code, id, inp, len);
	break;
    
    case TERMREQ:
	fsm_rtermreq(f, id, inp, len);
	break;
    
    case TERMACK:
	fsm_rtermack(f);
	break;
    
    case CODEREJ:
	fsm_rcoderej(f, inp, len);
	break;
    
    default:
	if( !f->callbacks->extcode
	   || !(*f->callbacks->extcode)(f, code, id, inp, len) )
	    fsm_sdata(f, CODEREJ, ++f->id, inpacket, len + HEADERLEN);
	break;
    }
}
```

* 如何解析IP地址，并设置生效？

```c
static void
fsm_rconfreq(f, id, inp, len)
    fsm *f;
    u_char id;
    u_char *inp;
    int len;
{
    int code, reject_if_disagree;

    switch( f->state ){
    case CLOSED:
	/* Go away, we're closed */
	fsm_sdata(f, TERMACK, id, NULL, 0);
	return;
    case CLOSING:
    case STOPPING:
	return;

    case OPENED:
	/* Go down and restart negotiation */
	if( f->callbacks->down )
	    (*f->callbacks->down)(f);	/* Inform upper layers */
	fsm_sconfreq(f, 0);		/* Send initial Configure-Request */
	f->state = REQSENT;
	break;

    case STOPPED:
	/* Negotiation started by our peer */
	fsm_sconfreq(f, 0);		/* Send initial Configure-Request */
	f->state = REQSENT;
	break;
    }

    /*
     * Pass the requested configuration options
     * to protocol-specific code for checking.
     */
    if (f->callbacks->reqci){		/* Check CI */
	reject_if_disagree = (f->nakloops >= f->maxnakloops);
	code = (*f->callbacks->reqci)(f, inp, &len, reject_if_disagree);
    } else if (len)
	code = CONFREJ;			/* Reject all CI */
    else
	code = CONFACK;

    /* send the Ack, Nak or Rej to the peer */
    fsm_sdata(f, code, id, inp, len);

    if (code == CONFACK) {
	if (f->state == ACKRCVD) {
	    UNTIMEOUT(fsm_timeout, f);	/* Cancel timeout */
	    f->state = OPENED;
	    if (f->callbacks->up)
		(*f->callbacks->up)(f);	/* Inform upper layers (注意这个 up 函数 )*/
	} else
	    f->state = ACKSENT;
	f->nakloops = 0;

    } else {
	/* we sent CONFACK or CONFREJ */
	if (f->state != ACKRCVD)
	    f->state = REQSENT;
	if( code == CONFNAK )
	    ++f->nakloops;
    }
}
```

* `(*f->callbacks->up)(f)`

```c
static fsm_callbacks ipcp_callbacks = { /* IPCP callback routines */
    ipcp_resetci,		/* Reset our Configuration Information */
    ipcp_cilen,			/* Length of our Configuration Information */
    ipcp_addci,			/* Add our Configuration Information */
    ipcp_ackci,			/* ACK our Configuration Information */
    ipcp_nakci,			/* NAK our Configuration Information */
    ipcp_rejci,			/* Reject our Configuration Information */
    ipcp_reqci,			/* Request peer's Configuration Information */
    ipcp_up,			/* Called when fsm reaches OPENED state */
    ipcp_down,			/* Called when fsm leaves OPENED state */
    NULL,			/* Called when we want the lower layer up */
    ipcp_finished,		/* Called when we want the lower layer down */
    NULL,			/* Called when Protocol-Reject received */
    NULL,			/* Retransmission is necessary */
    NULL,			/* Called to handle protocol-specific codes */
    "IPCP"			/* String name of protocol */
};


static void
ipcp_up(f)
    fsm *f;
{
    u_int32_t mask;
    ipcp_options *ho = &ipcp_hisoptions[f->unit];
    ipcp_options *go = &ipcp_gotoptions[f->unit];
    ipcp_options *wo = &ipcp_wantoptions[f->unit];

    IPCPDEBUG(("ipcp: up"));

    /*
     * We must have a non-zero IP address for both ends of the link.
     */
    if (!ho->neg_addr && !ho->old_addrs)
	ho->hisaddr = wo->hisaddr;

    if (!(go->neg_addr || go->old_addrs) && (wo->neg_addr || wo->old_addrs)
	&& wo->ouraddr != 0) {
	error("Peer refused to agree to our IP address");
	ipcp_close(f->unit, "Refused our IP address");
	return;
    }
    if (go->ouraddr == 0) {
	error("Could not determine local IP address");
	ipcp_close(f->unit, "Could not determine local IP address");
	return;
    }
    if (ho->hisaddr == 0 && !noremoteip) {
	ho->hisaddr = htonl(0x0a404040 + ifunit);
	warn("Could not determine remote IP address: defaulting to %I",
	     ho->hisaddr);
    }
    script_setenv("IPLOCAL", ip_ntoa(go->ouraddr), 0);
    if (ho->hisaddr != 0)
	script_setenv("IPREMOTE", ip_ntoa(ho->hisaddr), 1);

    if (!go->req_dns1)
	    go->dnsaddr[0] = 0;
    if (!go->req_dns2)
	    go->dnsaddr[1] = 0;
    if (go->dnsaddr[0])
	script_setenv("DNS1", ip_ntoa(go->dnsaddr[0]), 0);
    if (go->dnsaddr[1])
	script_setenv("DNS2", ip_ntoa(go->dnsaddr[1]), 0);
    if (usepeerdns && (go->dnsaddr[0] || go->dnsaddr[1])) {
	script_setenv("USEPEERDNS", "1", 0);
	create_resolv(go->dnsaddr[0], go->dnsaddr[1]);
    }

    /*
     * Check that the peer is allowed to use the IP address it wants.
     */
    if (ho->hisaddr != 0 && !auth_ip_addr(f->unit, ho->hisaddr)) {
	error("Peer is not authorized to use remote address %I", ho->hisaddr);
	ipcp_close(f->unit, "Unauthorized remote IP address");
	return;
    }

    /* set tcp compression */
    sifvjcomp(f->unit, ho->neg_vj, ho->cflag, ho->maxslotindex);

    /*
     * If we are doing dial-on-demand, the interface is already
     * configured, so we put out any saved-up packets, then set the
     * interface to pass IP packets.
     */
    if (demand) {
	if (go->ouraddr != wo->ouraddr || ho->hisaddr != wo->hisaddr) {
	    ipcp_clear_addrs(f->unit, wo->ouraddr, wo->hisaddr);
	    if (go->ouraddr != wo->ouraddr) {
		warn("Local IP address changed to %I", go->ouraddr);
		script_setenv("OLDIPLOCAL", ip_ntoa(wo->ouraddr), 0);
		wo->ouraddr = go->ouraddr;
	    } else
		script_unsetenv("OLDIPLOCAL");
	    if (ho->hisaddr != wo->hisaddr && wo->hisaddr != 0) {
		warn("Remote IP address changed to %I", ho->hisaddr);
		script_setenv("OLDIPREMOTE", ip_ntoa(wo->hisaddr), 0);
		wo->hisaddr = ho->hisaddr;
	    } else
		script_unsetenv("OLDIPREMOTE");

	    /* Set the interface to the new addresses */
	    mask = GetMask(go->ouraddr);
	    if (!sifaddr(f->unit, go->ouraddr, ho->hisaddr, mask)) {
		if (debug)
		    warn("Interface configuration failed");
		ipcp_close(f->unit, "Interface configuration failed");
		return;
	    }

	    /* assign a default route through the interface if required */
	    if (ipcp_wantoptions[f->unit].default_route) 
		if (sifdefaultroute(f->unit, go->ouraddr, ho->hisaddr))
		    default_route_set[f->unit] = 1;

	    /* Make a proxy ARP entry if requested. */
	    if (ho->hisaddr != 0 && ipcp_wantoptions[f->unit].proxy_arp)
		if (sifproxyarp(f->unit, ho->hisaddr))
		    proxy_arp_set[f->unit] = 1;

	}
	demand_rexmit(PPP_IP);
	sifnpmode(f->unit, PPP_IP, NPMODE_PASS);

    } else {
	/*
	 * Set IP addresses and (if specified) netmask.
	 */
	mask = GetMask(go->ouraddr);

#if !(defined(SVR4) && (defined(SNI) || defined(__USLC__)))
	if (!sifaddr(f->unit, go->ouraddr, ho->hisaddr, mask)) {
	    if (debug)
		warn("Interface configuration failed");
	    ipcp_close(f->unit, "Interface configuration failed");
	    return;
	}
#endif

	/* run the pre-up script, if any, and wait for it to finish */
	ipcp_script(_PATH_IPPREUP, 1);

	/* bring the interface up for IP */
	if (!sifup(f->unit)) {
	    if (debug)
		warn("Interface failed to come up");
	    ipcp_close(f->unit, "Interface configuration failed");
	    return;
	}

#if (defined(SVR4) && (defined(SNI) || defined(__USLC__)))
	if (!sifaddr(f->unit, go->ouraddr, ho->hisaddr, mask)) {
	    if (debug)
		warn("Interface configuration failed");
	    ipcp_close(f->unit, "Interface configuration failed");
	    return;
	}
#endif
	sifnpmode(f->unit, PPP_IP, NPMODE_PASS);

	/* assign a default route through the interface if required */
	if (ipcp_wantoptions[f->unit].default_route) 
	    if (sifdefaultroute(f->unit, go->ouraddr, ho->hisaddr))
		default_route_set[f->unit] = 1;

	/* Make a proxy ARP entry if requested. */
	if (ho->hisaddr != 0 && ipcp_wantoptions[f->unit].proxy_arp)
	    if (sifproxyarp(f->unit, ho->hisaddr))
		proxy_arp_set[f->unit] = 1;

	ipcp_wantoptions[0].ouraddr = go->ouraddr;

	notice("local  IP address %I", go->ouraddr);
	if (ho->hisaddr != 0)
	    notice("remote IP address %I", ho->hisaddr);
	if (go->dnsaddr[0])
	    notice("primary   DNS address %I", go->dnsaddr[0]);
	if (go->dnsaddr[1])
	    notice("secondary DNS address %I", go->dnsaddr[1]);
    }

    reset_link_stats(f->unit);

    np_up(f->unit, PPP_IP);
    ipcp_is_up = 1;

    notify(ip_up_notifier, 0);
    if (ip_up_hook)
	ip_up_hook();

    /*
     * Execute the ip-up script, like this:
     *	/etc/ppp/ip-up interface tty speed local-IP remote-IP
     */
    if (ipcp_script_state == s_down && ipcp_script_pid == 0) {
	ipcp_script_state = s_up;
	ipcp_script(_PATH_IPUP, 0);
    }
}
```