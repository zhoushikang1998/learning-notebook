- VMware 配置启动共享文件夹

  - 总是启用
  - 添加文件夹：路径/名称
  - 启用此共享

- 虚拟机如果没有发现共享文件夹，执行

  ```bash
  /usr/bin/vmhgfs-fuse .host:/ /mnt/hgfs -o subtype=vmhgfs-fuse,allow_other,nonempty
  ```

- 加入开机自启动

  ```bash
  # 修改文件
  vim /etc/rc.d/rc.local
  
  # 添加
  /usr/bin/vmhgfs-fuse .host:/ /mnt/hgfs -o subtype=vmhgfs-fuse,allow_other,nonempty
  ```

- 建立共享文件夹的软连接

  ```bash
  ln -s /mnt/hgfs/gopath /usr/local/services/gopath
  ```

  