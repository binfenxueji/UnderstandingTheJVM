# 第7章 虚拟机类加载机制

## 7.1 概述
- Java虚拟机把描述类的数据从Class文件加载到内存，并对数据进行校验，转换解析和初始化，最终形成可以被虚拟机直接使用的Java类型，这个过程被称作虚拟机的类加载机制。
 
## 7.2 类加载的时机
- 一个类型从被加载到虚拟机内存中开始，到卸载初内存为止，它的整个生命周期将会经历加载、验证、准备、解析、初始化、使用和卸载七个阶段，
其中验证、准备、解析三个部分统称为连接。这七个阶段的发生顺序如图7-1所示。
![类的生命周期](./pictures/类的生命周期.png)  
- <div style="text-align: center;">图7-1 类的生命周期 </div>
- 对于初始化阶段，《Java虚拟机规范》严格规定了有且只有六种情况必须对类进行“初始化”：
  - 1）遇到new、getstatic、putstatic或invokestatic这四条字节码指令时，如果类型没有进行过初始化，则需要先触发其初始化阶段。能够生成这四条指令的典型Java代码场景有：
    - 使用new关键字进行实例化对象的时候。
    - 读取或设置一个类型的静态字段（被final修饰、已在编译期把结果放入常量池的静态字段除外）的时候。
    - 调用一个类型的静态方法的时候。
  - 2）使用java.lang.reflect包的方法对类型进行反射调用的时候，如果类型没有进行过初始化，则需要先触发其初始化。
  - 3）当初始化类的时候，如果发现其父类还没有进行过初始化，则需要先触发其父类的初始化。
  - 4）当虚拟机启动时，用户需要指定一个要执行的主类（包含main()方法的那个类），虚拟机会先初始化这个类。
  - 5）当使用JDK7新加入的动态语言支持时，如果一个java.lang.invoke.MethodHandle实例最后的解析结果为REF_getStatic、REF_putStatic、REF_invokeStatic、REF_newInvokeSpecial四种类型的方法句柄，并且这个方法句柄对应的类没有进行过初始化，则需要先触发其初始化。
  - 6）当一个接口中定义了JDK8新加入的默认方法（被default关键字修饰的接口方法）时，如果有这个接口的实现类发生了初始化，那该接口要在其之前被初始化。
- 以上六种场景中的行为称为对一个类型进行主动引用。除此之外，所有引用类型的方式都不会出发初始化，称为被动引用。以下三点说明何为被动引用。
  - 1）通过子类引用父类的静态字段，不会导致子类初始化。
  - 2）通过数组定义来引用类，不会触发类的初始化。
  - 3）常量在编译阶段会存入调用类的常量池中，本质上没有直接引用到定义常量的类，因此不会触发定义常量类的初始化。  
  
## 7.3 类的加载过程
### 7.3.1 加载
- 在加载阶段，Java虚拟机需要完成以下三件事情：
  - 1）通过一个类的全限定名来获取定义此类的二进制字节流。
  - 2）讲这个字节流所代表的静态存储结构转化为方法区的运行时数据结构。
  - 3）在内存中生成一个代表这个类的java.lang.Class对象，作为方法区这个类的各种数据的访问入口。
- 对于数组类而言，情况就有所不同，数组类本身不通过类的加载器创建，它是由Java虚拟机直接在内存中动态构造出来的。但数组类与类加载仍然有很密切的关系，
因为数组类的元素类型最终还是要靠类加载来完成加载，一个数组类（下面简称为C）创建的过程遵循以下规则：
  - 如果数组的组件类型是引用类型，那就递归采用本节中定义的加载过程去加载这个组件类型，数组C将被标识在加载该组件类型的类加载器的类名称空间上。
  - 如果数组的组件类型不是引用类型（例如int[]数组的组件类型为int），Java虚拟机将会把数组C标记为与引导类加载器关联。
  - 数组类的可访问性与它的组件类型的可访问性一致，如果组件类型不是引用类型，它的数组类的可访问性将默认为public，可被所有的类的接口访问到。
