# Obsolete
This Readme is obsolete due to a new kernel version.

# Installation

Copy this files into your linux kernel.

For new kernel:
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

# Modifying kernel by hand
If you have trouble to build your kernel with files from this repository, you can try to modify vmx and svm files in your kernel by yourself.
## Changes for Intel users
Open vmx.c in text editor.

First you need to find _setup_vmcs_config_ function (about 2500 lines) and add this line:
```c
    CPU_BASED_CR3_LOAD_EXITING |
    CPU_BASED_CR3_STORE_EXITING |
    CPU_BASED_UNCOND_IO_EXITING |
    CPU_BASED_MOV_DR_EXITING |
    CPU_BASED_USE_TSC_OFFSETTING |
    CPU_BASED_MWAIT_EXITING |
    CPU_BASED_MONITOR_EXITING |
    CPU_BASED_INVLPG_EXITING |
    CPU_BASED_RDPMC_EXITING | 	
    CPU_BASED_RDTSC_EXITING; //added line
```

Next you need to find an array of pointers to handlers initialization called _static int (*kvm_vmx_exit_handlers[])(struct kvm_vcpu *vcpu)_ and add the line to the end:
```c
    	[EXIT_REASON_PREEMPTION_TIMER]	= handle_preemption_timer,
	[EXIT_REASON_ENCLS]		= handle_encls,
	[EXIT_REASON_BUS_LOCK]		= handle_bus_lock_vmexit,
    	[EXIT_REASON_RDTSC]		= handle_rdtsc, //added line
};
```

Then add this code before _static int (*kvm_vmx_exit_handlers[])(struct kvm_vcpu *vcpu)_:
```c
static u32 print_once = 1;

static int handle_rdtsc(struct kvm_vcpu *vcpu) 
{ 
	static u64 rdtsc_fake = 0;
	static u64 rdtsc_prev = 0;
	u64 rdtsc_real = rdtsc();

	if(print_once)
	{
		printk("[handle_rdtsc] fake rdtsc vmx function is working\n");
		print_once = 0;
		rdtsc_fake = rdtsc_real;
	}

	if(rdtsc_prev != 0)
	{
		if(rdtsc_real > rdtsc_prev)
		{
			u64 diff = rdtsc_real - rdtsc_prev;
			u64 fake_diff =  diff / 16; // if you have 4.2Ghz on your vm, change 16 to 20 
			rdtsc_fake += fake_diff;
		}
	}
	if(rdtsc_fake > rdtsc_real)
	{
		rdtsc_fake = rdtsc_real;
	}
	rdtsc_prev = rdtsc_real;
    	vcpu->arch.regs[VCPU_REGS_RAX] = rdtsc_fake & -1u;
    	vcpu->arch.regs[VCPU_REGS_RDX] = (rdtsc_fake >> 32) & -1u;  

    	return skip_emulated_instruction(vcpu);
}
```

## Changes for AMD users
Open svm.c in text editor.

Find _static int (*const svm_exit_handlers[])(struct kvm_vcpu *vcpu)_ (~2700 lines) and add this line:
```c
	[SVM_EXIT_AVIC_INCOMPLETE_IPI]		= avic_incomplete_ipi_interception,
	[SVM_EXIT_AVIC_UNACCELERATED_ACCESS]	= avic_unaccelerated_access_interception,
	[SVM_EXIT_VMGEXIT]			= sev_handle_vmgexit,
	[SVM_EXIT_RDTSC]			= handle_rdtsc_interception, //added line
};
```

Next find _static void init_vmcb(struct kvm_vcpu *vcpu)_ (~1000 lines) and add this line:
```c
	svm_set_intercept(svm, INTERCEPT_RDPRU);
	svm_set_intercept(svm, INTERCEPT_RSM);
	svm_set_intercept(svm, INTERCEPT_RDTSC); //added line
```

After that add this code on top of _static int (*const svm_exit_handlers[])(struct kvm_vcpu *vcpu)_:
```c
static u32 print_once = 1;

static int handle_rdtsc_interception(struct kvm_vcpu *vcpu) 
{
    	static u64 rdtsc_fake = 0;
	static u64 rdtsc_prev = 0;
	u64 rdtsc_real = rdtsc();

	if(print_once)
	{
		printk("[handle_rdtsc] fake rdtsc svm function is working\n");
		print_once = 0;
		rdtsc_fake = rdtsc_real;
	}

	if(rdtsc_prev != 0)
	{
		if(rdtsc_real > rdtsc_prev)
		{
			u64 diff = rdtsc_real - rdtsc_prev;
			u64 fake_diff =  diff / 20; // if you have 3.2Ghz on your vm, change 20 to 16
			rdtsc_fake += fake_diff;
		}
	}
	if(rdtsc_fake > rdtsc_real)
	{
		rdtsc_fake = rdtsc_real;
	}
	rdtsc_prev = rdtsc_real;

	vcpu->arch.regs[VCPU_REGS_RAX] = rdtsc_fake & -1u;
    	vcpu->arch.regs[VCPU_REGS_RDX] = (rdtsc_fake >> 32) & -1u;

    	return skip_emulated_instruction(vcpu);
}
```

# Kernel building
* Clone the kernel from [official](https://github.com/torvalds/linux) repository or from repository of your os.
* Make changes for your cpu
* Install the requirements for kernel building
```bash
sudo apt-get install build-essential libncurses-dev bison flex libssl-dev libelf-dev
```
* Copy your current config into your kernel folder
```bash
cp -v /boot/config-$(uname -r) .config
```
* Make kernel
```bash
make -j $(nproc)
```
* Install kernel
```bash
sudo make INSTALL_MOD_STRIP=1 modules_install && sudo make install
```
* Reboot your system
* Check if a new kernel was installed
```bash
uname -a
```
