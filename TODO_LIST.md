#TODO LIST
####[] add a hypercall
bind a particular event channel to a particular cpu and vector.
####[] modify evtchn structure
add some extra field. bind a particular event channel to a particular cpu and vector.
<!--lang:c-->
	struct evtchn
	{
	#define ECS_FREE         0 /* Channel is available for use.                  */
	#define ECS_RESERVED     1 /* Channel is reserved.                           */
	#define ECS_UNBOUND      2 /* Channel is waiting to bind to a remote domain. */
	#define ECS_INTERDOMAIN  3 /* Channel is bound to another domain.            */
	#define ECS_PIRQ         4 /* Channel is bound to a physical IRQ line.       */
	#define ECS_VIRQ         5 /* Channel is bound to a virtual IRQ line.        */
	#define ECS_IPI          6 /* Channel is bound to a virtual IPI line.        */
	    u8  state;             /* ECS_* */
	    u8  xen_consumer;      /* Consumer in Xen, if any? (0 = send to guest) */
	    u16 notify_vcpu_id;    /* VCPU for local delivery notification */
	    u32 port;
	    union {
	        struct {
	            domid_t remote_domid;
	        } unbound;     /* state == ECS_UNBOUND */
	        struct {
	            u16            remote_port;
	            struct domain *remote_dom;
	        } interdomain; /* state == ECS_INTERDOMAIN */
	        struct {
	            u16            irq;
	            u16            next_port;
	            u16            prev_port;
	        } pirq;        /* state == ECS_PIRQ */
	        u16 virq;      /* state == ECS_VIRQ */
	    } u;
	    u8 priority;
	    u8 pending:1;
	    u16 last_vcpu_id;
	    u8 last_priority;
	#ifdef FLASK_ENABLE
	    void *ssid;
	#endif
	};

####[] struct evtchn. notify_vcpu_id?
what does that mean?
####[]modify hvm\_assert\_evtchn_irq()
####[] how to test?




#Question
####[x]The function flow of hvm\_set\_callback\_irq\_level()
    hvm_set_callback_irq_leve
	hvm_assert_evtchn-irq
	vcpu_mark_events_pending
	evtchn_2l_set_pending (2-level ABI)
	.set_pending = evtchn_2l_set_pending
	evtchn_2l_init
	evtchn_init
	domain_create

####[x]what is the difference between callback\_vector, callback\_gsi and callback\_pci\_intx?
callback\_vector is for PVHVM.    
callback\_gsi is for emulated APIC. When the guest os isn't running and an interrupt comes, so VCPU should have bitmap reprenting the pending interrupt. When VMENTRY, the related field of VMCS whill be set to inject the interrupt into guest os.


####[x]How does APIC work in Xen?


####[]How does callback\_vector work? 
It is for PVHVM, bypass the vAPIC. And it should modify the guest os.  
for in guest os  
<!--lang:c-->
	1667 #ifdef CONFIG_XEN_PVHVM
	1668 /* Vector callbacks are better than PCI interrupts to receive event
	1669  * channel notifications because we can receive vector callbacks on any
	1670  * vcpu and we don't need PCI support or APIC interactions. */
	1671 void xen_callback_vector(void)
	1672 {
	1673         int rc;
	1674         uint64_t callback_via;
	1675         if (xen_have_vector_callback) {
	1676                 callback_via = HVM_CALLBACK_VECTOR(XEN_HVM_EVTCHN_CALLBACK);
	1677                 rc = xen_set_callback_via(callback_via);
	1678                 if (rc) {
	1679                         printk(KERN_ERR "Request for Xen HVM callback vector"
	1680                                         " failed.\n");
	1681                         xen_have_vector_callback = 0;
	1682                         return;
	1683                 }
	1684                 printk(KERN_INFO "Xen HVM callback vector for event delivery is "
	1685                                 "enabled\n");
	1686                 /* in the restore case the vector has already been allocated */
	1687                 if (!test_bit(XEN_HVM_EVTCHN_CALLBACK, used_vectors))
	1688                         alloc_intr_gate(XEN_HVM_EVTCHN_CALLBACK, xen_hvm_callback_vector);
	1689         }
	1690 }
	1691 #else
	1692 void xen_callback_vector(void) {}
	1693 #endif

In xen,

	void hvm_assert_evtchn_irq(struct vcpu *v)
	{
	    if ( unlikely(in_irq() || !local_irq_is_enabled()) )
	    {
	        tasklet_schedule(&v->arch.hvm_vcpu.assert_evtchn_irq_tasklet);
	        return;
	    }
	
	    if ( is_hvm_pv_evtchn_vcpu(v) )
	        vcpu_kick(v);
	    else if ( v->vcpu_id == 0 )
	        hvm_set_callback_irq_level(v);
	}

####[?]How APIC work with evtchn ?
it  has nothing to do with evtchn
####[?]How does the windows PV driver work[]?


####[?] what are the application scenarios?
Is it uesed to improve windows pv drivers?







































