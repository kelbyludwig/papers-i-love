# Hacking the Linux Kernel Network Stack

* [Paper](http://phrack.org/issues/61/13.html)

## 2. The various Netfilter hooks and their uses
* At a super abstract level, Netfilter is able to hook key positions in a packet's journey through the Linux kernel.
* Netfilter provides 5 IPv4 Hooks:

| Hook               | Called                                                    |
|--------------------|-----------------------------------------------------------|
| NF_IP_PRE_ROUTING  | After sanity checks, before routing decisions.            |
| NF_IP_LOCAL_IN     | After routing decisions, if packet is for this host.      |
| NF_IP_FORWARD      | If the packet is destined for another interface.          |
| NF_IP_LOCAL_OUT    | For packets coming from local processes on their way out. |
| NF_IP_POST_ROUTING | Just before outbound packets "hit the wire".              |

* After a Netfilter hook is called, it must return a predefined Netfilter return code.

| Return Code | Meaning                        |
|-------------|--------------------------------|
| NF_DROP     | Discard the packet.            |
| NF_ACCEPT   | Keep the packet.               |
| NF_STOLEN   | Forget about the packet.       |
| NF_QUEUE    | Queue packet for userspace.    |
| NF_REPEAT   | Call this hook function again. |

* NF_STOLEN is cool. It tells Netfilter that the hooks will take care of this packet, and to not worry about further processing.
* The author omits a description of NF_QUEUE.

## 3. Registering and un-registering Netfilter hooks

```
/* Sample code to install a Netfilter hook function that will
 * drop all incoming packets. */

#define __KERNEL__
#define MODULE

#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/netfilter.h>
#include <linux/netfilter_ipv4.h>

/* This is the structure we shall use to register our function */
static struct nf_hook_ops nfho;

/* This is the hook function itself */
unsigned int hook_func(unsigned int hooknum,
                       struct sk_buff **skb,
                       const struct net_device *in,
                       const struct net_device *out,
                       int (*okfn)(struct sk_buff *))
{
    return NF_DROP;           /* Drop ALL packets */
}

/* Initialisation routine */
int init_module()
{
    /* Fill in our hook structure */
    nfho.hook = hook_func;         /* Handler function */
    nfho.hooknum  = NF_IP_PRE_ROUTING; /* First hook for IPv4 */
    nfho.pf       = PF_INET;
    nfho.priority = NF_IP_PRI_FIRST;   /* Make our function first */

    nf_register_hook(&nfho);
    
    return 0;
}
    
/* Cleanup routine */
void cleanup_module()
{
    nf_unregister_hook(&nfho);
}
```

## Sidetracked

I got distracted and didn't finished my notes. I wrote [this](https://kel.bz/post/netfilter/) instead.
