---
title: "记一次MySQL删库的恢复"
date: 2020-12-22T13:51:28+08:00
draft: false
tags:
- mysql
- c++
- python
- reverse engineering
categories:
- mysql
summary: 一次误操作导致整个数据库被drop，怎么恢复？
---

# 背景
最近正在做语音相关的工作，所有的语料数据都放在一个mysql数据库里面，同事一次误操作导致整个数据库被drop，里面包括了全部的文本语料和词库。
数据库放在一台linux机器上，文件系统是XFS。

# 从磁盘恢复数据

说实话，之前没有相关经验，只有先google一下看看有没有现成可以参考的解决方案。在[stackexchange](https://dba.stackexchange.com/questions/23251/is-there-a-way-to-recover-a-dropped-mysql-database/72760)上看到有人提到[TwinDB data recovery toolkit](https://github.com/twindb/undrop-for-innodb)，提到**先需要停掉mysql，并停止一切写入**，赶紧systemctl stop mysql，umount掉disk。

MySQL 5.5之前是所有数据都放在ibdata1文件里面的，因此被删除的数据还在ibdata里面，而5.6之后是放在单独的目录里面，drop database/table是直接删除文件的。我们用的是5.8，不能直接扫描ibdata1，需要扫描整个磁盘，同时也意味着其他OS的写入也很可能导致数据被覆盖掉。不过既然是删除了文件，可以先试试能不能从xfs里面恢复删除的文件，找到一个[工具](https://github.com/ianka/xfs_undelete)，扫描了一遍磁盘，没能够找到对应的ibd文件。

先下载到本地编译
```bash
git clone https://github.com/twindb/undrop-for-innodb
cd undrop-for-innodb
make
```

使用stream_parser扫描整个磁盘
```bash
./stream_parser /dev/sdc1
```
会在当前目录下生成一个文件夹pages-XXX，XXX为扫描的文件名，我这里生成的是pages-sdc1，里面包含了FIL_PAGE_INDEX和FIL_PAGE_BLOB两个目录。

使用c_parser得到表corpora的ID，0000000000000001.page是存放SYS_TABLES的page文件
```bash
./c_parser -4f pages-sdc1/FIL_PAGE_INDEX/0000000000000001.page -t dictionary/SYS_TABLES.sql | grep corpora

000000000504    85000001320110  SYS_TABLES      "corpora"  54      4       1       0       0       ""      1
```


因为InnoDB的数据是放在primary index上的，通过获取primary index的ID，就能够得到存放数据的page文件。0000000000000003.page是存放SYS_INDEXES的page文件
```bash
./c_parser -4f pages-sdc1/FIL_PAGE_INDEX/0000000000000003.page -t dictionary/SYS_INDEXES.sql | grep 54

000000000504    85000001320178  SYS_INDEXES     54      112      "PRIMARY"       1       3       1       3
```

找到了corpora的数据是在0000000000000112.page文件里面的
在恢复数据之前需要先确定表结构，还好我们知道，创建一个包含corpora DDL的sql文件放到corpora.sql

开始恢复数据
```bash
mkdir -p  dumps/default
./c_parser -6f pages-sdc1/FIL_PAGE_INDEX/0000000000000112.page \
    -t corpora.sql > dumps/default/corpora 2> dumps/default/corpora.sql
```

运行完成后，看看是否恢复成功
```bash
head -100 dumps/default/corpora
```
结果是数据不全，而且还有很多错误数据，看来已经被覆盖掉了不少page。尝试恢复另外两张表也没什么收获。难道只能从删库到跑路了？

# 从backup恢复
之前有做一个定时backup的脚本，每天半夜进行backup，赶紧到backup的机器上看看，结果发现最近一次的backup是3个月前的了。执行
```bash
crontab -l
```
是有对应的命令的，查了一下，发现crontab最后必须有一行空行才能正确执行，先改掉。

# 从solr恢复
还好文字语料在solr中是有cache的，用于加速查询，赶紧连接到solr中，dump到文件中
```bash
curl "http://solr-server/solr/corpora/select?q=*:*&wt=csv&indent=true&rows=26159501" -o corpora.csv
```
有了csv文件，写一个python脚本直接导入即可。


# 从内存中恢复
但是还有词表没恢复，这张表是没有solr缓存的，怎么办？
还好，我们有一个在线分词查询程序，会将词表cache到内存中，这是一个python写的http server，分词是使用c++实现的，运行在windows上。

词表保存在一个hash table里面，结构如下：
```c++
struct Word
{
    unsigned char   nbytes;   /* number of bytes */
    char            length;   /* number of characters */
    unsigned int    freq;
    char            text[word_embed_len];
};
struct Entry
{
    Word *word;
    Entry *next;
};
static Entry **bins = static_cast<Entry **>(std::calloc(init_size,
                                            sizeof(Entry *)));
```
从代码可以看出来，用的是[Seperated chaining](https://en.wikipedia.org/wiki/Hash_table#Separate_chaining)方式存放的，如下图：

![Hash table](/img/Hash_table.png)

怎么从内存中还原词表呢？
* [ReadProcessMemory()](https://docs.microsoft.com/en-us/windows/win32/api/memoryapi/nf-memoryapi-readprocessmemory)
* [MiniDump文件](https://en.wikipedia.org/wiki/Core_dump#MINIDUMP)

我选择第二个，使用[Process Explorer](https://docs.microsoft.com/en-us/sysinternals/downloads/process-explorer)保存一个full dump文件。

## minidump文件格式
不像其他文件格式，MS对minidump文件的格式描述还是很清楚的，在[这里](https://docs.microsoft.com/en-us/windows/win32/api/minidumpapiset/)可以找到。有人已经给我们写好了[python解析minidump的库](https://github.com/skelsec/minidump)，我们只需要找到全局变量数组bins，然后遍历这个数组找出所有的Word即可。

## 定位数组
在windows中，一个进程是有自己独立进程空间的，这个进程依赖的DLL都会作为module被加载到这个进程的地址空间中来，我们在minidump中可以找到这个DLL所在的base address，然后在这个DLL中定位到这个变量所在偏移量，两者相加即可得到这个变量在整个进程空间中的地址。

minidump中包含了所有module的相关信息，而变量的偏移地址，就需要借助一些其他方法来获取了。如果有PDB文件，则比较容易，但是我们这个DLL的PDB文件也没有保存下来，我们只能想其他办法。

我使用IDA Pro，打开这个DLL，通过exported function，和容易定位到这个操作这个变量的函数，然后在这个函数中找到这个全局变量的位置，然后用这个地址减去IDA定义的base address，就得到了这个变量的offset。

有了offset，接下来就是遍历hash table了。这里我使用了python的struct库来进行数据的数据的读取。

## 读取minidump
minidump这个库给我们提供了方便进行内存位置读取的方法，简化后的代码如下：
```python
from minidump.minidumpfile import MinidumpFile
from minidump.minidumpreader import MinidumpFileReader
import struct

INT_SIZE = 4
PTR_SIZE = 8

class Word:
    def __init__(self, reader, ptr):
        self.reader = reader
        self.ptr = ptr
        buf = reader.read(ptr, 8)
        # ignore padding between length and freq
        self.nbytes, self.length, padding, self.freq = struct.unpack('<BBHL', buf)
        if self.nbytes > 0:
            self.text = reader.read(ptr + 8, self.nbytes).decode('utf8')
        else:
            self.text = ''

    def __str__(self):
        if self.nbytes > 0:
            return 'text: {}, freq: {}'.format(self.text, self.freq)
        else:
            return '<EMPTY>'

class Entry:
    def __init__(self, reader, ptr):
        self.reader = reader
        self.ptr = ptr
        self.word = None

        buf = reader.read(ptr, 2 * PTR_SIZE)
        self.pword, self.pnext = struct.unpack('<QQ', buf)
        if self.pword != 0:
            self.word = Word(reader, self.pword)

    def __str__(self):
        if self.pword != 0:
            return '{}, pnext: {:x}'.format(self.word, self.pnext)
        else:
            return 'pnext:{:x}'.format(self.pnext)

    @staticmethod
    def size():
        return 2 * PTR_SIZE

def main():
    mp = MinidumpFile.parse('python.dmp')
    reader = MinidumpFileReader(mp)

    dll = reader.get_module_by_name('XXX.dll')
    baseaddr = dll.baseaddress
    # get offset from IDA Pro
    p_n_bins = baseaddr + 0x1EA58
    pp_bins = baseaddr + 0x1FD60
    p_bins = struct.unpack('<Q', reader.read(pp_bins, PTR_SIZE))[0]

    n_bins = struct.unpack('<L', reader.read(p_n_bins, INT_SIZE))[0]


    print('p_n_bins: {:x}, n_bins: {:x}'.format(p_n_bins, n_bins))
    for i in range(n_bins):
        pentry = struct.unpack('<Q', reader.read( p_bins + i * PTR_SIZE, PTR_SIZE))[0]
        if pentry != 0:
            entry = Entry(reader, pentry)

            while entry.pnext != 0:
                entry = Entry(reader, entry.pnext)


if __name__ == '__main__':
    main()
```

# 总结
这次恢复虽然成功了，还是暴露了我们很多问题的：
1. 数据库的权限问题，导致很容易就能删库跑路
2. 数据库的备份没有起作用