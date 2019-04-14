# goal in lab3 you will implement the basic kernel facilities  

* get a protected user-mode environment running

* set up the data structures to keep track of user environments  

* create a single user environment, load a program image into it, and start it running 
>in the jos , environment and process are interexchangable，a abstraction load the programe

# Part A: User Environments and Exception Handling
in kern/env.c 中 kernel maintain three main global variables pertaining to environment  
```
struct Env *envs = NULL;		// All environments
struct Env *curenv = NULL;		// The current env
static struct Env *env_free_list;	// Free environment list
```
* in jos, jos mantain pointer to points a array of Env to get( representing all environments in sys  
environments at any given time. (NENV is a constant #define'd in inc/env.h.) Once it is allocated  
the envs array will contain a single instance of the Env data structure for each of the NENV possible environments  

* JOS keep track of inactive Env structure on the 'env_free_list'  
    this allows easys allocation and deallocation of environments,as added or removed from list  

* 'curenv' to keep track of the currently executing env,before the first environment is run, 'curenv'is initially set to NULL

## Env state defined in inc/env.h 

```struct Env {
	struct Trapframe env_tf;	// Saved registers while that environment is not running，defined in inc/trap.h,same as stack reserves。  
	struct Env *env_link;		// Next free Env，This is a link to the next Env on the env_free_list. env_free_list points to the first free environment on the list.
	envid_t env_id;			// Unique environment identifier
	envid_t env_parent_id;		// env_id of this env's parent
	enum EnvType env_type;		// Indicates special system environments used to distinguish special environments. For most environments, it will be ENV_TYPE_USER. We'll introduce a few more types for special system service environments in later labs. 
	unsigned env_status;		// Status of the environment
	uint32_t env_runs;		// Number of times environment has run

	// Address space
	pde_t *env_pgdir;		// Kernel virtual address of page dir
};
env_status:ENV_FREE   ENV_RUNNABLE ENV_RUNNING ENV_NOT_RUNNABLE  ENV_DYING
```

## Creating and Running Environments
because we do not have filesystem,we will set uo the kernel to load a staticc binary image that is embedded within the kernel itsel.  
JOS embeds this binay in kernel as a ELF executable image.
* start (kern/entry.S)
* i386_init (kern/init.c)
  * cons_init
  * mem_init
  * trap_init
  * env_create
  * env_run
    * env_pop_tf
    
 # Basics of Protected Control Transfer
    Exceptions and interrupts are both "protected control transfers," which cause the processor to switch from user to kernel mode (CPL=0)
    without giving the user-mode code any opportunity to interfere with the functioning of the kernel or other environments.
  
    In Intel's terminology, an interrupt is a protected control transfer that is caused by an asynchronous event usually external to the processor, 
    such as notification of external device I/O activity. 
  
    An exception, in contrast, is a protected control transfer caused synchronously by the currently running code, for example due to a divide by zero or an invalid memory access.
    In order to ensure that these protected control transfers are actually protected.
  
    mechanism is designed so that the code currently running when the interrupt or exception occurs does not get to choose arbitrarily where the kernel is entered or how.
    Instead, the processor ensures that the kernel can be entered only under carefully controlled conditions. On the x86, two mechanisms work together to provide this protection: 
    
## The Interrupt Descriptor Table IDT
  The CPU uses the vector as an index into the processor's interrupt descriptor table (IDT), which the kernel sets up in kernel-private memory, much like the GDT.   
  From the appropriate entry in this table the processor loads:   
  
  * the value to load into the instruction pointer (EIP) register, pointing to the kernel code designated to handle that type of exception.
  
  * the value to load into the code segment (CS) register, which includes in bits 0-1 the privilege level at which the exception handler is to run. (In JOS, all exceptions are handled in kernel mode, privilege level 0.)

## The Task State Segment TSS
  save old process state (like routine call stack protect the old call's regsiter) But this save area for the old processor state must in turn be protected from unprivileged user-mode code;  
  otherwise buggy or malicious user code could compromise the kernel.  
  JOS only uses it to define the kernel stack that the processor should switch to  
  when it transfers from user to kernel mode. Since "kernel mode" in JOS is privilege level 0 on the x86, the processor uses the ESP0 and SS0 fields of the TSS to define the kernel stack when entering kernel mode. JOS doesn't use any other TSS fields. 


```
 mem_init() 
 env_init()
 env_setup_vm()
 region_alloc()
 load_icode()
 env_create()
 env_run()

```
