# CPU虚拟化

## CPU虚拟化介绍

​		对于仿真器或是模拟器，其需要把目标机中的指令在物理机上进行模拟执行，并且需要处理目标机的某些指令更新或是读取CSR等寄存器的操作，并且可以检测中断事件并按照目标机架构的设计进行处理，同时还需要处理目标机指令的访存请求。这些功能在物理机上都由CPU和其外围硬件负责处理，但在模拟器中往往只有前两者，即指令模拟执行与寄存器管理在CPU虚拟化中实现，后两者在内存虚拟化和中断虚拟化中单独实现。

​		此外因为目标机与主机的架构往往不同，因此仿真器和模拟器多采用指令解释或是动态翻译的机制实现目标机指令的模拟执行，而不像KVM这类采用了硬件辅助虚拟化技术的程序。Spike是一个标准的仿真器，其虚拟CPU实现了目标架构指令的解释执行、处理器寄存器管理与中断处理三个功能，并且采用了解释器的机制来实现 RISC V 指令集架构目标机的模拟执行。

​		本文主要介绍了 Spike 中虚拟CPU对应的类，虚拟CPU的初始化过程，虚拟CPU对象如何处理CSR寄存器的读写，以及虚拟CPU对象如何处理中断。对于指令集的模拟（RISC-V 架构指令的解释执行）会在另外的文章中进行更详细的说明。

### Spike的CPU虚拟化

​		Spike中 `processor_t` 类用于表示虚拟CPU，其成员变量与函数按照功能可以大致分为五类，分别是处理CSR寄存器读写，设置和检测对指令集的支持，处理中断和陷入事件，注册 RISC-V 指令的解释方法，以及进行指令的解释执行。根据功能的不同，该类的成员函数主要在 `processor.[cc,h]` 和 `execute.[cc]` 两个文件文件中被声明和实现。

+ `processor.[cc,h]` 文件中包含 `processor_t` 类的全部变量和方法声明，并且包含如下的方法定义：
  + 有关CSR寄存器读写`get_csr()` `set_csr()` 等方法，用于设定和读取CSR寄存器，或是定义PMP这类数量可变寄存器的数量；
  + 有关指令集支持 `parse_isa_string()` `parse_priv_string()` `parse_varch_string()`  `register_extension()` `supports_extension()` 等方法，用于设定虚拟CPU支持的指令集和扩展指令集，或是检测当前虚拟CPU是否支持某指令集；
  + 有关中断处理 `take_interrupt()` `take_trap() `  等方法，用于检测当前是否有中断事件发生，并且处理已发生的中断或是陷入；
  + 用于向虚拟CPU中注册指令的 `register_base_instructions()` 和  `register_insn()`等方法，用于向虚拟CPU中注册每条 RISC V 指令对应的解释方法，供模拟时解释执行应用程序的指令使用。

+ `execute.[cc]` 文件中则主要是 `processor_t` 类中有关指令仿真的成员函数的定义。由于  `processor_t` 类中的方法较多较杂，这些方法会在讲解该功能的章节中再进行详细介绍。

​		模拟器进行初始化时，`sim_t` 类的构造函数中会创建多个 `processor_t` 实例，并存储到 `procs` 数组中，代表虚拟机的CPU；`processor_t` 类的构造函数会根据参数初始化虚拟CPU的一些可变配置，如 虚拟CPU支持的指令集模块，虚拟CPU支持的指令长度（32位或是64位），各条 RISC-V 指令的解释方法，虚拟 RISC-V CPU 的PMP寄存器数目等等；每一个 `processor_t` 实例代表一个虚拟CPU核心，这个实例会维护该虚拟CPU的运行状态和寄存器的值，如根据虚拟CPU的寄存器的值进行取指令时读取PC寄存器，在进行目标程序的解释执行时读取和更新参与计算的通用寄存器，也会根据中断状态寄存器处理中断事件。

​		在目前版本，无论虚拟机有多少核心，Spike 只有一个解释执行线程，所有虚拟机的仿真计算都会在这一个线程内进行，这意味着同时只会有一个虚拟CPU可以执行目标程序。在主模拟`sim_t` 类的控制下，模拟器会在 `sim_t::step` 方法中调用 `processor_t::step` 方法执行目标程序的代码；在这个函数中，当前活跃的虚拟CPU会进行目标程序的解释执行，而每个虚拟CPU只能连续解释执行有限数目的指令，当指令达到预设值后（默认为5000），模拟器会把将下一个虚拟CPU标记为活跃CPU，当再次调用 step 方法执行目标程序时则会让新的活跃CPU进行目标程序的解释执行，依次循环往复，实现多个虚拟CPU的伪并行执行。

