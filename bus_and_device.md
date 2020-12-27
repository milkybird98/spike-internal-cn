## 总线与设备

### Spike中的总线与设备的概念

Spike中通过 **虚拟总线(bus)** 这一概念和对象来将某个设备(device)和某个总线地址(addr)进行绑定，允许设备注册到其在内存总线的基地址(BASEADDR)上，保存这个设备的读写方法(load/store)，也能够通过需要访问的总线地址反向查找到注册到该基地址上的设备并获取其读写方法，并允许仿真器运行的仿真代码通过设备的地址访问对应设备的数据。

在Spike中， **设备(device)** 的概念和实际的soc中的设备的概念基本一致，所有的连接到总线上的、实现某个特定功能组件被为一个设备，一个设备是其所占据的总线地址空间、实际存储数据的空间和功能函数的合集，代码能够通过调用设备的 *load/store* 方法来读写设备的存储数据，或是控制设备实现特定功能。设备间将通过使用总线来获取其他设备的对象。

### 总线类的定义

总线使用 ```bus_t``` 类来表示，也是 ```abstract_device_t``` 类的一个子类，并在 ```sim_t``` 中被实例化为 ```bus```，作为 ```sim_t``` 实例的一个私有成员而被创建。  
```bus_t``` 共提供了4个public方法，一个私有成员变量。4个public方法分别为： *load/store/add_device/find_device*，前两个是```abstract_device_t```中虚方法的实现，后两者是新增的方法；1个私有成员为一个map对象，用于存储总线上“连接”设备的 **设备地址** 和 **表示设备的对象** 之间的映射关系。  

```c++
class abstract_device_t {
 public:
  virtual bool load(reg_t addr, size_t len, uint8_t* bytes) = 0;
  virtual bool store(reg_t addr, size_t len, const uint8_t* bytes) = 0;
  virtual ~abstract_device_t() {}
};
```

```c++
class bus_t : public abstract_device_t {
 public:
  bool load(reg_t addr, size_t len, uint8_t* bytes);
  bool store(reg_t addr, size_t len, const uint8_t* bytes);
  void add_device(reg_t addr, abstract_device_t* dev);
  std::pair<reg_t, abstract_device_t*> find_device(reg_t addr);
 private:
  std::map<reg_t, abstract_device_t*> devices;
};
```

#### 总线类的方法

+ load 
*load* 方法用于从特定地址处读取指定长度的字节。为了实现在总线上进行任意地址、跨设备的读取，会先根据需要访问的地址查找对应的设备，如果能够找到基地址满足要求的设备，那么就会计算读取地址在该设备内的地址偏移，并调用该设备的load方法以进行实际的数据读取；如果没有找到基地址满足要求的设备，那么就会返回 *false* 表示读取失败。

+ store
*store* 方法用于向特定地址写入指定长度的字节。store方法的执行逻辑与load方法一致。

+ find_device
*find_device* 用于查找并返回满足地址要求的设备对象。该方法会查询所有已经注册的设备，从中找到基地址不大于需要访问的地址且基地址最大的设备，如果能够找到这样的设备，那么认为该设备有可能符合要求，并将该设备的基地址与对象一并返回；如果无法找到这样的设备，或是总线中注册设备列表为空，那么返回一个空值。

+ add_device
*add_device* 用于向总线中注册一个新的设备。该方法较为简单，直接将设备的基地址作为key、设备对象作为value添加到```bus_t```的私有成员 *std::map<reg_t, abstract_device_t*> devices* 当中。同时利用了std：：map会有序存储key值这一特定，使用 *lower_bound/upper_bound* 方法，根据访问地址查询基地址满足要求的设备。

### 其他设备

每种设备都是使用其类来表示，这些类也都是 ```abstract_device_t``` 类的子类，重载其 *load/store* 方法，并实现了其他独有的方法，且大多添加了私有成员对象用以存储状态或数据。  
目前Spike中已经实现的设备有如下： *rom_device/mem/clint/mmio_plugin_device* ；接下来将分别介绍这四种设备与其类。

