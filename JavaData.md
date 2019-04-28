# Java 备忘点

## 无符号类型的处理

### 如果在Java中进行流(Stream)数据处理，可以用DataInputStream类对Stream中的数据以Unsigned读取

Java在这方面提供了支持，可以用java.io.DataInputStream类对象来完成对流内数据的Unsigned读取

### 利用Java位运算符，完成Unsigned转换。

    //将data字节型数据转换为0~255 (0xFF 即BYTE)。
    public int getUnsignedByte (byte data){       
        return data&0x0FF;
    }
    //将data字节型数据转换为0~65535 (0xFFFF 即 WORD)。
    public int getUnsignedByte (short data){      
        return data&0x0FFFF;
    }
    //将int数据转换为0~4294967295 (0xFFFFFFFF即DWORD)。
    public long getUnsignedIntt (int data){       
        return data&0x0FFFFFFFFl;
    }

## 内存管理（堆、栈、方法区）

### 方法区

>方法区是各个线程共享的内存区域，它用于存储已被虚拟机加载的类信息、常量、静态变量、即时编译器编译后的代码等数据 （重点）。  
>相对而言，垃圾收集行为在这个区域比较少出现，但并非数据进了方法区就永久的存在了，这个区域的内存回收目标主要是针对常量池的回收和对类型的卸载。  
>运行时常量池：它用于存放编译期生成的各种字面量和符号引用。

实例分析：
    
    Foo foo = new Foo();
    foo.f();

以上代码的内存实现原理为：
 >1.Foo类首先被装载到JVM的方法区，其中包括类的信息，包括方法和构造等。  
 >2.在栈内存中分配引用变量foo。  
 >3.在堆内存中按照Foo类型信息分配实例变量内存空间；然后，将栈中引用foo指向foo对象堆内存的首地址。  
 >4.使用引用foo调用方法，根据foo引用的类型Foo调用f方法。  
 
 ## 并发
 [Java并发编程与高并发](http://naotu.baidu.com/file/6808ea88451b49ba4964e2c81d0d2c8b?token=3a5de17f2ea7220d "Java并发编程与高并发解决方案")