​		虚拟CPU中断的处理也在`processor_t::step` 方法，在函数内有一循环参数为该CPU本次可以执行的指令条数的循环，每次循环开始时会尝试获取中断状态，如果成功获取中断到则会丢出一个 *c++异常* 并被循环内的 `catch` 语句捕获，进行中断或是陷入的处理。如果当前没有中断事件发生或是对应的中断状态未被使能，则虚拟CPU会按照正常的仿真流程，获取需要执行的指令，并根据已经注册的指令解释方法进行解释执行。

### 虚拟CPU的创建

​		在Spike启动后，会在 `main` 函数中进行参数的读入和处理，并根据用户参数创建MMIO设备，分配虚拟内存实例，载入目标可执行程序，构造 `sim_t` 实例，设定用于追踪统计数据的回调并设定 debug 设备。`sim_t` 实例将负责管理Spike模拟器有关目标机模拟的所有操作。

​		在 `sim_t` 类的构造函数中将进行其他虚拟组件的初始化，首先会将传入的虚拟内存作为一个设备挂载到虚拟总线上，随后将在 `main` 中创建的MMIO设备也挂载到虚拟总线上，虚拟机中的程序可以通过总线，访问被注册到特定地址上的设备，具体实现可以阅读有关总线和设备的章节。随后构造函数会将虚拟机的虚拟总线挂载到debug模块总线上，这样当使用debug模块时，可以根据需要读写虚拟机某地址的数值，并且还给debug模块创建了一个MMU单元，用于在使用debug模块时可以访问虚拟内存。

​		之后构造函数会开始创建虚拟CPU。首先检测hartids数组是否被创建，如果创建其数目是否与虚拟CPU的核心数相匹配，如果hartids数组不为空并且数目与核心数不匹配，Spike模拟器将直接退出。接下来构造函数会根据 nprocs 参数创建对应数目的虚拟CPU核心，这一步将创建 nprocs 个 `processor_t` 实例，并根据hartids数组中的元素（hartids不为空）或是从0开始（hartids为空）依次设定核心id。

​		`processor_t` 的构造函数会专注进行虚拟处理器核心的初始化。首先构造函数会根据传入的指令集字符串解析出当前虚拟机应对支持的指令集，并设定对应的 misa CSR 寄存器和 extension_table 哈希表，用于在随后检查虚拟CPU是否支持某特定指令集，并且还会根据指令集字符串设定 max_len 值，该值表示虚拟CPU的指令长度。在完成支持指令集的初始化后，构造函数会调用 `register_base_instructions` 函数设定虚拟CPU翻译目标指令的方法，该函数会根据 "encoding.h" 和编译过程中生成的 "insn_list.h" 文件，向 `instructions` 这个类私有数组中添加新的指令翻译方法，将指令的编码与以c语言写成的指令仿真执行函数相映射；同时该函数也会初始化 `opcode_cache` 用于快速处理近期已经执行过的指令，加快模拟速度。回到 `processor_t` 的构造函数，接下会创建虚拟MMU单元，每个核心都会拥有自己的虚拟MMU单元，用于访问  `sim_t` 实例中被注册的虚拟内存。

​		`processor_t` 的构造函数接下来会注册反汇编器用于在debug时对指令进行反汇编，然后设定pmp寄存器的参数，并根据 `max_len` 设定虚拟MMU单元的能力。最后构造函数会调用 `reset` 函数重置处理器状态。至此`processor_t` 的构造函数结束返回。

​		 `sim_t` 类的构造函数会将创建的  `processor_t` 实例储存在 procs 数组中。在完成虚拟CPU的创建后，接下来将生成dtb文件，创建 `clint` 核心本地中断控制器，并将 `clint` 挂载到虚拟总线上，如果dtb中有其地址则按dtb中设定的地址进行注册，否则使用默认值 `CLINT_BASE` 。之后将根据dtb进一步设定虚拟CPU核心的配置，如pmp寄存器的数目，虚拟MMU单元的能力和地址翻译方法。当所有虚拟核心都已经根据dtb完成设定后， `sim_t` 的构造函数也便结束返回。

### CSR寄存器的读写

​	 	`processor_t` 类主要通过`get_csr()` `set_csr()` 以及直接读写私有成员变量的方法读写虚拟CPU的CSR寄存器。

​		在 `get_csr()` `set_csr()` 方法中，对于PMP有关寄存器的读写会先单独处理；因为这些寄存器有不定的数目，以及需要修改的值要先计算对应的偏移值，因此这些读写请求会在函数开始处单独使用判断和循环进行进行处理。

​		对于其他的CSR寄存器，这两个方法主要通过一个巨大的 switch...case 语句，switch 判断的对象为需要读写的寄存器名称，随后在对应的 case 语句块中根据寄存器的设计要求对传入或返回的数值进行处理，比如对寄存器的值进行掩码计算以过滤掉一些位，或是当写入值满足一定条件时忽略本次写请求。其中 `CSR_MSTATUS` 寄存器的写入较为复杂，在修改时需要格外注意。