#### rom_device

*rom_device* 用于代表只读存储器设备(rom)，可以用于存储对于cpu只读的代码或是数据。在Spike中，一个*rom_device*的实例被用于存储 **bootrom** 代码和 **device tree** 数据，并在模拟开始时被初始化并作为一个设备被注册到总线地址 *0x1000* 处。  

该设备的类(```rom_device```)有3个public的方法，一个自定义的构造函数用于设定“rom”中的数据，以及一个私有成员变量，用于保存“rom”中需要存储的数据。该类的定义如下：

```c++
class rom_device_t : public abstract_device_t {
 public:
  rom_device_t(std::vector<char> data);
  bool load(reg_t addr, size_t len, uint8_t* bytes);
  bool store(reg_t addr, size_t len, const uint8_t* bytes);
  const std::vector<char>& contents() { return data; }
 private:
  std::vector<char> data;
};
```

+ rom_device_t构造函数
*rom_device_t*构造函数用于初始化“rom”内部的数据。该函数接受一个*vector<char>*参数传入，作为“rom”中需要固化保存的数据，并将传入的vector复制到该“rom”对象的私有成员*data*中，后者也是一个*vector<char>*对象。

+ load
*load* 方法用于读取“rom”中保存的数据。该方法首先确认要读取的数据处于“rom”的存储空间范围内，由于在总线中已经确认了访问地址是大于该“rom”的基地址的，因此只需要确定访问的地址的上限（addr+length）也在“rom”的存储空间内即可；如果读取的范围符合要求，该方法便会将私有的*data*对象中对应地址处的、要求长度的数据拷贝到传入的指针或引用中。

+ store
*store* 方法会直接返回*false*，不允许进行写操作；因为“rom”是只读设备，不能向其中写入数据。  

+ contents
*contents* 方法会直接返回当前“rom”中私有成员*data*的只读引用。

#### mem
*mem* 用于代表内存设备(ram)，允许预先写入程序或是数据供cpu执行，也可以让cpu在内存中进行读写操作用于存放运行中的数据。Spike中，一个*mem*的实例被用作模拟riscv soc的主存使用，在模拟开始时会初始化一个2GB的空间作为riscv soc的主存，并作为一个设备注册到总线地址 *0x80000000* 处。

该设备的类(```mem_t```)有4个public的方法，其中的*load*和*store*较为特殊，在Spike的实现中，内存的读写不由```mem_t```的*load*和*store*方法进行，而是由*mmu*模块通过```mem_t```的*contents*方法获取*ram*的数据的引用后，直接读写地址对应下标处的数据，如果开启了va则先读取页表并将va翻译为pa后在进行访问；此外禁用了复制其他```mem_t```的构造函数并定义了一个新的、根据传入的尺寸进行calloc的构造函数；该类还有两个私有成员变量，分别用作“ram”中数据的存储空间和保存“ram”的尺寸。```mem_t```类的定义如下：

```c++
class mem_t : public abstract_device_t {
 public:
  mem_t(size_t size) : len(size) {
    ...
  }
  mem_t(const mem_t& that) = delete;
  ~mem_t() { free(data); }
  bool load(reg_t addr, size_t len, uint8_t* bytes) { return false; }
  bool store(reg_t addr, size_t len, const uint8_t* bytes) { return false; }
  char* contents() { return data; }
  size_t size() { return len; }
 private:
  char* data;
  size_t len;
};
```

+ mem_t构造函数
*mem_t* 构造函数用于分配“ram”内部的空间。该函数接受一个*size_t*类型的参数，作为“ram”的存储空间的总大小，以byte为单位，并把该大小赋值给成员变量*len*；若空间大小符合要求，则会调用*calloc*申请分配一片连续内存空间，并把该空间的指针赋值给*data*，如果空间大小不符合要求或是分配失败，会触发*runtime_error*并结束程序。

