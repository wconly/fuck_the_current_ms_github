不用说，网上有大把大把解说前端的文章，但看来看去都不太完整，要不就是老版本（其实区别也不算太大惹）。所以自己抽空总结一下，理清一下思路总是吼的~

clang的main位于tools\driver\driver.cpp文件中。

前面提到过，clang的作用其实如文件名所示，一名老司机罢惹。负责吸入用户给出的参数、源文件，进行解析，根据参数对源文件进行处理，最后得到用户想要的东东（不都是这个套路吗）。进一步说明则是，除了解析源码为ast，还会驱动后端在不同阶段生成ir、汇编码、机器码，中端的优化，乃至连接器最后链接为可执行文件等。

发展至今，clang已经很复杂惹，处理的路径万万条，所以这里来看最简单的处理方式：

clang -O3 hello.c

即，以-O3优化级别去编译hello.c。

简单想一下就知道，光是这几个参数根本不足以支撑整个编译过程。是滴，其实clang暗地里帮我们补充了其他默认的参数，查看完整参数的方法为：

clang -O3 -### hello.c

![输入图片说明](https://github.com/wconly/fuck_the_current_ms_github/blob/main/pics/010618_8ed47b8c_5282271.png)

仔细看的话，红线上为编译过程，下则为链接过程，除了红框框出的输入参数，其他都是clang自己脑补粗来的……可见其默认用的是GNU ld，亦可以添加参数-fuse-ld=lld使用llvm自己的链接器。

利用-ccc-print-bindings也可以印证这一点（显示实际执行动作的应用）：

![输入图片说明](https://github.com/wconly/fuck_the_current_ms_github/blob/main/pics/014052_728df161_5282271.png)

clang的参数定义位于 Options.td ，查找###，可见：

def _HASH_HASH_HASH : Flag<["-"], "###">, Flags<[DriverOption, CoreOption]>,
    HelpText<"Print (but do not run) the commands to run for this compilation">;

hash即#的意思，“定义 _HASH_HASH_HASH 的旗标为 -###，这是老司机的选项，也是核心选项，注释：打印（但不执行）当前编译过程中所用的命令”。
以此类推。且可以推广到查看其它*.td文件来查看类似选项集合的定义。

通过参数-ccc-print-phases，可以打印出编译过程中的各个阶段（phases）：

![输入图片说明](https://github.com/wconly/fuck_the_current_ms_github/blob/main/pics/012819_8af9b3db_5282271.png)

输入 “hello.c” 这个c源码  
预处理器 将c源码 处理为cpp输出文件  
编译器 根据cpp输出文件 生成ir  
后端 根据ir 生成汇编码  
汇编器 根据汇编码 生成编译对象文件（.o）  
连接器 根据编译对象文件 生成可执行的镜像文件  

默认情况下，clang会根据所在平台进行编译，也就是说编译后的程序会执行在编译发生的机器上。这一点，clang找到所在平台的信息是小菜一碟的，例如上图中，她看穿了我的机器平台为x86_64上linux-gnu。你也可以设想一下，换作是你的话，有多少方法去识别具体平台？

可以看到，clang补充的第一个参数为“-cc1”，这是clang的一种模式，各个阶段分工明细，可用于调试。

除此之外，还另有-cc1as与-cc1gen-reproducer两种模式。

前者从名字可以看出多了as（ass），稍微敏感一点儿的一定会想到assembler（果真）。简单来说，这种模式不再跑cc1中那么多的阶段，而是直接利用llvm的MC汇编器，根据需求去生成汇编码、各种格式的机器码。

同理，后者的关键位于gen-reproducer。直译便是“生成可复现的内容”，简言之，即生成以libclang制成的各种工具（围绕libclang做的工具，其实就是把clang当作核心，将其与ast操作有关的部分功能封装成库，留出C接口，向外提供服务，那些所谓的工具就是调用这些接口实现的）所产生的调用记录文件。从网上来看，用的不太多，想必有了这些接口的调用记录，就可以方便地调试自己基于libclang做的小工具惹，对于那些做前端工作的人也许会有大帮助吧……

可见，考察的核心应该是-cc1模式。下面提取出其核心函数：


```c
main()
  // 构造驱动的基本信息，如clang的路径、平台信息等。
  // 根据Triple可推测平台信息有3条属性，举个例子，如"i686-pc-win32"。详见Triple.cpp。
  Driver TheDriver(Path, llvm::sys::getDefaultTargetTriple(), Diags); 
  // ExecuteCC1Tool为-cc1、-cc1as、-cc1gen-reproducer三选一的入口
  TheDriver.CC1Main = &ExecuteCC1Tool; 
  // 构造编译流程
  std::unique_ptr<Compilation> C(TheDriver.BuildCompilation(argv));
  // 执行编译流程
  Res = TheDriver.ExecuteCompilation(*C, FailingCommands);
```


```c
Compilation *Driver::BuildCompilation(ArrayRef<const char *> ArgList)
  // Compilation只是个信息携带者，其中保存了流程中的所有关键数据结构、参数。
  Compilation *C = new Compilation(*this, TC, UArgs.release(), TranslatedArgs,
                                   ContainsError);
  // 构建actions
  BuildActions(*C, C->getArgs(), Inputs, C->getActions());
  // 构建jobs
  BuildJobs(*C);
```

提到action与job，不得不搬出一张出自llvm官网中《Driver Design & Internals》的图：

![输入图片说明](https://images.gitee.com/uploads/images/2020/1125/015501_44c265d1_5282271.png "屏幕截图.png")

action其实就是由-ccc-print-phases参数导出的那些phase (action)组成，也就是单个input编译过程中“抽象”出的各个环节。

你可以这样理解：
编译一个.c文件的任务称为一个input（两个.c文件就是两个input），一个input对应一个action，而每个action又是由一系列phase步骤构成（phase用phase action描述）。

action的类型由ActionClass结构来描述：


```c
enum ActionClass {
    InputClass = 0,
    BindArchClass,
    OffloadClass,
    PreprocessJobClass,
    PrecompileJobClass,
    HeaderModulePrecompileJobClass,
    AnalyzeJobClass,
    MigrateJobClass,
    CompileJobClass,
    BackendJobClass,
    AssembleJobClass,
    LinkJobClass,
    IfsMergeJobClass,
    LipoJobClass,
    DsymutilJobClass,
    VerifyDebugInfoJobClass,
    VerifyPCHJobClass,
    OffloadBundlingJobClass,
    OffloadUnbundlingJobClass,
    OffloadWrapperJobClass,

    JobClassFirst = PreprocessJobClass,
    JobClassLast = OffloadWrapperJobClass
  };
```

每个action类型有对应的类实现，如OffloadClass类型的对应action为OffloadAction，用于描述有主从（host/device）关系这样的硬件（如GPU等）。  
明显地，我们上面列出的phases也有对应的action。

需要注意的是，phases有自己的action集合，所有的phases项为：


```
  enum ID {
    Preprocess,
    Precompile,
    Compile,
    Backend,
    Assemble,
    Link,
    IfsMerge,
  };
```



如果说action只是抽象步骤，job则更趋近实际执行的步骤。

为什么仅说是“趋近”呢？因为job下还有还有一层command（其实把二者看作等价也是行的吧……）。


```cpp
// 待执行的一系列job
class JobList {
public:
  using list_type = SmallVector<std::unique_ptr<Command>, 4>;
private:
  // 其实乔布斯是Command构成的
  list_type Jobs;
}

class Command {
  // 本Command由哪个Action转换而来？
  const Action &Source;
  // 执行本Command的程序
  const char *Executable;
  // 执行本Command所需的参数
  llvm::opt::ArgStringList Arguments;
  // 由本Command处理的源文件
  llvm::opt::ArgStringList InputFilenames;
}
```


继续看上图，将action与工具（链）绑定（bind），也就是确定执行action ox的工具tool ox。  
再将action a转译（translate）到job，也就是抽象步骤到实际执行步骤的转换(而拿到实际步骤的则为job下的command)。

上面提到，BuildActions的构造结果与-ccc-print-phases参数显示的内容相同，而且也不是什么技术核心，这里就跳过去惹……


```c
BuildJobs(*C)
  // 遍历每个action，并构建出对应的job
  for (Action *A : C.getActions()) { 
    BuildJobsForAction(C, A, &C.getDefaultToolChain(),
                       /*BoundArch*/ StringRef(),
                       /*AtTopLevel*/ true,
                       /*MultipleArchs*/ ArchNames.size() > 1,
                       /*LinkingOutput*/ LinkingOutput, CachedResults,
                       /*TargetDeviceOffloadKind*/ Action::OFK_None);
  }
```


```c
InputInfo Driver::BuildJobsForAction(...)
  InputInfo Result = BuildJobsForActionNoCache(...);
```


```c
InputInfo Driver::BuildJobsForActionNoCache(...)
  // 尽量将可用同一工具处理的phase action们合并在一起
  const Tool *T = TS.getTool(Inputs, CollapsedOffloadActions);
  // 正式构建job
  T->ConstructJob(
          C, *JA, Result, InputInfos,
          C.getArgsForToolChain(TC, BoundArch, JA->getOffloadingDeviceKind()),
          LinkingOutput);
``` 

根据-ccc-print-phases的结果来看，我们应该有6个phase actions，对应生成6个jobs。

但是看到-###的结果，我们仅用两条指令，分别用clang与GNU ld生成.o与可执行文件。这是为神马呢？

答案就在getTool(...)函数里。这个函数会尽量将可用同一工具完成的actions合并（combine）在一起，毕竟这样会简化其中的构建与执行过程。而我们执行的简单编译命令正好符合其中的条件，所以除了最后的linker，其他actions都合并成为一条指令，且统一由clang来执行。

因此const Tool *T即为clang程序，下面的T->ConstructJob跑的也就是Clang::ConstructJob。


```c
void Clang::ConstructJob(...)
  // 构建指令参数（clang开始脑补参数中...）
  ArgStringList CmdArgs;
  CmdArgs.push_back("-cc1");
  CmdArgs.push_back("-triple");
  ......
  // 将参数添加至类型为CC1Command的Command
  C.addCommand(
        std::make_unique<CC1Command>(JA, *this, Exec, CmdArgs, Inputs));
```

Clang::ConstructJob的任务基本就是构造command，如果追寻，可以发现向CmdArgs里push的参数与-###给出的一致，最后将这些参数附到CC1Command上。

前戏搞好后就该正式执行惹~


```c
int Driver::ExecuteCompilation(Compilation &C, ...)
  // 执行jobs
  C.ExecuteJobs(C.getJobs(), FailingCommands);


void Compilation::ExecuteJobs(const JobList &Jobs, ...)
  // 遍历每个job，依次执行其command
  for (const auto &Job : Jobs) {
    ExecuteCommand(Job, FailingCommand);
  }


int Compilation::ExecuteCommand(const Command &C, ...)
  // 执行该command
  C.Execute(Redirects, &Error, &ExecutionFailed);
```

注意，上面提到，我们目前跑的路线是普通的CC1Command，而又有


```c
class CC1Command : public Command {
public:
  CC1Command(...);
  void Print(...);
  int Execute(...);
  void setEnvironment(...);
}
```

上述继承关系，所以上面的C.Execute(...)实际执行的是CC1Command::Execute(...)。


```c
int CC1Command::Execute(...)
  // 在当前执行的编译任务里要跳至Command::Execute
  return Command::Execute(...);


int Command::Execute(...)
  // 当前的Executable也就是ExecutableClang::ConstructJob(...)中的Exec，即clang
  // Args也就是Clang::ConstructJob(...)中的CmdArgs，即clang执行时所需参数
  // Executable+Args=参数-###打印粗来的内容 
  return llvm::sys::ExecuteAndWait(Executable, Args, Env, Redirects,
                                   /*secondsToWait*/ 0,
                                   /*memoryLimit*/ 0, ErrMsg, ExecutionFailed);


int sys::ExecuteAndWait(StringRef Program, ArrayRef<StringRef> Args, ...)
  Execute(PI, Program, Args, ...);


static bool Execute(ProcessInfo &PI, StringRef Program, ArrayRef<StringRef> Args, ...)
  // fuck出一个子进程，让它去执行-###中的指令
  int child = fork();
  switch (child) {
    case 0: {
      std::string PathStr = Program;
      // 走你！
      execve(PathStr.c_str(), const_cast<char **>(Argv), const_cast<char **>(Envp));
    }
  }
```


这次fuck粗的线程执行clang是带完整参数的，根据


```c
main()
  // 第一个参数不是最后一个参数（不是唯一的参数），而且为-cc1，参考-###
  if (FirstArg != argv.end() && StringRef(*FirstArg).startswith("-cc1")) {
    return ExecuteCC1Tool(argv);
  }
```

因此，


```c
static int ExecuteCC1Tool(SmallVectorImpl<const char *> &ArgV)
  // 当然是选cc1_main惹
  if (Tool == "-cc1")
    return cc1_main(makeArrayRef(ArgV).slice(2), ArgV[0], GetExecutablePathVP);
  if (Tool == "-cc1as")
    return cc1as_main(...);
  if (Tool == "-cc1gen-reproducer")
    return cc1gen_reproducer_main(...);


int cc1_main(...)
  // 创建编译实例
  std::unique_ptr<CompilerInstance> Clang(new CompilerInstance());
  // 根据编译参数构建CompilerInstance实例Clang的CompilerInvocation，也就是各种编译选项（options）
  bool Success = CompilerInvocation::CreateFromArgs(Clang->getInvocation(), Argv, Diags);
  // 执行前端的编译动作
  Success = ExecuteCompilerInvocation(Clang.get());
```


```c
bool ExecuteCompilerInvocation(CompilerInstance *Clang)
  // 解析-mllvm参数，这是llvm后端指令，所以可以看到要通过llvm::域的函数去执行。
  // -mllvm后跟随的参数效果同opt+其参数的执行效果
  if (!Clang->getFrontendOpts().LLVMArgs.empty()) {
    ...
    llvm::cl::ParseCommandLineOptions(NumArgs + 1, Args.get());
  }

  // 创建前端（frontend）action，也就是clang前端的抽象，根据编译器获得的参数去创建具体的前端类型
  std::unique_ptr<FrontendAction> Act(CreateFrontendAction(*Clang));
  // 执行上述编译器前端的动作
  bool Success = Clang->ExecuteAction(*Act);
```


```c
std::unique_ptr<FrontendAction> CreateFrontendAction(CompilerInstance &CI)
  // 创建具体的action
  std::unique_ptr<FrontendAction> Act = CreateFrontendBaseAction(CI);
  return Act;


static std::unique_ptr<FrontendAction> CreateFrontendBaseAction(CompilerInstance &CI)
  switch (CI.getFrontendOpts().ProgramAction) {
    ...
    // 根据-###可以查到clang编译命令中，clang脑补粗了"-emit-obj"参数，因此选择创建EmitObjAction类型的action，也就是编译出.o文件惹
    case EmitObj:
      return std::make_unique<EmitObjAction>();
    ...
  }
```

其他的ProgramAction可参见enum ActionKind中的各种枚举项。
值得注意的一点是EmitObjAction类的继承关系：


```c
FrontendAction  
      ↑  
ASTFrontendAction  
      ↑  
CodeGenAction  
      ↑  
EmitObjAction  
```


实际上，EmitObjAction的工作都是其父类实现的……（这买卖好啊，甩手儿掌柜的，光挂个名神马都不干惹，天啦噜~）

```c
bool CompilerInstance::ExecuteAction(FrontendAction &Act)
  // action执行前的准备工作，EmitObjAction没有实现这个函数，采用FrontendAction的默认实现，即返回true
  Act.PrepareToExecute(*this)

  // 设置目标平台，相当于确定了编译产物的平台，具体讲也就是目标平台（CPU）的特征
  setTarget(TargetInfo::CreateTargetInfo(getDiagnostics(), getInvocation().TargetOpts));

  for (const FrontendInputFile &FIF : getFrontendOpts().Inputs) {    // 逐个吸入用户指定的输入文件
    if (Act.BeginSourceFile(*this, FIF)) {                           // 处理输入文件的准备工作
      if (llvm::Error Err = Act.Execute()) {                         // 执行action
        consumeError(std::move(Err)); // FIXME this drops errors on the floor.
      }
      Act.EndSourceFile();                                           // 处理完的后续工作
    }
  }
```

遍历EmitObjAction及其父类可以发现，EmitObjAction的BeginSourceFile(...)、Execute()与EndSourceFile()，实际分别执行的是FrontendAction类的BeginSourceFile(...)、Execute()与EndSourceFile()实现。

事实上，BeginSourceFile(...)与EndSourceFile()对于我们当前的逻辑主流程关系不大。这里仅简单三言两语描述一下惹……

输入文件的格式有三种：


```c
enum Format {
  Source,      // 源代码文件
  ModuleMap,   // 名为module.modulemap的文件，用于描述模块间头文件的映射关系，详细参见官档《MODULES》中的《Module maps》部分
  Precompiled  // 预编译过的文件，有Clang AST文件与预编译模块文件（precompiled module file，简作PCM）
};
```

对于源文件还有具体语言之分：


```c
enum class Language : uint8_t {
  Unknown,

  /// 汇编码：仅为了预处理（preprocess）而接受这种源文件。
  Asm,

  /// LLVM IR：用这种编码进行优化处理，并将其编译为汇编码或.o文件。
  LLVM_IR,

  ///@{ 下列为前端可解析并编译的语言。
  C,
  CXX,
  ObjC,
  ObjCXX,
  OpenCL,
  CUDA,
  RenderScript,
  HIP,
  ///@}
};
```

上述文件格式与语言类型是根据文件扩展名判定的，由`InputKind FrontendOptions::getInputKindForExtension(StringRef Extension)`函数来实现，例如：


```c
...
// InputKind结构描述的是输入文件的全部信息，可以看作Format、Language及其他一些属性的集合
// 将以.ast与.pcm结尾的文件看作是Language为Unknown且Format为Precompiled的文件
Cases("ast", "pcm", InputKind(Language::Unknown, InputKind::Precompiled))
...
```

这样便对所有输入的文件有了进一步的划分。


