# Installation

Copy this files into your linux kernel.
* vmx.c is going to /arch/x86/kvm/vmx
* svm.c is going to /arch/x86/kvm/svm

Don't forget to disable rdtsc in your qemu xml config:

Add rdtscp=off to your qemu:arg, so it will look similar to this:

<qemu:arg value="-cpu"/>
<qemu:arg value="host,rdtscp=off,hv_time,kvm=off,hv_vendor_id=null,-hypervisor"/>