+ load
*load* 该方法直接返回*false*，不允许进行读操作。

+ store
*store* 该方法直接返回*false*，不允许进行写操作。

+ contents
*contents* 方法会返回该“ram”中私有成员*data*的可读写引用。

+ size
*size* 方法返回当前“ram”的存储空间的总大小，以byte为单位。
 
#### clint
*clint* 表示了RISC V的核心本地中断控制器。在Spike中实现了其计时功能和时间比较功能，并作为处理器核心的实时时钟(RTC)使用，同时也可使用 *clint* 作为软件中断源置MIP寄存器中的MSIP位(机器态软件中断标志位)。实际上Spike中的 *clint* 并不一定"本地"，其超时后会在所有其绑定的核心上置MTIP位，而用 *clint* 置MSIP位时也同样可以根据需要，在多颗核心上触发中断，这些都取决于用户在初始化 *clint* 时绑定的核心有多少。 

在Spike中，*clint*更多的作为 *RTC* 使用，并且有两种计时方式：
1. 基于真实时间的计时：
初始化时从系统获取并保存当前的真实时间，在每次进行时间计算时再次从系统获取当前的真实时间，并认虚拟机的执行速度是基本不变的，将当时间和初始化时间的差值乘上固定的系数，将其虚拟机中时间的增量。
2. 基于仿真执行的计时：
认为虚拟机中每条指令所消耗的时间基本一致，在仿真了一定数目的指令后(缺省为5000条)，将虚拟机中在这个计时时间段内执行了的指令条数乘上固定系数，作为虚拟机中增加的时间。

该设备的类(```clint_t```)有4个public的方法，*load*和*store*用于读写内部寄存器，*size*获取其占据的内存空间的大小，*increment*计算时间增量；该类有较多的私有成员，*procs*表示绑定的核心，*freq_hz*表示*RTC*的增长频率(用于根据真实时间计算*RTC*的时间增长)，*real_time*用于决定计时模式，*real_time_ref_secs*和*real_time_ref_usecs*用于记录开始时的真实时间并作为参考时间，*mtime*记录*RTC*的当前时间，*mtimecmp*表示*Clint*绑定的第n个核心设定的超时中断比较值。

```c++
class clint_t : public abstract_device_t {
 public:
  clint_t(std::vector<processor_t*>&, uint64_t freq_hz, bool real_time);
  bool load(reg_t addr, size_t len, uint8_t* bytes);
  bool store(reg_t addr, size_t len, const uint8_t* bytes);
  size_t size() { return CLINT_SIZE; }
  void increment(reg_t inc);
 private:
  typedef uint64_t mtime_t;
  typedef uint64_t mtimecmp_t;
  typedef uint32_t msip_t;
  std::vector<processor_t*>& procs;
  uint64_t freq_hz;
  bool real_time;
  uint64_t real_time_ref_secs;
  uint64_t real_time_ref_usecs;
  mtime_t mtime;
  std::vector<mtimecmp_t> mtimecmp;
};
```

+ clint_t 构造函数
*clint_t* 构造函数会给私有变量赋初值，并获取当前的真实时间初始化*real_time_ref_secs*和*real_time_ref_usecs*，作为真实时间模式下计时的参考值。

+ load
*load* 方法可以读取*clint*的寄存器，根据读取地址的不同，分别能读取其绑定核心的MSIP位状态，绑定核心设定的超时中断时间以及RTC的内部时间。

+ store
*store* 方法可以设定*clint*的寄存器，根据写入地址的不同，分别能写入其绑定核心的MSIP标志位，设置绑定核心的超时中断的时间以及设置RTC的内部时间。在每次调用*store*方法后都会调用一次*increment(0)*更新RTC的内部时间。

+ size
*size* 方法获取 *clint* 占据的内存空间的大小。

+ increment
*increment* 方法会计算一次*RTC*的时间增长，具体细节可以参考前文。