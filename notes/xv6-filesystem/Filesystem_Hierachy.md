<img src="img.assets/filesystem_hierachy.png" alt="filesystem_hierachy" style="zoom:50%;" />

- 物理磁盘，可持久化存储文件
- buffer cache，缓存了磁盘中的盘块，避免频繁读取磁盘
- logging，文件系统的持久性
- inode cache，缓存使用到的inode
- file，管理不同类型文件，文件描述符
- syscall，文件系统接口