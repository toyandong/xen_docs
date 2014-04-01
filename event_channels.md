TODO List
=============

####[x]vIRQ vs pIRQ?####
<!--lang:c++-->
	/*
	 * EVTCHNOP_bind_virq: Bind a local event channel to VIRQ <irq> on specified
	 * vcpu.
	 * NOTES:
	 *  1. Virtual IRQs are classified as per-vcpu or global. See the VIRQ list
	 *     in xen.h for the classification of each VIRQ.
	 *  2. Global VIRQs must be allocated on VCPU0 but can subsequently be
	 *     re-bound via EVTCHNOP_bind_vcpu.
	 *  3. Per-vcpu VIRQs may be bound to at most one event channel per vcpu.
	 *     The allocated event channel is bound to the specified vcpu and the
	 *     binding cannot be changed.
	 */
	struct evtchn_bind_virq {
	    /* IN parameters. */
	    uint32_t virq; /* enum virq */
	    uint32_t vcpu;
	    /* OUT parameters. */
	    evtchn_port_t port;
	};
	typedef struct evtchn_bind_virq evtchn_bind_virq_t;
	
	/*
	 * EVTCHNOP_bind_pirq: Bind a local event channel to a real IRQ (PIRQ <irq>).
	 * NOTES:
	 *  1. A physical IRQ may be bound to at most one event channel per domain.
	 *  2. Only a sufficiently-privileged domain may bind to a physical IRQ.
	 */
	struct evtchn_bind_pirq {
	    /* IN parameters. */
	    uint32_t pirq;
	#define BIND_PIRQ__WILL_SHARE 1
	    uint32_t flags; /* BIND_PIRQ__* */
	    /* OUT parameters. */
	    evtchn_port_t port;
	};

####[x]Is bsp of SMP VCPU0? ####

####[x]Where to bind evtchn to pIRQ?####

pv drivers? modify the guest?

####[x]where to trigger the evtchn of pIRQ?####
####[x] trap, event, interrupt?####
During startup the guest OS installs two handlers (event and failsafe)
via the *HYPERVISOR\_set\_callbacks* hypercall:  
1. The *event\_callback* is the handler to be called to notify an event to
the guest OS  
2. The *failsafe\_callback* is used when a fault occurs when using the
event callback  
3. The guest OS can install a handler for a physical IRQ through the
*HYPERVISOR\_event\_channel\_op* hypercall, specifying as operation
*EVTCHNOP\_bind\_pirq*.  


When an interrupt occurs control passes to the Xen
common_interrupt routine, that calls the Xen do_IRQ function.  
*do\_IRQ*  
Checks who has the responsibility to handle the interrupt:
The VMM: the interrupt is handled internally by the VMM
One ore more guest OS: it calls __do_IRQ_guest function  
*\_\_do\_IRQ\_guest*:  
For each domain that has a binding to the IRQ sets to 1 the pending
flag of the event channel via send_guest_pirq

The entry point in Linux is the hypervisor\_callback function (is
the event callback handler installed at startup), that calls 
evtchn\_do\_upcall.    
*evtchn_do_upcall*:  
1. Checks for pending events  
2. Resets to zero the pending flag  
3. Uses the evtchn_to_irq array to identify the IRQ binding for
the event channel  
4. Calls Linux do_IRQ interrupt handler function
####what is callback() ?####
the function in guest to handle the irq;


####[x] the meaning of "to bind"  ####
* Connecting the endpoints of a channel to a port 
* Assigning a VCPU for receiving events on a given channel  
* Setting the handler for a specified event  
 
####[x]"PVHVM"  vs "PV-on-HVM" ?  ####
(“PVHVM” mode should not be confused with “PV-on-HVM” mode, which is a term sometimes used in the past for “fully virtualized with PV drivers”.) 
 
