TODO List
=============

#####[x]vIRQ vs pIRQ?#####

#####[x]Is bsp of SMP VCPU0? #####

#####[x]Where to bind evtchn to pIRQ?#####

pv drivers? modify the guest?

#####[x]where to trigger the evtchn of pIRQ?#####
#####[x] trap, event, interrupt?#####

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











Code Annotation 
============












