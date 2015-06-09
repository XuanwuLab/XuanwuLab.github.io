title: Research report on using JIT to trigger RowHammer
date: 2015-06-09 10:42:07
tags:
---
# 0x00 Overview
In a post published on Google Project Zero Blog, researchers explain that the RowHammer technique works by repeatedly accessing memory rows in DRAM to flip bits in adjacent rows. People worry about it because it need to update BIOS to fix this problem, it's hard to solve. However it need to run the customized asm code on target machine to trigger RowHammer, so RowHammer is not easy to be used to attack.  
We have an idea that try to use script language to trigger RowHammer. If it works, RowHammer will be more dangerous. In order to verify our idea, we analyzed the Java Hotspot, Chrome V8, .NET CoreCLR and Firefox SpiderMonfey.  
Finally we didn't find useful attack vector. Some of them don't generate instructions needed to trigger RowHammer, some of them cannot trigger RowHammer due to small amount and slow speed of the instruction generation, some of them need environment modified to trigger so that we cannot use them to attack directly.   

# 0x01 RowHammer
In this section, we briefly review the root cause of RowHammer, how to trigger it and the limitation we will face with when trying to use it to attack.   

## 1.1 What`s RowHammer?

RowHammer is a problem with some DDR3 in which repeatedly accessing a row of memory can cause bit flips in adjacent rows. As shown in Figure 1.1(a), DRAM comprises a two-dimensional array of DRAM cells. As shown in Figure 1.1(b), one cell consists of a capacitor and an access-transistor. The access-transistor connects to wordline and the capacitor stores the data. The data in a row can be accessible only if the wordline is in high voltage. The data in the row is transferred to row-buffer. When a wordline`s voltage toggle on and off repeatedly, some cells on nearby rows lose voltage. If it cannot retrain charge for even 64ms, this will lead to lose data.

Figure 1.2 shows a 2GB rank, whose 256K rows are vertically partitioned into eight banks of 32K rows, where each row is 8KB (64Kb). Each bank has its own dedicated row-buffer. Notice that accessing the rows in different bank is not able to trigger the RowHammer.
{% asset_img img1.png [Figure 1.1] %}
                     Figure 1.1
{% asset_img img2.png [Figure 1.2] %}
                     Figure 1.2

## 1.2 Trigger RowHammer
Google Project Zero gives the snippet of code that can cause RowHammer. 
 
Address of X and Y is very important if you want to trigger RowHammer. X and Y must point to the same bank but different rows. Because each bank has its row-buffer and if we access the same row the wordline will not toggle on and off repeatedly. 

This snippet of the code is available to trigger the RowHammer. But it isn`t the only one we can use. Notice that any code that can toggle the wordline can be used to trigger the RowHammer.

## 1.3 Instruction needed
In order to toggle the wordline on and off repeatedly, we have to deal with CPU Cache, if the address we want to access is already in Cache, the wordline will not be set to high voltage. 

| Instruction |	Action                           |
| ----------- |:--------------------------------:|
| CLFLUSH     | Flush the address form cache     |
| PREFETCH    | Prefetch the data into the cache |
| MOVNT*	  | Non-temporal memory access       |

The instructions above can be used to access some address and bypass the cache. So using these instructions we can toggle the wordline and trigger the RowHammer.  

# 0x02 Using Script to trigger RowHammer
	The POC that Google Project Zero provides uses ASM code, it can be used to verify whether your devices are vulnerable. We know that most of the script languages have JIT compiler. If we can use the script to control the JIT compiler trigger RowHammer, the things will get worse. We research the Java Hotspot, Chrome V8, .Net CoreCLR and Firefox SpiderMonkey to verify the feasibility of our idea. 
## 2.1 Java Hotspot 
The Java Hotspot Virtual Machine is a core component of the Java SE platform. It implements the Java Virtual Machine Specification. As the Java bytecode execution engine, it also includes dynamic compilers that adaptively compile Java bytecodes into optimized machine instructions. Hotspot is the Stack based virtual machine. The bytecodes are stored in the class file. As the input to hotspot, it is user controllable. What we care about is whether we can customize class file to make the Java Hotspot trigger the RowHammer.
When Java Hotspot runs Java bytecode, it continually analyzes the program`s performance for ¡°hot spot¡± which are frequently or repeatedly executed. The JIT compiler would be used to compile these codes.  
The default interpreter that comes with the Hotspot is the so called ¡°Template Interpreter¡±. A second interpreter existed beside the template interpreter is a C++ interpreter its main interpreter loop is implemented in C++. The JIT compiler in Java Hotspot has three implementation, the client compiler (C1 Compiler), the server compiler (C2 Compiler) and the Shark Compiler ( LLVM based Compiler).
   
   
Figure 2.1
### 2.1.1 Interpreter trigger RowHammer?
a)	The working mechanism of template interpreter
The template interpreter is basically created at runtime from a kind of assembler templates which are translated into real machine code. It interprets a Java program by bytecode. When the interpreter gets a new bytecode, the corresponding native machine code would be called.
In order to interpret the Java program, Java Hotspot generates a lot of code stub when it starts such as StubRoutines::call_stub and StubRoutines::catch_exception. The command ¡°java ¨CXX:+PrintInterpterter¡± can be used to show the code stub that could be called in the interpret process.  
	Notice that the native machine code stub is a little big. For example, the code size of ¡°invokevitual¡± is 352 bytes, and the code size of ¡°putstatic¡± is 512 bytes. 

b)	Whether the interpreter can trigger RowHammer?
Whether we can customize the class file to make the Interpreter generate the machine code that we need to trigger the RowHammer. 
After analysis, we could not find the instructions such as prefetch, clflush and movnt* in the machine codes that generated in the interpreter. So we can`t use the template interpreter to trigger the RowHammer.   

