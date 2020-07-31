# Installation

Copy this files into your linux kernel.
* vmx.c is going to /arch/x86/kvm/vmx
* svm.c is going to /arch/x86/kvm/svm

Don't forget to disable rdtscp in your qemu xml config:

Add rdtscp=off to your qemu:arg, so it will look similar to this:

<qemu:arg value="-cpu"/>

<qemu:arg value="host,rdtscp=off,hv_time,kvm=off,hv_vendor_id=null,-hypervisor"/>

# Changing timer

You can play with ticks if you want to.

For Intel users:

* Open vmx.c in text editor
* Find handle_rdtsc function
* Change **u64 fake_diff =  diff / 16;**
* 16 is a divider of actual difference in timestamp, you can increase and decrease it

For AMD users:

* Open svm.c in text editor
* Find handle_rdtsc_interception function
* Change **u64 fake_diff =  diff / 20;**
* 20 is a divider of actual difference in timestamp, you can increase and decrease it 