http://wiki.xen.org/wiki/Virtualization_Spectrum#Paravirtualizing_little_by_little:_PVHVM_mode   
http://wiki.xen.org/wiki/Xen_Linux_PV_on_HVM_drivers  
http://www.rackspace.com/knowledge_center/article/choosing-a-virtualization-mode-pv-versus-pvhvm  
http://wiki.xen.org/wiki/PV_on_HVM  



####PVonHVM extensions bypass the emulated APICs for issuing events to guests.#####
what does this mean?there are two means to handle interrupt:  
1. hardware --> xen --> evtchn --> APIC --> VM  
2. hardware --> xen -->evtchn --> VM


####what does this exactly mean? ####
Windows PV drivers currently have to multiplex all event channel processing onto a single interrupt which is registered with Xen using the *HVM\_PARAM\_CALLBACK\_IRQ* parameter.  

<!--lang:c++-->
	/*
	 * Parameter space for HVMOP_{set,get}_param.
	 * How should CPU0 event-channel notifications be delivered?
	 * val[63:56] == 0: val[55:0] is a delivery GSI (Global System Interrupt).
	 * val[63:56] == 1: val[55:0] is a delivery PCI INTx line, as follows:
	 *                  Domain = val[47:32], Bus  = val[31:16],
	 *                  DevFn  = val[15: 8], IntX = val[ 1: 0]
	 * val[63:56] == 2: val[7:0] is a vector number, check for
	 *                  XENFEAT_hvm_callback_vector to know if this delivery
	 *                  method is available.
	 * If val == 0 then CPU0 event-channel notifications are not delivered.
	 */
	#define HVM_PARAM_CALLBACK_IRQ 0  


1. N:1?

	
####GSI IRQ VECTOR####
Xen get mp_irqs[] from MADT,   the mp_irqs[]  struct is as below
<!--lang:c++-->
	struct mpc_config_intsrc
	{
		unsigned char mpc_type;
		unsigned char mpc_irqtype;
		unsigned short mpc_irqflag;
		unsigned char mpc_srcbus;  #SOURCE BUS ID
		unsigned char mpc_srcbusirq;   # SOURCE BUS IRQ
		unsigned char mpc_dstapic;
		unsigned char mpc_dstirq;    #destination IOAPIC INTN
	};

 
Can I get the conclusion as blow:  
1. the mp_irqs[]  acts as a map between IRQ and IOAPIC PIN, and we can get  GSI from mpc_config_intsrc.mpc_dstirq  (GSI=GSI base + pin).   
2. For PCI IRQs, the IRQ equal to the GSI.  
3. But for ISA IRQs, the two maybe not match, because of the ISO(interrupt source overrides).  
4. And the map between IRQ and GSI  is decided by hardware, not by xen.Because the map info was get from MADT.  
5. ps :  another question, can I rename "apic_pin_2_gsi_irq"  with  "apic_pin_2_gsi" for they are the same semantics.  ???

<!--lang:c-->
	     /*
	445          * A PV guest has no concept of a GSI (since it has no ACPI
	446          * nor access to/knowledge of the physical APICs). Therefore
	447          * all IRQs are dynamically allocated from the entire IRQ
	448          * space.
	449          */


