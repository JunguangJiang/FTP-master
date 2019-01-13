本文的石墨文档地址：[https://shimo.im/docs/0ZKUwhjFsqMIHTR2/](https://shimo.im/docs/0ZKUwhjFsqMIHTR2/) 《FTP大作业 stage2报告》
# 使用说明
首先进入server/src文件夹，运行 
```
sudo ./server -port 2000 -root /tmp
```
其中-port指定了控制连接的端口号，-root指定了将哪个文件夹用于客户端的根目录。然后进入client/src文件夹，按照client/doc中的说明运行客户端。
# 代码说明
以下列出核心代码的路径和功能。
```
.
├── client
│   └── src
│       ├── client.py：实现ftp客户端的核心逻辑
│       ├── cmd.py： ftp的命令行程序
│       ├── gui.py： ftp的GUI程序
│       ├── mainwindow.ui： 使用Qt Designer设计的客户端界面
│       ├── mainwindow.py： 由mainwindow.ui生成的python文件
│       ├── task.py：基于多线程实现数据传输类
│       ├── task_manager.py：基于工厂模式管理数据传输任务
│       └── util.py：在实现ftp客户端时常用的函数库
├── server
│   └── src
│       ├── Makefile 
│       ├── handle.c： 对不同的ftp命令进行处理
│       ├── handle.h
│       ├── server：ftp服务器程序
│       ├── server.c： 启动ftp服务器，并开始等待服务器的连接
│       ├── util.c： 在实现handle.c时常用的函数库
│       └── util.h：ftp服务器常用结构体和函数的声明
└── udp
```
# 算法说明
## 断点续传算法
为了实现断点续传，在服务器端增加了REST、APPE、STAT等命令。过程如下（只列出思路，忽略了文件大小的校验）。
当客户端进行断点下载时：
    1. 客户端向服务器发送REST命令；
    2. 服务器根据REST命令得知该从文件的哪个位置开始读取数据；
    3. 客户端发送RETR命令，开始文件下载。

当客户端进行断点上传时：
    1. 客户端向服务器发送STAT命令，获得服务器上相应文件的信息，从中提取出文件大小remote_file_size；
    2. 客户端将文件读指针指向remote_file_size；
    3. 客户端发送APPE命令，开始文件的上传。
## 对多客户端的支持
采用多进程的方式。在主进程中监听客户端的请求，为每个请求分配一个新的进程。由于每个登录请求只会被分配一个进程，因此在进行文件传输时，不会响应客户端新的控制命令。而其余客户端对应着其他进程，因此它们的控制命令和数据传输不会受到影响。
## 对大文件传输的支持
由于服务器采用了多进程的方式，因此当客户端A开始数据传输时，客户端B可以发送其他的命令。对于客户端B而言，服务器是不阻塞的。虽然客户端A在进行数据传输时不能发送其他命令。
在GUI中进行大文件的数据传输，主要有两方面的要求：
* GUI界面不能卡死
* 能够向服务器继续发送其他命令，包括开始新的数据传输

为了保证GUI界面不卡死，我将数据传输任务放到一个新的进程中（具体实现见task.py TaskThread)，并定期检查进程是否结束。
为了实现并行的数据传输，以及在数据传输时继续向服务器发送命令，我分配给每个数据传输任务一个新的ftp连接（具体实现见task.py TransferTask)。
由于每个数据传输任务独占一个ftp连接，因此彼此可以互不干扰地同时工作。
但是频繁创建新的ftp连接增加了网络消耗，因此在GUI程序和数据传输任务之间增加了一层（传输任务工厂 TaskManager)，由TaskManager进行传输任务的创建和回收。数据传输任务执行过程图如下，具体原理见“对传输任务的管理”。
![图片](https://images-cdn.shimo.im/WrrPkx5E4VckMGvd/屏幕快照_2018_11_16_下午8.03.36.png!thumbnail)
## 对传输任务的管理
当GUI需要执行一个新的数据传输任务时，其实是向TaskManager的等待队列waiting_queue中插入新任务的相关信息（TaskInfo)。
TaskManager在适当的时候从等待队列中取出任务，并交给TransferTask执行。这样GUI创建的多个传输任务可以重复利用一个ftp连接。同时TaskManager也可以设置数据传输的最大并行度。具体算法如下：