- 加载阶段结束后，Java虚拟机外部的二进制字节流就按照虚拟机所设定的格式存储在方法区之中了，方法区中的数据存储格式完全由虚拟机实行自定义。
类型数据妥善安置在方法区之后，会在Java堆内存中实例化一个java.lang.Class类的对象，这个对象将作为程序访问方法区中的类型数据的外部接口。
  
### 7.3.2 验证
- 验证是连接阶段的第一步，这一阶段的目的是确保Class文件的字节流中包含的信息符合《Java虚拟机规范》的全部约束要求，
保证这些信息被当作代码运行不会危害虚拟机自身的安全。
- 验证阶段大致上会完成下面四个阶段的检验动作：文件格式验证、元数据验证、字节码验证和符号引用验证。

#### 1.文件格式验证  
- 第一阶段要验证字节流是否符合Class文件格式的规范，并且能被当前版本的虚拟机处理。这一阶段可能包括下面这些验证点：
  - 是否以魔数0xCAFEBABE开头。
  - 主、次版号是否在当前Java虚拟机接受范围之内。
  - 常量池的常量中是否有指向不存在的常量或不符合类型的常量。
  - CONSTANT_Utf8_info型常量中是否不符合UTF-8编码的数据。
  - Class文件中各个部分及文件本身是否有被删除的或附加的其他信息。
  - ......
- 该验证阶段的主要目的是保证输入的字节流能正确地解析并存储于方法区之内，格式上符合描述一个Java类型信息的要求。
这阶段的验证是基于二进制字节流进行的，只有通过了这个阶段的验证之后，这段字节流才被允许进入Java虚拟机内存的方法区中进行存储，
所以后面的三个验证阶段全部都是基于方法区的存储结构上进行的，不会再直接读取、操作字节流了。

#### 2.元数据验证
- 第二阶段是对字节码描述的信息进行语义分析，以保证其描述的信息符合《Java语言规范》的要求，这个阶段可能包括的验证点如下：
  - 这个类是否有父类（除了java.lang.Object之外，所有的类都应当有父类）。
  - 这个类的父类是否继承了不允许被继承的类（被final修饰的类）。
  - 如果这个类不是抽象类，是否实现了其父类或接口之中要求实现的所有方法。
  - 类中的字段、方法是否与父类产生矛盾（例如覆盖了父类的final字段，或者出现不符合规则的方法重载，例如方法参数都一致，但返回值类型却不同等）。
  - ......
- 第二阶段的主要目的是对类的元数据信息进行语义校验，保证不存在与《Java语言规范》定义相悖的元数据信息。

#### 3.字节码验证
- 第三阶段主要目的是通过数据流分析和控制流分析，确定程序语义是合法的、符合逻辑的。该阶段对类的方法体（Class文件中的Code属性）进行分析，
保证被校验类的方法在运行时不会做出危害虚拟机安全的行为，例如：
  - 保证任意时刻操作数栈的数据类型与指令代码序列都能配合工作，例如不会出现类似于"在操作栈放置了一个int类型的数据，
  使用时却按long类型来加载入本地变量表中"这样的情况。
  - 保证任何跳转指令都不会跳转到方法体以外的字节码指令上。
  - 保证方法体中的类型转换总是有效的，例如可以把一个子类对象赋值给父类数据类型，这是安全的，但是把父类对象赋值给子类数据类型，
  甚至把对象赋值给与它毫无继承关系、完全不相干的一个数据类型，则是危险和不合法的。
  - ......
  
#### 4.符号引用验证
- 最后一个阶段的校验行为发生在虚拟机将符号引用转化为直接引用的时候，这个转化动作将在连接的第三阶段————解析阶段中发生。
符号引用验证可以看做是对类自身以外（常量池中的各种符号引用）的各类信息进行匹配性校验，通俗来说就是，
该类是否缺少或者被禁止访问它依赖的某些外部类、方法、字段等资源。本阶段通常需要校验下列内容：  
  - 符号引用中通过字符串描述的全限定名是否能找到对应的类。
  - 在指定类中是否存在符合方法的字段描述符及简单名称所描述的方法和字段。
  - 符号引用中的类、字段、方法的可访问性（private、protected、public、<package>)是否可被当前类访问。
  - ...
  