#####what is regs->entry_vector and error_code ?####
<!--lang:c-->
	void do_IRQ(struct cpu_user_regs *regs)
	{
	    struct irqaction *action;
	    uint32_t          tsc_in;
	    struct irq_desc  *desc;
	    unsigned int      vector = (u8)regs->entry_vector;

entry_vector:  is it the interrupt vector which is coming?
<!--lang:c-->
	struct cpu_user_regs {
	    uint64_t r15;
	    uint64_t r14;
	    uint64_t r13;
	    uint64_t r12;
	    __DECL_REG(bp);
	    __DECL_REG(bx);
	    uint64_t r11;
	    uint64_t r10;
	    uint64_t r9;
	    uint64_t r8;
	    __DECL_REG(ax);
	    __DECL_REG(cx);
	    __DECL_REG(dx);
	    __DECL_REG(si);
	    __DECL_REG(di);
	    uint32_t error_code;    /* private */
	    uint32_t entry_vector;  /* private */
	    __DECL_REG(ip);
	    uint16_t cs, _pad0[1];
	    uint8_t  saved_upcall_mask;
	    uint8_t  _pad1[3];
	    __DECL_REG(flags);      /* rflags.IF == !saved_upcall_mask */
	    __DECL_REG(sp);
	    uint16_t ss, _pad2[3];
	    uint16_t es, _pad3[3];
	    uint16_t ds, _pad4[3];
	    uint16_t fs, _pad5[3]; /* Non-zero => takes precedence over fs_base.     */
	    uint16_t gs, _pad6[3]; /* Non-zero => takes precedence over gs_base_usr. */
	};

####two   evtchn\_to\_irq?####

one in guest, one in xen?
<!--lang:c-->
	1695 void __init xen_init_IRQ(void)
	1696 {
	1697         int i;
	1698 
	1699         evtchn_to_irq = kcalloc(NR_EVENT_CHANNELS, sizeof(*evtchn_to_irq),
	1700                                     GFP_KERNEL);
	1701         BUG_ON(!evtchn_to_irq);
	1702         for (i = 0; i < NR_EVENT_CHANNELS; i++)
	1703                 evtchn_to_irq[i] = -1;
	1704 
	1705         init_evtchn_cpu_bindings();
	1706 
	1707         /* No event channels are 'live' right now. */
	1708         for (i = 0; i < NR_EVENT_CHANNELS; i++)
	1709                 mask_evtchn(i);
	1710 
	1711         if (xen_hvm_domain()) {
	1712                 xen_callback_vector();
	1713                 native_init_IRQ();
	1714                 /* pci_xen_hvm_init must be called after native_init_IRQ so that
	1715                  * __acpi_register_gsi can point at the right function */
	1716                 pci_xen_hvm_init();
	1717         } else {
	1718                 irq_ctx_init(smp_processor_id());
	1719                 if (xen_initial_domain())
	1720                         pci_xen_initial_domain();
	1721         }
	1722 }

####xen\_callback\_vector is for PVHVM to bypass PCI otr APIC####
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


    union {
        enum {
            HVMIRQ_callback_none,
            HVMIRQ_callback_gsi,
            HVMIRQ_callback_pci_intx,
            HVMIRQ_callback_vector
        } callback_via_type;
    };
    union {
        uint32_t gsi;
        struct { uint8_t dev, intx; } pci;
        uint32_t vector;
    } callback_via;
	
	
	 76 /*
	 77  * Packed IRQ information:
	 78  * type - enum xen_irq_type
	 79  * event channel - irq->event channel mapping
	 80  * cpu - cpu this event channel is bound to
	 81  * index - type-specific information:
	 82  *    PIRQ - vector, with MSB being "needs EIO", or physical IRQ of the HVM
	 83  *           guest, or GSI (real passthrough IRQ) of the device.
	 84  *    VIRQ - virq number
	 85  *    IPI - IPI vector
	 86  *    EVTCHN -
	 87  */
	 88 struct irq_info {
	 89         struct list_head list;
	 90         enum xen_irq_type type; /* type */
	 91         unsigned irq;
	 92         unsigned short evtchn;  /* event channel */
	 93         unsigned short cpu;     /* cpu bound */
	 94 
	 95         union {
	 96                 unsigned short virq;
	 97                 enum ipi_vector ipi;
	 98                 struct {
	 99                         unsigned short pirq;
	100                         unsigned short gsi;
	101                         unsigned char vector;
	102                         unsigned char flags;
	103                         uint16_t domid;
	104                 } pirq;
	105         } u;
	106 };

About the implementation
============
1.  Where to implement the map? In the function of bind(). But which bind(), there are a lot bind();   Maybe I shoud init map during creating a domain.
2.  can it solve the "multiple VIFs" problem?
3.  How to implement a new mechanism to replace the hvm_set_callback_irq_level? And what's the old mechanism?


Code Annotation 
============












