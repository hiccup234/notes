
### SRAM 与 DRAM
    SRAM：静态存储器，一个bit数据需要6~8个晶体管来存储，电路简单访问快速，但存储密度不高，常用来做CPU的高速缓存（L1/L2/L3）。
    DRAM：动态存储器，由一个晶体管+一个电容来存储一个bit，但是电容会不断漏电，所以要求定时刷新充电才能保持数据不丢失，常用来做内存。
    
### SSD 与 HDD
    SSD(Solid-state disk)：SLC(Single-Level Cell)、MLC、TLC、QLC    PE擦写    闪存转换层（FTL）    预留空间7-15%    写入放大   
    HDD(Hard Disk Drive)：机械硬盘，磁盘
    基于SSD的k-v数据库：AeroSpike

### 局部性原理与缓存
    CPU与高速缓存L1/L2/L3的映射关系：
        1、直接映射，通常采用mod运算（实际是h&(n-1)，类似HashMap），缓存行中用组标记（Tag）存储取模截取后留下的高位，来解决取模冲突的问题。
        2、全相连映射
        3、组相连映射
    高速缓存写策略：
        1、写直达（Write-Through），即就算Cache Block里有数据，也每次都要写入主内存（类似volatile的效果）
        2、写回（Write-Back），即判断是否Cache Block有数据且原数据是否为脏数据，如果是把源数据写回内存，并把当前数据写入Cache Block，并标记为脏数据。
    ```
    int[] arr = new int[64 * 1024 * 1024];
    // 循环 1
    for (int i = 0; i < arr.length; i++) arr[i] *= 3;
    // 循环 2（Cache Line的大小一般为64Byte，所以这里步长为16）
    for (int i = 0; i < arr.length; i += 16) arr[i] *= 3
    ```如上程序，循环1与循环2的执行时间相差不大
    
### 缓存一致性原理：
    1、写传播，CPU1写L1后需要传播到CPU2的L1
    2、事务串行化，同时有两个核心写L1，写的值不同，最后传播到其他核心的顺序就可能不同，所以要“加锁”来串行化
    `总线嗅探机制` 和 `MESI协议`：
    1、总线嗅探就是把所有的读写请求都通过总线Bus广播给所有CPU核心，让每个核心嗅探这些请求并做出处理
    2、MESI写失效协议，只有一个CPU核心写数据，写入之后广播 “失效” 请求告诉其他CPU核心，
        还有一种写广播，即将写请求广播到所有CPU核心，同时更新所有核心里的Cache Block。写广播需要广播写的数据，所以更占带宽
        M：代表已修改（Modified），Cache Block中数据已修改还未写回内存
        E：代表独占（Exclusive），表示该数据只有当前CPU核心加载到了缓存里，其他CPU核心都未加载，
                               此时如果从总线收到对应数据的读请求，则证明其他CPU核心也从内存加载了，所以要改成共享状态
        S：代表共享（Shared），此时如果要更新缓存数据，则需要先广播其他核心将对应数据变成失效状态之后才能写当前缓存
        I：代表已失效（Invalidated），Cache Block中数据已不可用，需要重新从主内存加载
    
    注：MESI协议不仅仅局限于CPU Cache的场景，其还适用于所有缓存及一致性同步的场景。
    
### 虚拟内存 & 保护内存
    虚拟内存地址怎么映射到物理内存地址？
    1、简单页表：每个进程都要维护一张映射表，32位系统占用4MB内存，过于浪费（完整的映射整个虚拟内存地址）
    2、多级页表：页表树，按需添加多级页表，8M进程页表占用大概9KB空间，但是多级页表解析时需要多次访问内存，导致性能降低
    解决办法：缓存，TLB 地址变换高速缓冲（Translation-Lookaside Buffer），TLB又分为指令缓存和数据缓存
    内存安全保护：
    可执行空间保护（Executable Space Protection）
    地址空间布局随机化（Address Space Layout Randomization）