设同时运行的最大任务数为max_occurs。
设置两个队列,空闲任务队列free_queue（存储可用的TransferTask）和等待队列waiting_queue（存储尚未执行的任务的TaskInfo)，初始时均为空。
当需要执行一个新任务时，
若当前TransferTask总数（正在执行的+空闲的）已经超过max_occurs,则将需要执行的新任务信息加入等待队列中；
若空闲任务队列中非空，则从中取出一个TransferTask，并直接开始运行新任务；
若空闲任务队列为空，则创建一个新的TransferTask，并直接开始运行新任务；

当回收一个TransferTask时，
若当前等待队列非空，从队首取出任务信息TaskInfo，并让回收的TransferTask开始执行；
否则，将回收的TransferTask加入到空闲队列中，
检查：若空闲队列中的任务数大于max_occurs，则不断关闭空闲队列中的队首任务，以保证空闲队列中的任务个数不超过max_occurs。

采用TaskManager的额外好处是，可以判断新创建的任务是否与当前正在运行或者等待的任务相冲突。例如数据传输时的源文件或者目标文件是否相同（具体处理见task_manager.py is_task_valid)。由于支持了并行的数据传输，因此判断任务是否冲突是非常重要的。
## 文件夹的传输
在实际使用中，文件夹传输能够大大提高使用效率。文件夹传输的实现主要有两种思路。
### 方案1 递归遍历文件夹
优点：没有使用额外的FTP指令，文件夹传输功能能够兼容其他ftp服务器。
缺点：当文件夹中文件数量较多但是每个文件都不大时，由于每个小文件都需要单独进行传输，传输效率非常低。
### 方案2 压缩文件夹后再进行传输
优点：传输效率较高。
缺点：引入了额外的FTP指令，并且在文件夹下存在较少文件，并且每个文件较大时，会花费较多时间用于数据的压缩和解压。

最终我采用了方案2。引入了ZIP和UNZIP指令用于向服务器发送压缩和解压的命令。在client.py中实现了上传文件夹put_folder和下载文件夹get_folder。以put_foler为例介绍这个过程：
1. 压缩本地文件夹
2. 将压缩后的文件上传到远程服务器
3. 向远程服务器发送UNZIP命令解压文件
4. 删除远程服务器的压缩文件
5. 删除本地的压缩文件
## 进度计算与传输速率计算
在大文件传输时，文件当前的传输进度以及传输速率对于使用者有较大的价值。
为了能够获取文件传输进度，GUI会定时查询本地文件和远程文件的大小，然后针对每个不同的任务计算其进度。具体公式如下：
| 传输任务类型   | 进度   | 
|:----|:----|
| get,reget   | 本地文件大小/远程文件大小   | 
| put,reput   | 远程文件大小/本地文件大小   | 
| append   | 远程文件大小增长量/本地文件大小   | 
| get_folder   | 本地压缩文件大小/远程压缩文件大小   | 
| put_folder   | 远程压缩文件大小/本地压缩文件大小   | 

通过记录被传输文件的大小增长，我们可以计算总传输速率
| 传输方向   | 传输速率   | 
|:----:|:----:|
| 传输方向   | 传输速率   | 
| 传输方向   | 传输速率   | 
| 传输方向   | 传输速率   | 
| 上传速率   | 所有正在执行的任务的远程文件大小增长量/单位时间   | 
| 下载速率   | 所有正在执行的任务的本地文件大小增长量/单位时间   | 


# 存在的局限性
* 服务器和客户端进行传输时不支持含有空格的文件名。
* zip和unzip在服务器端未使用多线程，当压缩和解压缩文件较大时会服务器无法接受响应当前客户端的其他控制命令。
* 在进行数据传输时，如果修改数据传输个数和数据传输模式（Passivee或Port）会导致无法预料的结果。


