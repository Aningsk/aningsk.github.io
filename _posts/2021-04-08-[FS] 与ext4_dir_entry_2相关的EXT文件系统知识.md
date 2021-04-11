# 1. 数据结构的定义
```c
struct ext4_dir_entry_2 {
	__le32 inode;
	__le16 rec_len;
	__u8 name_len;
	__u8 file_type;
	char name[EXT4_NAME_LEN];
};
```

# 2. 数据结构的含义
inode: 目录项对应的 inode 编号  
rec\_len: 目录项的长度，即此结构体在 block 中的长度(单位:字节)    
name\_len: 目录名长度  
file\_type: 文件类型(目录的类型为 0x02)  
name: 保存目录名(长度最大 255 字节)  

# 3. 一些命令注记
```bash
df -i .
```
显示当前位置的磁盘中 inode 使用情况。

```bash
du -h
```
显示当前目录占用的磁盘空间(可以接参数指明具体文件)。

```bash
sudo debugfs /dev/nvme0n1p4
```
在/dev/nvme0n1p4 分区中使用 debugfs，之后进入 debugfs 的交互终端。   
输入 stat \<path/file\>  
可以查看:    
**ctime** – 修改时间(change)指文件的修改，比如权限   
**atime** – 访问时间(access)   
**mtime** – 修改时间(modify)指内容的修改  
**crtime** – 创建时间(create)可能这是 EXT 文件系统查看创建时间的唯一方法  
**EXTENTS** – 显示\<path/file\>有几个数据块(block)以及这些数据块的编号
例如   
(0):9249 意为只有一个数据块(0)其编号为 9249
(0-17): 164187648-164187665 意为 18 个数据块(0-17)及其编号

```bash
sudo hexdump -C /dev/nvme0n1p4 -s 36996k -n 4096
```
**-C** : Canonical hex+ASCII display 显示十六进制数据，及其对应的 ASCII 码  
**/dev/nvme0n1p4** : 这里用块设备作为输入源  
**-s \<num\>** : 从输入源跳过\<num\>数量的字节，可以带有 k、m 等，可以写十进制或十六进制  
 **-n \<num\>** : 显示\<num\>数量的字节内容  

# 4. 关于目录的inode
目录作为文件的一种，也会对应一个 inode，这个 inode 的编号被记录在 ext4\_dir\_entry\_2 中。 从下面这副图可以看出，每创建一个目录，当前分区中的可用 inode 数量减一。  
![ext4-dir-entry-2/df-i.png]({{ site.baseurl }}/images/ext4-dir-entry-2/df-i.png)

# 5. 查看ext4\_dir\_entry\_2在磁盘上的存储
## 1)首先确定想看的目录项在哪个数据块中 使用 debugfs
例如我们想看/home/下某个目录的 ext4\_dir\_entry\_2    
下图显示的是:块设备/dev/nvme0n1p4(挂载在/home)的当前位置的状态(stat .)，即 /home 的状态。  
![ext4-dir-entry-2/debugfs-stat.png]({{ site.baseurl }}/images/ext4-dir-entry-2/debugfs-stat.png)

可以得知/home 占用了一个数据块，该数据块编号为 9249，即/dev/nvme0n1p4 的第 9249个数据块。

## 2) 使用 hexdump 输出这个数据块的内容
我们想看的是第 9249 个数据块的内容，一个数据块大小为 4096 字节;所以，hexdump 要 跳过 36996KB，如下图:   
![ext4-dir-entry-2/hexdump.png]({{ site.baseurl }}/images/ext4-dir-entry-2/hexdump.png)

在/home 下，已有的目录是:lost+found，aningsk，temp\_dir，1234，123 。我们可以 在这里看到这些/home 下的目录项，其存在顺序，就是创建顺序。其实除了这几个目录外， 还有“.”和“..”，只是从 ASCII 里看不出来，简单列出来是:  
![ext4-dir-entry-2/hexdump-table.png]({{ site.baseurl }}/images/ext4-dir-entry-2/hexdump-table.png)

在 rec\_len 这一项中，可以看到最后一个目录项的 rec\_len 非常大(0x0FA8)。这其实是 EXT 文件系统的特性:最后一个目录项的长度都会占满所在的数据块。如下图，最后一项的长度始终占满。  
![ext4-dir-entry-2/more.png]({{ site.baseurl }}/images/ext4-dir-entry-2/more.png)

# 6. 关于删除
删除一个目录，其实对应的目录项依然存在。但是，如果删除的是最后一个目录项，那么新的“最 后一个目录项”还是有点变化的:最后一个始终保持占满当前 block。  
如下图，删除中间某一个目录项，没有发生变化。  
![ext4-dir-entry-2/del-1.png]({{ site.baseurl }}/images/ext4-dir-entry-2/del-1.png)

如下图，删除最后一个，新的最后一个发生了变化;但已被删除的旧的最后一项的数据还存在。  
![ext4-dir-entry-2/del-2.png]({{ site.baseurl }}/images/ext4-dir-entry-2/del-2.png)

在 EXT 文件系统中的删除操作，其实并不会移除相关文件的数据，只是将对应的 inode 标记为未使用(上图并没有体现 inode 的变化)。对于 ext4\_dir\_entry\_2 的数据，在删除过程中也不会 回收，这里是为了在这个 block 中保持所有的目录项是连续首尾相接的“链表”，中间的目录项 对应的文件被删除了，为了保证这个“链表”不断掉，所以这个 ext4\_dir\_entry\_2 依然存在。虽然 block 中的这段空间没有释放，但在新建其他文件时，这个区域是会被重复利用的。如下图:   
![ext4-dir-entry-2/touch-new.png]({{ site.baseurl }}/images/ext4-dir-entry-2/touch-new.png)

但是，这种首位相接的链表形式会导致性能问题:一个目录中存在大量的文件，中间的文件删除后，为了找到之后的文件，依然要遍历中间那些已经删除掉的目录项，这样就会降低性能。实际上，现在的文件系统已经解决了这个问题:当一个目录下的文件(目录项)大于一个 block 所能容纳的，就会使用 HASH 树来索引目录项，不再是 ext4\_dir\_entry\_2 了，而是 dx\_root。这里暂不描述，可以参考:  

[Understanding EXT4 (Part 6): Directories][1]  中的“Large Directories”章节  


Aningsk   
2021/04/08

[1]:	https://www.sans.org/blog/understanding-ext4-part-6-directories/ "Understanding EXT4 (Part 6): Directories"