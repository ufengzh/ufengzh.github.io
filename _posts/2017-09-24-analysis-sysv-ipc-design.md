---
layout: post
title:  "从Key冲突分析SysV IPC设计缺陷"
date:   2017-09-24 14:32:00 +0800
categories: Linux
---
## 前言

后台开发在IPC操作中比较常使用共享内存，最近恰巧有同学在操作共享内存时发现数据一直对不上，折腾一段时间后发现是Key冲突问题。沟通后确认使用的是System V共享内存接口，再加上后台测试环境各种服务混合部署，出现Key冲突也不足为怪，如果用的是POSIX接口就不会出现这么怪异的情况了。Key都冲突了，更别提进程会如何内存越界访问等各种异常退出问题了，哈，这是为什么呢？

## SysV接口

`SysV`是[System V](https://en.wikipedia.org/wiki/UNIX_System_V)简称，Unix操作系统众多版本中的一种，最初由[AT&T](https://en.wikipedia.org/wiki/AT%26T)开发。SysV IPC包含消息队列、信号量、共享内存三大进程间通信接口。

SysV IPC部分主要接口：

```cpp
# 获取消息队列ID
int msgget(key_t key, int msgflg);

# 获取信号量ID
int semget(key_t key, int nsems, int semflg);

# 获取共享内存ID
int shmget(key_t key, size_t size, int shmflg);
```

这些接口有个共性，都需要一个*key_t*类型的*key*值用于唯一标识操作对象，`key_t`在Linux平台的定义就是一个有符号32位整数，即`int32_t`。这个*key*可以手动指定，或者通过`ftok`接口获取，直接传整数值比较容易记错，后者通过文件系统路径获取整数*Key*值更易使用，接下来将从glibc分析下*ftok*实现。

```cpp
key_t ftok(const char *pathname, int proj_id);
```

## 源码分析

`ftok`是libc库函数，glibc实现源码*sysvipc/ftok.c*（为方便查阅部分代码有调整）：

```cpp
key_t ftok(const char *pathname, int proj_id)
{
	struct stat st;

	if (stat(pathname, &st) < 0)
		return (key_t)-1;

	/**
	 * st_dev: device id
	 * st_ino: inode number
	 */
	return (key_t) ((proj_id & 0xff) << 24 | (st.st_dev & 0xff) << 16 | (st.st_ino & 0xffff));
}
```

通过源码可以看出：
1. 要求传入路径必须存在，以便*stat*系统调用获取文件路径状态信息。
2. *proj_id*虽然是*int*，实际只用到低8位，一个字节，就是一个*char*。
3. 返回IPC Key（四个字节）计算方式如下表（Little-Endian表示）：

| ftok | 第4字节 | 第3字节 | 第2字节 + 第1字节 |
| ---- | -------:| ------: | ------: |
| IPC Key | *proj_id*低8位 | 设备ID低8位 | Inode低16位 |

`ftok`实现看似比较合理，实际却隐藏一定缺陷：
1. 实现缺陷：`可能会产生重复Key`，如果调用*ftok*第二个参数*proj_id*是0，而且传入路径都在同一个文件系统当中（同一分区设备ID相同），那么当文件系统路径足够多时（超过$2^{16}$），就肯定会产生重复Key（接下来我们会重现这个场景）。
2. 设计缺陷：`依赖已经存在的系统路径却无法做到彼此一一对应`，*Key*值是32位整数，而文件系统路径由设备*Device ID*、*Inode Number*唯一定位（这是远超32位数值的），后面我们对比*POSIX*设计的共享内存接口就很好理解了。

恰恰是SysV对*Key ID*这样的设计缺陷导致`ftok`实现无法做到完美。

> 数据类型：
> Device ID：*unsigned long int*
> Inode Number：*unsigned long int*

## 重复Key

前面已经提到过，当*proj_id*为0、同一分区文件系统路径足够多（超过$2^{16}$）就必然出现重复Key，利用*Shell*脚本就能重现这一场景（也可以选择*C/C++*）。

> 伪码原理：
> 1. 生成 > $2^{16}$个文件。
> 2. 通过命令*stat*获取文件设备ID、Inode Number，生成IPC Key。
> 3. 通过*awk*或其他文本处理工具找到重复IPC Key。
> 
> 原理较简单，实现可自行适当优化。

*Shell*脚本简单实现*dup_ipc_key.sh*：

```bash
CURR_DIR=$(pwd)
WORD_DIR=/home/xianfengzhu/projects/sh/work
FILE_NUM=$((2 ** 16))
KEY_FILE=ipc_key.txt

function get_ipc_key()
{
    local st_dev=$1
    local st_ino=$2
    local proj_id=0

    echo $(((proj_id & 0xff) << 24 | (st_dev & 0xff) << 16 | (st_ino & 0xffff)))
}

# 0. Initial
mkdir -p $WORD_DIR
cd $WORD_DIR

# 1. Create files
echo "start generate files..."
start_time=$(date +%s)
for ((i = 0; i <= $FILE_NUM; i++))
do
    touch src_file.${i}
    st=$(stat -c '%d,%i' src_file.${i})
    st_dev=${st%,*}
    st_ino=${st#*,}

    ipc_key=$(get_ipc_key $st_dev $st_ino)
    echo "${ipc_key},src_file.${i}" >> $KEY_FILE
done

end_time=$(date +%s)
echo "generate files done, cost $((end_time - start_time)) seconds"

# 2. Find duplicated ipc key
awk -F ',' '{
    if (length(key_map[$1]) > 0) {
        dup_cnt += 1;
        printf("duplicated key: %s, files: %s, %s\n", $1, $2, key_map[$1]);
    } else {
        key_map[$1] = $2;
    }
}
END {
    printf("total duplicated count: %d\n", dup_cnt);
}' $KEY_FILE

cd $CURR_DIR
```

执行脚本：

```plain
xianfengzhu@Tencent64 ~/p/sh> ./dup_ipc_key.sh
start generate files...
generate files done, cost 217 seconds
duplicated key: 62501, files: src_file.19372, src_file.0
duplicated key: 62503, files: src_file.19374, src_file.1
duplicated key: 62504, files: src_file.19375, src_file.2
duplicated key: 62505, files: src_file.19376, src_file.3
......
total duplicated count: 101542
```

结果表明：当*proj_id*为0、同一分区当中，`(64K + 1)`个文件竟然出现重复Key的次数超过10万，没想到你是这样的`ftok`，好失望。

## V.S POSIX

既然SysV IPC设计缺陷很容易引起`ftok`计算出重复Key问题，那么有没有其他更好的IPC接口呢？POSIX共享内存就不存在这样的问题。

简单介绍下POSIX共享内存主要接口：

```cpp
# 打开或创建共享内存对象
int shm_open (const char *name, int oflag, mode_t mode);

# 关闭共享内存对象（当所有打开进程都关闭时才销毁共享内存对象）
int shm_unlink (const char *name);

# 设置共享内存Size
int ftruncate (int fd, off_t length);

# 映射共享内存至当前进程地址空间
void *mmap (void *addr, size_t length, int prot, int flags,
            int fd, off_t offset);
```

POSIX共享内存操作步骤：
1. 通过*shm_open*打开或创建共享内存对象，返回一个文件描述符。
2. 通过*ftruncate*传入文件描述符，设置共享内存大小。
3. 通过*mmap*映射共享内存至当前进程地址空间，返回共享内存地址。
4. 通过共享内存地址执行读、写操作。

为什么POSIX解决了共享内存Key冲突问题呢？
> 1. *shm_open*通过文件路径生成文件描述符，经由*mmap*映射共享内存对象，本质上是通过文件定位一个共享内存ID，只要文件路径唯一，即能操作指定共享内存。
> 2. 而*SysV*的*ftok*接口看似由文件路径定位*Key*值，实际并没有使用完整的Device ID、Inode Number（这两者唯一定位一个文件）数据，从而可能生成重复*Key*值。

`shm_open`在glibc实现`sysdeps/posix/shm_open.c`（为方便查阅部分代码有调整，*shmfs*作为文件系统一般会挂载在*/dev/shm*，用户也可能挂载在其他路径）：

```cpp
/* Open shared memory object.  */
int shm_open (const char *name, int oflag, mode_t mode)
{
#define SHMDIR ("/dev/shm/")
    const char *shm_dir = SHMDIR;
    size_t shm_dirlen = sizeof(SHMDIR);

    /* Construct the filename.  */
    while (name[0] == '/')
    {
        ++name;
    }

    size_t namelen = strlen(name) + 1;

    /* Validate the filename.  */
    if (namelen == 1 || namelen >= NAME_MAX || strchr(name, '/') != NULL)
    {
        __set_errno(EINVAL);
        return -1;
    }

    /* Copy the shm path */
    char *shm_name = __alloca(shm_dirlen + namelen);
    memcpy(shm_name, shm_dir, shm_dirlen);
    memcpy(shm_name + shm_dirlen, name, namelen);

    oflag |= O_NOFOLLOW | O_CLOEXEC;

    int fd = open(shm_name, oflag, mode);
    if (fd == -1 && __glibc_unlikely(errno == EISDIR))
    {
        /* It might be better to fold this error with EINVAL since
           directory names are just another example for unsuitable shared
           object names and the standard does not mention EISDIR.  */
        __set_errno(EINVAL);
    }

    return fd;
}
```

通过*shm_open*实现可发现：
1. 传入的路径可以是以`/`开头的普通字符串（*/my_shm_path*），中间不能包含`/`，唯一即可，最终转换成路径*/dev/shm/my_shm_path*。
2. 共享内存ID由文件系统路径唯一定位，路径不同标识不同共享内存，`不会出现不同路径标识同一共享内存对象`。
3. 对比SysV由*ftok*接口生成的*key_t*，字符串对接口使用者更`友好`，而且不易出错。

既然POSIX操作的是文件系统路径，那么就可以直接使用`lsof`、`stat`这样的命令直接查看共享内存对象了。

下面的`lsof`命令显示有PID为*31116*进程*shm_test*打开了共享内存对象*/dev/shm/my_shm_path*，大小为2048个字节。对比`ipcs -m`输出，在一堆整数ID中查找具体进程打开的SysV共享内存对象，`lsof`显然清晰明了多了。

```bash
ufeng@ubuntu ~/p/c> lsof /dev/shm
COMMAND    PID  USER  FD   TYPE DEVICE SIZE/OFF NODE NAME
shm_test 31116 ufeng mem    REG   0,21     2048    3 /dev/shm/my_shm_path
ufeng@ubuntu ~/p/c> 
ufeng@ubuntu ~/p/c> stat /dev/shm/my_shm_path
  File: '/dev/shm/my_shm_path'
  Size: 2048            Blocks: 8          IO Block: 4096   regular file
Device: 15h/21d Inode: 3           Links: 1
Access: (0644/-rw-r--r--)  Uid: ( 1002/   ufeng)   Gid: ( 1002/   ufeng)
Access: 2017-09-24 11:30:59.473758864 +0800
Modify: 2017-09-24 11:30:59.473758864 +0800
Change: 2017-09-24 11:30:59.473758864 +0800
 Birth: -
```

SysV共享内存操作工具`ipcs`：

```bash
xianfengzhu@mmsearchtest1[qq]:~> ipcs -m 
------ Shared Memory Segments --------
key        shmid      owner      perms      bytes      nattch     status
0x110130f3 622611     qspace     666        102400     2
0x110130f7 655380     qspace     666        102400     4
0x13457283 688149     qspace     666        50628952   1
0x13457284 720918     qspace     666        50628952   1
```

## 总结

通过对比分析SysV、POSIX共享内存接口（共有的优点不谈）：

| 共享内存  |     SysV |   POSIX  |
| :-------- | :------- | :------ |
| ID标识    | 32位整数 | 文件系统路径（如*/my_shm_path*） |
| 接口友好  | 整数不友好 | 文件系统路径容易配置使用 |
| Key冲突   | 设计缺陷导致`ftok`可能生成重复Key | 文件系统路径唯一定位，字符串路径正确就不会重复 |
| 操作工具  | `ipcs`显示整数ID，分辨困难 | `lsof`显示字符串路径及进程ID，查看进程打开共享内存对象一目了然 |

总而言之，推荐使用POSIX共享内存。如果你对SysV情有独钟，那么在使用时请注意：
1. 使用`ftok`尽量传入不同数值的*proj_id*参数。
2. 可选择在共享内存头部使用部分字节存储`幻数`进行校验。