### 2.1.2 JIT Compiler trigger RowHammer?
a)	The working mechanism of C1 Compiler
	C1 Compiler is a fast, lightly optimizing bytecode compiler. It performs some value numbering, inlining, and class analysis. It uses a simple CFG-oriented SSA ¡°high¡± IR, a machine-oriented ¡°low¡± IR, a linear scan register allocation, and a template-style code generator.
	The compiler is asynchronous, the ¡°CompilerThread¡± thread in Hotspot compile the method that needed to be compiled. The command ¡°-XX: +CompileThreshold¡± can be used to set the number of method invocations before compiling.
	 
Figure 2.2

In the source code of the hotspot£¬C1 Compiler has such phases:
typedef enum {
  _t_compile,
  _t_setup,
  _t_optimizeIR,
  _t_buildIR,
  _t_emit_lir,
  _t_linearScan,
  _t_lirGeneration,
  _t_lir_schedule,
  _t_codeemit,
  _t_codeinstall,
  max_phase_timers
} TimerName;

C1 Compiler can be briefly described as Figure 2.2 shows:  
1)	Build HIR
C1 Compiler iterates the Java bytecodes in the class file and translates it into CFG (Control Flow Graph). The basic block of the CFG uses SSA to represent the instructions. HIR is a ¡°high¡± IR far from machine code.
2)	Emit LIR
Iterate the basic blocks in the CFG, and iterate the instructions in the basic block. Translate the HIR to LIR. LIR is a ¡°low¡± IR which is close to machine code. 
3)	Register allocation
LIR uses many virtual register, in this phase, the compiler need to allocate the real register. The C1 Compiler uses the linear scan to allocate the register.
4)	Machine code generate
This is the phase to emit code, to genrate the real code. It iterates the instructions in the LIR to generate the machine code. The compile uses the LIR_Assembler class to finish this job, just as blow:

  LIR_Assembler lir_asm(this);
  lir_asm.emit_code(hir()->code());

The compiler iterates the LIR_list, invoke each instruction`s emit code. All instructions are the sub class of the LIR_Op. so they have ¡°emit_code¡± method.
    op->emit_code(this);

Let`s see LIR_Op1£¬its ¡°emit_code¡± method is:

void LIR_Op1::emit_code(LIR_Assembler* masm) {    //emit_code
  masm->emit_op1(this);
}

If the Operand of the LIR_Op1 is ¡°lir_prefetchr¡±

    case lir_prefetchr:
      prefetchr(op->in_opr());
      break;

Then it will invoke the prefetchr function, it is platform-dependent, in x86, the code is in assembler_x86.cpp
void Assembler::prefetchr(Address src) {
  assert(VM_Version::supports_3dnow_prefetch(), "must support");
  InstructionMark im(this);
  prefetch_prefix(src);
  emit_byte(0x0D);
  emit_operand(rax, src); // 0, src
}

At last the compiler will generate the real executable machine code.

b)	Whether the C1 Compiler can trigger RowHammer
Whether the C1 Compiler can trigger RowHammer or not? We actually find the prefetch instruction in X86, it means we have hope to trigger RowHammer.
From bottom to top, if we want to get prefetch instruction, we need the LIR_Op1, and its openrand is lir_prefetchr or lir_prefetchw in LIR. To achieve this, we need to invoke the GraphBuilder::append_unsafe_prefetch function in HIR. The function is called by GraphBuilder::try_inline_instrinsics function. As last we found if we invoke the prefetch* method in sun.misc.Unsafe, we can get it. So the Hotspot does support prefetch. It treats Unsafe.prefetchRead() and Unsafe.prefetchWrite() methods as intrinsics. The method would generate prefetch instruction in the machine code. But unfortunately, sun.misc.Unsafe in rt.jar dose not declare such methods. We have to modify rt.jar to trigger that. What a pity!
	
	In Hotspot, we also find the CLFLUSH instruction, when hotspot starts, it will generate a code stub
	
  __ bind(flush_line);                         
  __ clflush(Address(addr, 0));          //addr: address to flush 
  __ addptr(addr, ICache::line_size);                                         
  __ decrementl(lines);                   //lines: range to flush
  __ jcc(Assembler::notZero, flush_line);