### IO_WAIT & IOPS
    1、性能调试工具
        top: 查看wa指标（wait average）, 即CPU等待IO操作完成花费的时间占CPU总时间的百分比
        iostat: 查看tps指标（硬盘的IOPS）
        iotop: 查看IO占用高的进程
        stress -i 2 : 模拟IO压力
    2、普通机械硬盘IOPS(随机访问)只有100左右，SSD的IOPS能达到2W甚至4W
            机械硬盘随机访问时间 =  平均延时 + 寻道时间
            2000-2010这10年间，谷歌通过Partial Stroking技术（减少寻道时间，但磁盘可以空间会变小）来提升IOPS，
            从而支撑起互联网的蓬勃发展
    3、SSD不能经常做磁盘碎片整理（磨损均衡FTL、TRIM、写入放大）
        
### DMA（Direct Memory Access）
    IO设备的升级，HDD -> SSD 、SATA SSD -> PCI Express SSD，使得SSD的IOPS达到2W、4W等
    但CPU主频2GHz，即每秒20亿次操作，跟IO设备仍然有巨大差距。
    
    DMA是一个协处理器，用来解耦CPU与IO设备
    
    Kafka利用DMA来实现零拷贝，调用Java NIO库的FileChannel的transferTo方法，直接在硬盘的读缓冲区和网卡的缓冲区交换数据，
    即数据交换都在内核态，没有内核态和用户态的数据拷贝，大约有65%的性能提升。
    
### 零拷贝（Zero-Copy） https://developer.ibm.com/articles/j-zerocopy/
    Kafka里有两种常见的海量数据传输
        1、从网络中接收上游数据，然后落盘到本地磁盘中持久化
        2、从本地磁盘读取数据，然后通过网络发送出去
        
    下面代码包括了四次传输（文件读写是由操作系统内核态完成）
    ByteBuf buf = new ByteBuf();
    File.read(file, buf, len);      // 1、从硬盘通过DMA读取到操作系统内核的缓冲区中 2、从内核缓冲区复制到程序内存中（buf）
    Socket.send(socket, buf, len);  // 3、从程序内存中复制到操作系统的Socket的缓冲区中 4、从Socket缓冲区复制到网卡缓冲区再发送出去（DMA方式）
    
    // 零拷贝   
    public long transferFrom(FileChannel fileChannel, long position, long count) throws IOException {
        return fileChannel.transferTo(position, count, socketChannel);
    }
    
    Nginx可以通过配置sendfile的开关on/off来控制以zero-copy运行
    
    
    零拷贝实现方式：mmap+write方式，sendfile方式
    
    零拷贝的概念比较宽泛，比如在linux中的零拷贝一般指如下三个理念：
    
    1、减少甚至避免操作系统内核和用户程序地址空间这两者之间进行数据拷贝，这一般是需要操作系统提供的系统调用实现的，比如mmap()、splice()等，
    可以这样做事因为很多应用程序并不需要处理数据，只需要将磁盘中的静态数据比如文件转发到另外一个服务器上。
    
    2、内核旁路，也就是最大可能的让用户态的程序可以直接与设备进行通信，绕开内核的直接io，这种情况有时候也需要内核进行一定的辅助参与到控制其中，
    但核心理念是能够让数据不经过内核缓冲区，可以从设备直接到达用户，这种实现方式一般需要定制的硬件网卡才行。通常云数据库上会采取这种技术来加速数据的提取。
    
    3、内核缓冲区和用户缓冲区之间的传输优化。所以零拷贝的概念是比较宽泛的，并不是完全不经过cpu。很多时候也需要cpu进行辅助控制。
    
### 单比特反转
    硬件层面有时候也会出现错误，如内存的质量问题引起漏电，外部射线的电磁干扰等，从而引起单比特反转，解决方案有：
    1、奇偶校验，用额外的一个比特位来记录数据中1的个数是奇数个还是偶数个，即校验码位。
       优点：实现简单、额外占用空间少、性能高（O(n)的遍历即可）
       缺点：只能发现奇数个反转的问题，且只能发现错误，不能自动纠正错误
    2、ECC，Error-Correcting-Code memory
       海明码：常见7-4海明码
       
### 分布式计算（把人类的大脑联网）
    1、垂直扩展（升硬件）、水平扩展（加硬件）
    2、高可用 -- 健康检查、单点故障、故障转移、异地多活
    3、一致性
    
    