### 7.3.3 准备
- 准备阶段是正式为类中的变量（即静态变量，被static修饰的变量）分配内存并设置类变量初始值的阶段。
- 关于准备阶段，有两个概念需要着重强调，首先是这时候进行内存分配的仅包括类变量，而不包括实例变量，
实例变量将会在对象实例化的时候随着对象一起分配在Java堆中。其次是这里所说的初始值"通常情况"下是数据类型的零值，假设一个类变量定义为：
```java
  public static int value = 123;
```
- 那变量value在准备阶段过后的初始值为0而不是123，因为这时尚未开始执行任何Java方法，而把value赋值为123的putstatic指令是程序编译后，
存放于类构造器<clinit>()方法之中，所以把value赋值为123的动作要到类的初始化阶段才会被执行。
- 某些"特殊情况"下初始值不是零值：如果类字段的字段属性表中存在ConstantValue属性，那在准备阶段变量值就会被初始化为ConstantValue属性所指定的初始值，
假设上面类变量value的定义修改为：
```java
  public static final int value = 123;
```
- 编译时Javac将会为value生成ConstatntValue属性，在准备阶段虚拟机就会根据ConstantValue的设置将value赋值为123。


### 7.3.4 解析
- 解析阶段是Java虚拟机讲常量池内的符号引用替换为直接引用的过程。直接引用与符号引用有什么关联关系？
- 符号引用：符号引用以一组符号来描述所引用的目标，符号可以是任何形式的字面量，只要使用时能无歧义地定位到目标即可。
符号引用与虚拟机实现的内存布局无关，引用的目标并不一定是已经加载到内存及内存当中的内容。各种虚拟机实现的内存布局可以各不相同，
但是它们能接受的符号引用必须都是一致的，因为符号引用的字面量形式明确定义在《Java虚拟机规范》的Class文件格式中。
- 直接引用：直接引用是可以直接指向目标的指针、针对偏移量或者是一个能间接定位到目标的句柄。直接引用是和虚拟机实现的内存布局直接相关的，
同一个符号引用在不同虚拟机实例上翻译出来的直接引用一般不会相同。如果有了直接引用，那引用的目标必定已经在虚拟机的内存中存在。
- 对同一个符号引用进行多次解析请求是很常见的事情，除invokedynamic指令以外，虚拟机实现可以对第一次解析的结果进行缓存，
譬如在运行时直接引用常量池中的记录，并把常量标识为已解析状态，从而避免解析动作重复进行。无论是否真正执行了多次解析动作，
Java虚拟机都需要保证的是在同一个实体中，如果一个符号引用之前已经被成功解析过，那么后续的引用解析请求就应当一直能够成功；
同样地，如果第一次解析失败了，其他指令对这个符号的解析也应该收到同样地异常，哪怕这个请求的符号在后来已成功加载进Java虚拟机内存之中。
- 解析动作主要针对类或接口、字段、类方法、接口方法、方法类型、方法句柄和调用点限定符这7类符号引用进行。


### 7.3.5 初始化
- 直到初始化阶段，Java虚拟机才真正开始执行类中编写的Java程序代码，将主导权移交给应用程序。
- 初始化阶段就是执行类构造器<clinit>()方法的过程。<clinit>()并不是程序员在Java代码中直接编写的方法，它是Javac编译器的自动生成物，
但我们非常有必要了解这个方法具体是如何产生的，以及其可能影响程序运行行为的细节。
  - <clinit>()方法是由编译器自动收集类中的所有变量的赋值动作和静态语句块（static{}块）中的语句合并产生的，
  编译器收集的顺序是由语句在源文件中出现的顺序决定的，静态语句块中只能访问到定义在静态语句块之前的变量，定义在它之后的变量，
  在前面的静态语句块可以赋值，但是不能访问，如代码清单7-5所示。
- 代码清单7-5 非法前向引用变量
```java
public class Test {
    static {
        i = 0; // 给变量复制可以正常编译通过
        System.out.println(i); // 这句编译会提示"非法向前引用"
    }
    static int i = 1;
}
```