The code stub can be invoked in the last phase of C1 compiler.
  // done
  masm()->flush();             //invoke ICache flush  

It try to flush the cache that instruction stored.

void AbstractAssembler::flush() {
  sync();
  ICache::invalidate_range(addr_at(0), offset());
}

Use this we can flush a range of address, but the problem is the address is uncontrollable, we cannot flush the address we wanted. And the compiler will spend much time, we cannot get enough CLFLUSH in 64ms.

c)	Other Compiler
C2 Compiler is different from C1 Compiler. C2 Compiler is highly optimizing bytecode compiler. Optimizations include global value numbering, conditional constant type propagation, constant folding, and global code motion and so on. We cannot generate prefetch, clflush, movnt* instruction directly.
Shark Compiler is based on LLVM, we ignore it because it`s not the default compiler.

## 2.2 Chrome V8
The V8 JavaScript Engine is an open source JavaScript engine developed by Google Chrome web browser. V8 compiles JavaScript to native machine code before executing it. V8 does not have interpreter, it translates JavaScript to AST (Abstract Syntax Tree), then walks the AST to generate machine codes.
After research, we did not find clflush, movnt* instructions when V8 compile the JavaScript. We found prefetch instruction in a function.
The function that generate prefetch is:

MemMoveFunction CreateMemMoveFunction() {

  __ prefetch(Operand(src, 0), 1);
  __ cmp(count, kSmallCopySize);    //kSmallCopySize=8
  __ j(below_equal, &small_size);  
  __ cmp(count, kMediumCopySize);   //kMediumCopySize=63
  __ j(below_equal, &medium_size);
  __ cmp(dst, src);
  __ j(above, &backward);


This function is used to move memory, when the instruction buffer is not big enough to store the compiled code. V8 has to enlarge the buffer, at this time the instruction can be generated one time. It cannot be used to trigger RowHammer.

## 2.3 .NET CoreCLIR
CoreCLR is the .NET Core Runtime. It includes RyuJIT, the .NET GC and many other components. RyuJIT is the JIT compiler in .NET CoreCLR. RyuJIT only defines some common x86 instructions (Figure 2.3) that do not include the instruction needed to trigger RowHammer.
We found the prefetch instruction in .NET GC component, just as shown in Figure 
## 2.4 But unfortunately the prefetch is disable by default (Figure 2.5). 	
 
Figure 2.3



void gc_heap::relocate_survivor_helper (BYTE* plug, BYTE* plug_end)
{
    BYTE*  x = plug;
    while (x < plug_end)
    {
        size_t s = size (x);
        BYTE* next_obj = x + Align (s);
        Prefetch (next_obj);
        relocate_obj_helper (x, s);
        assert (s > 0);
        x = next_obj;
    }
}

Figure 2.4


//#define PREFETCH
#ifdef PREFETCH
__declspec(naked) void __fastcall Prefetch(void* addr)
{
   __asm {
       PREFETCHT0 [ECX]
        ret
    };
}
#else //PREFETCH
inline void Prefetch (void* addr)
{
    UNREFERENCED_PARAMETER(addr);
}
#endif //PREFETCH
Figure 2.5
## 2.4 Firfox SpiderMonkey
SpiderMonkey provides JavaScript support for Mozilla Firefox, we didn`t find the instructions needed to trigger RowHammer in it.

# 0x03 Conclusion
The purpose of our research is to trigger RowHammer through script languages. In order to improve the efficiency of the language, most of them has JIT compiler. We analyzed Hotspot, Chrome V8, .NET CoreCLR and SpiderMonkey to try to verify our idea. But finally we found it is hard to get what we want.

1£©	Trigger RowHammer is not easy, we only have 64ms. It means the number of irrelevant instructions must be very few. Otherwise the number of the wordline toggle on and off is not enough to trigger RowHammer. 
2£©	The instruction we need is uncommon. The compiler does not generate these instructions directly.
3£©	In the view of JIT compiler developers, in order to cross-platform, JIT usually abstracts the instruction, and implements it on different platforms. So the abstracted instruction is as few as possible. Because more instructions means much more codes. In the source code we analyzed, only hotspot abstracts the prefetch instruction. JIT compiler always try to avoid a lot of instruction definitions. (There are some special cases where script languages use third-party JIT engine such as AsmJIT, the engine usually supports all instructions. But for now, most languages always build JIT dependently).
4£©	In our research, we found the instruction we need always generated to assistance JIT compile process. For example, using prefetch to increase the speed of data move, using clflush to flash cache to assure the execution of the code generated. Instructions are not translated from script directly, the RowHammer problem cannot be triggered due to the small amount and slow speed of the instruction generation.


# Resources
1. Google Project Zero
http://googleprojectzero.blogspot.com/2015/03/exploiting-dram-rowhammer-bug-to-gain.html
2. Paper: Flipping Bits in Memory Without Accessing Them: An Experimental Study of DRAM Disturbance Errors  http://users.ece.cmu.edu/~yoonguk/papers/kim-isca14.pdf
3. Source code


