TODO List
=============

#####[x]vIRQ vs pIRQ?#####

#####[x]Is bsp of SMP VCPU0? #####

#####[x]Where to bind evtchn to pIRQ?#####

pv drivers? modify the guest?

#####[x]where to trigger the evtchn of pIRQ?#####
#####[x] trap, event, interrupt?#####
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


#####[x] the meaning of "to bind"  #####
* Connecting the endpoints of a channel to a port 
* Assigning a VCPU for receiving events on a given channel  
* Setting the handler for a specified event  
 
#####[x]"PVHVM"  vs "PV-on-HVM" ?  #####
(“PVHVM” mode should not be confused with “PV-on-HVM” mode, which is a term sometimes used in the past for “fully virtualized with PV drivers”.) 
 
http://wiki.xen.org/wiki/Virtualization_Spectrum#Paravirtualizing_little_by_little:_PVHVM_mode   
http://wiki.xen.org/wiki/Xen_Linux_PV_on_HVM_drivers  
http://www.rackspace.com/knowledge_center/article/choosing-a-virtualization-mode-pv-versus-pvhvm  

#####About the implementation  #####
1.  Where to implement the map? In the function of bind(). But which bind(), there are a lot bind();   Maybe I shoud init map during creating a domain.
2.  How to implement a new mechanism to replace the hvm_set_callback_irq_level? And what's the old mechanism?

#####PVonHVM extensions bypass the emulated APICs for issuing events to guests.######
what does this mean?there are two means to handle interrupt:  
1. hardware --> xen --> evtchn --> APIC --> VM  
2. hardware --> xen -->evtchn --> VM


#####what does this mean? #####
Windows PV drivers currently have to multiplex all event channel processing onto a single interrupt which is registered with Xen using the HVM_PARAM_CALLBACK_IRQ parameter.  

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


1. 

	
#####GSI IRQ VECTOR
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



Code Annotation 
============