​		在目前版本的 Spike 中，能够实际发挥功能的CSR寄存器对象并不多。主要有有运行状态的有关的 MSTATUS 寄存器，与PMP保护有关的 PMPADDR 和 PMPCFG 寄存器，与中断有关的 MIP，MIE，SIP，SIE 中断使能与中断状态寄存器，以及用于保存中断原因的寄存器。当然还有设定和指示处理器所支持的指令集的 MISA 寄存器。其他的寄存器大多不会被 Spike 本身的虚拟设备所使用，更多用于在仿真执行的过程中保证仿真代码修改 CSR 寄存器后能够被追踪，以及某些需要读取 CSR 寄存器的程序仍可以正常运行，尽管其读取到的值可能毫无意义。

### 支持指令集的设置与检测

​		因为 RISC-V 处理器的指令集是模块化的设计，可以根据需要启用、禁用指令集或是添加新的自定义指令集。

​		在添加新指令集时会向MISA寄存器中写入新的标志位，同时会在 `processor_t` 实例的 `extension_table` map对象添加 key为新指令集名称、value为true的项。添加新的指令集主要会使用四个方法：

+ `parse_isa_string()` 该函数接收一个表示当前虚拟CPU支持的指令集的字符串，会解析该字符串并设置指令集支持；

+ `parse_rpiv_string()` 该函数接收一个表示当前虚拟CPU支持的特权级的字符串，会解析该字符串并设置特权指令集的支持；

+ `parse_varch_string()` 该函数接收一个表示当前虚拟CPU支持的向量化硬件变量的字符串，会解析该字符串并设置向量化参数；

+ `register_extension()` 该函数接收一个 `exension_t` 类型的参数，用于从外部运行库中引入自定义的扩展指令集。

  

​		在查询是否支持某一指令集时，可以使用  `supports_extension()` 进行查询，或者在仿真指令翻译文件中使用 `require_extension()` 宏进行查询，后者是对前者的简单包装，会调用负责执行当前指令的虚拟CPU对象的 `supports_extension()` 方法查询某一指令集是否被支持。该函数接收一个 **unsigned** **char** 类型的参数，该参数就是指令集的名称，如 *'A' 'I' 'F'*，此外对于一些自定义指令集，该参数也可能是 *EXT_ZFH EXT_ZVEDIV* ，这些是定义在 *processor.h* 中的枚举量。

​		`supports_extension()` 会检查传入的参数，如果参数在 'A' 到 'Z' 的范围内，即参数为一个大写英文字母，则函数会查询其所属  `processor_t` 实例的 state.misa 变量，即虚拟CPU的MISA寄存器；如果参数不在该范围内，那么便认为传入的参数是预先定义的自定义扩展指令集，则函数会查询其所属  `processor_t` 实例的  extension_table 对象。

### 中断陷入事件处理

​		Spike的中断陷入事件处理较为简单，本身也并未实现中断回调处理函数的支持，且由于中断与陷入处理函数属于 `processor_t` 类，故暂时将中断陷入事件处理部分置于CPU虚拟化章节中。

​		在虚拟CPU的指令执行循环中，如果发生中断或者陷入事件，Spike会丢出一个 c++ 异常，并通过 catch 语句捕捉执行过程中被丢出的异常，然后调用 `take_trap()` 方法处理被捕获的异常中描述的中断或是陷入事件。

​		在指令执行循环的开始处，会调用 `take_pending_interrupt()` 函数获取已发生的被阻塞等待的中断事件，该函数是对 `take_interrupt()` 函数的包装，会将后者的调用参数设置为 *state.mip & state.mie* ，即当前被使能且已经发生的中断标志位，随后 `take_interrupt()` 函数会检查中断对应的特权级是否被启用，对应的中断源是否被启用，然后根据不同特权级中断的优先级和不同中断事件的优先级依次获取中断事件，然后将中断事件打包为一个 `trap_t` 类型的 c++ 异常并丢出。`take_interrupt()`  函数中仅仅获取了当前被使能的、已发生的、有最高优先级的中断事件，中断事件的具体处理则被交给 `take_trap()` 函数一并进行。

​		对于其他的陷入事件，可能会在指令的翻译执行过程中任何时候被丢出，然后根据调用堆栈和 try...catch 嵌套一步步向外层丢出，直到被虚拟CPU的 step 函数的执行循环中的 catch 方法捕获，然后该异常被作为参数调用 `take_trap()` 函数进行处理。在 `take_trap()` 函数中，会先根据配置进行debug输出，如果陷入的原因为 断点 "BreakPoint" 事件，则会根据当前状态进入debug模式然后跳转到断点处理函数，或者已经在debug模式时则直接修改pc寄存器进行跳转。这之后，`take_trap()` 函数会检查当前虚拟CPU的中断陷入事件降级处理设置，根据设置的CSR寄存器的值，会在M模式，HS模式和VS模式中进行中断和陷入事件的处理；在具体的处理中，则会根据陷入的原因设定虚拟CPU寄存器，然后返回指令执行循环。