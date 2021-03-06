# OS_Linux_FileSystem_Design
- Design my filesystem based on 100M space

- Develop Environment --Xcode C++


 ## 磁盘空间分配
 
100M存储空间分配（block 1kB） 102400块 块0-块102399

 模块|分配|备注
 ---|:--:|---:
 superblock|块0（1块）
 inode 88字节,每块11个inode|块1-1000(1000块）
 Inode位图|块1001-1002(2块）
 free_block位图|块1003-1015(13块）
 data_block|块1016-102399(101384块)99M


## inode(FCB结构)

- inode包含文件的元信息，具体来说有以下内容：
   - 文件的字节数
   - 文件拥有者的User ID
   - 文件的Group ID
   - 文件的读、写、执行权限
   - 文件的时间戳，共有三个：ctime指inode上一次变动的时间，mtime指文件内容上一次变动的时间，atime指文件上一次打开的时间。
   - 链接数，即有多少文件名指向这个inode
   - 文件数据block的位置
  
   注：目录操作模块和inode模块都维护了一个X_sb指针指向超级块，从而可以获得整个文件系统的元数据信息。
   
   块指针分配 int i_zone
   
  指针|类型|备注
  ---|:--:|---:
  0-9|直接指针
  10|一级间接索引指针
  11|二级间接索引指针
  12|三级间接索引指针
  
   inode大小：88字节
   空闲inode位示图


## superblock

  目录操作模块和inode模块都维护了一个X_sb指针指向超级块，从而可以获得整个文件系统的元数据信息 [done]


## dirent 目录结构

  文件的文件名，以及该文件名对应的inode号码
  具体操作：
   - 获取路径
   - 显示子目录
   - 判断是否为目录
   - 设置或者更换路径（父节点）
   - 删除子节点（删除一个目录项只能在其父节点处删）
   - 设置子节点


## freeblock.h 成组链接法


### 思想：
   - 空闲盘块栈用来存放当前可用的一组空闲盘块的盘块号（最多含100个号），以及栈中尚有的空闲盘块号数N，顺便指出，N兼做栈顶指针使用，栈是临界资源，系统设置一把锁供进程互斥访问。其中，S.free(0)是栈底，栈满时栈顶为S.free(99)。
   - 文件区中的所有空闲盘块被分成若干个组，如每100个盘块作为一组。
   - 将每一组含有的盘块总数N和该组所有的盘块号记入其前一组的第一个盘块S.free(0)~S.free(99)中，这样，由各组的第一个盘块可链接成一条链。
   - 将第一组的盘块总数和所有的盘块号记入空闲盘块号栈中，作为当前可供分配的空闲盘块号。
   - 最末一组只有99个盘块，其盘块号分别记入其前一组的S.free(1)~S.free(99)中，而在S.free(0)中则存放0，作为空闲盘块链的结束。
   
### 分配操作
   - 首先检查空闲盘块号栈是否上锁，如未上锁，便从栈顶取出一空闲盘块号，将与之对应的盘块分配给用户，然后将栈顶指针下移一格。
   - 若该盘块号已是栈底，即S.free(0)，这是当前栈中最后一个可分配的盘块号。将栈底盘块号所对应盘块的内容读入栈中，作为新的盘块号栈的内 容，并把原栈底对应的盘块分配出去(其中的有用数据已读入栈中)。
   - 然后，再分配一相应的缓冲区(作为该盘块的缓冲区)。最后，把栈中的空闲盘块数减1并返回。
   
### 回收操作
   - 将回收盘块的盘块号记入空闲盘块号栈的顶部，并执行空闲盘块数加1操作。
   - 当栈中空闲盘块号数目已达100时，表示栈已满，便将现有栈中的100个盘块号，记入新回收的盘块中，再将其盘块号作为新栈底
   

## open & control 打开文件结构

   - 每个file结构体都指向一个file_operations结构体，这个结构体的成员都是函数指针，指向实现各种文件操作的内核函数。注：release而不叫close是因为它不一定真的关闭文件，而是减少引用计数，只有引用计数减到0才关闭文件。
   - 每个file结构体都有一个指向dirent结构体（目录项）的指针。open、stat等函数的参数的是一个路径。
   - 每个dirent结构体都有一个指针指向inode结构体。
   - inode结构体有一个指向super_block结构体的指针。
   - file、dirent、inode、super_block这几个结构体组成了VFS的核心概念
   

## file结构体

   - file_operations
   - dirent


## user 用户结构


## 总要求

 - 多用户、多级目录、用户登录
 - 建立文件存储介质的管理机制
 - 文件系统功能（显示目录、创建、删除、打开、关闭、读、写）
 - 文件操作接口（显示文件、创建、删除、打开、关闭、读、写）
 - 权限控制、读写保护等其他功能


### 具体设计ops

 - 目录：ls mkdir rm cd ll
 - 权限/用户：chmod、login、logout、register（root下）注：root是最早init写死了的
 - 文件：create(创建文件)、delete（删除文件）、show（显示文件）、 write、 read、 open、 close
 - 解析命令


## 分层：

- 块
    - 分配
    - 回收
- inode
    - 创建
    - 删除
- 目录（成链条）
    - Dirent root:path root
    - Dirent parent_path/filename
- 文件
    - 先选出一个inode
    - 再创建dirent
    - 打开则是指向dirent（根据路径层层链表遍历找）
- 用户（权限）

**********************************************************************************
## 任务进度

任务进度|code|test
--|:--:|--:
Design mem|[done]|---
superblock|[done]|[done]
datablock|[done]|[done]
inode|[done]|[done]
dirent|[done]|[done]
filesystem|[done]|[done]
user|[done]|[done]


**********************************************************************************
## Reference:
<a href="https://withcic.cn/2018/03/09/FileSystem/index.html">Filesystem</a>

<a href="https://www.jianshu.com/p/f98ae1e289cb">Freeblock</a>

<a href="https://blog.csdn.net/u014379540/article/details/53456070">Open</a>

<a href="https://www.cnblogs.com/huxiao-tee/p/4657851.html">Other</a>

**********************************************************************************
       
