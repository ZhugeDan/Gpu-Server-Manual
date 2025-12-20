

此次经历，全程在gemini的帮助下完成。这是[聊天记录](https://gemini.google.com/share/ad342085855f)，附带全部细节。另有[参考文献](https://xungejiang.com/2022/07/14/lxd-new/)。

分为部署，使用（管理员和用户），可能遇到的问题，容器表共四个文件，以及接下来的QA。



0. 整体结构

	​	  
	
	```
		   ubuntu
	     	 |虚拟
	     Lxd模板（视情况而定，可以有不同的模板）
	         |复制
	    +----+----+----+
	    |    |    |    |
	  usr1 usr2 usr3 usr4
	         |创建
	      +--+----+
	      |       |
	    env1 ... env4
	```
	
	




1. **Q:** **为什么要引入lxd？lxc加上conda虚拟环境，相当于两层隔离**。

   **A:** 核心答案：Conda 只能隔离“软装”（Python包），LXD 隔离的是“硬装”（操作系统）。

   - 如果只用 Conda (裸机共享)：

     - 场景： 大家都在宿主机上。张三想装 CUDA 11，李四想装 CUDA 12。显卡驱动只能装一个版本，冲突！
     - 风险： 张三想删自己的文件，手滑敲了 `sudo rm -rf /`，整个服务器挂了，李四和王五的数据全部陪葬。
     - Conda 的局限： Conda 只能管 Python，管不了 `gcc` 版本、`apt` 安装的系统库、环境变量等。

   - 加上 LXD (容器隔离)：

     - 场景： LXD 给每个人分配了一套独立的 Ubuntu 系统（相当于每人一间公寓）。

     - 优势：

       1. 安全（最重要）： 张三在他的容器里把系统删光了，宿主机毫发无损，李四完全不受影响。
       2. 环境彻底解耦： 张三可以用 Ubuntu 18.04 + CUDA 10，李四用 Ubuntu 22.04 + CUDA 12，互不干涉。

     - 结论： LXD 是为了保护服务器不被学生搞崩，也是为了让学生互不干扰。

       

2. **Q: lxd的模板，复制后可以共享给其他人什么？****

   **A** :**核心答案：它共享的是“精装修样板房”的初始状态。**

   当你用“模板容器”创建“张三”时，张三得到的不是一个空壳，而是：

   1. 预装软件： 已经装好的 Vim, Git, Wget 等工具。
   2. 预设配置： 已经配好的清华源（Conda/Pip/Apt）、已经修好的 SSH 密码登录权限。
   3. 驱动能力： 已经打通的显卡驱动路径。

   **注意：** 复制完成的那一刻起，张三的容器就独立了。张三以后在里面装的文件，不会出现在“初始模板”里，也不会出现在“李四”那里。

   

3. **Q: 如果不同的人使用不同的容器，如果下载相同的包，会重复下载占用空间吗？我windows的conda，不同虚拟环境的包可以共享，而不同容器间可以共享吗？**

   **A: 会重复占用（但在服务器上，这点空间成本可以忽略）。**

   -  在 Windows 的 Conda 里，不同环境的 PyTorch 可能会通过硬链接共享。但在 LXD 中，容器 A 和容器 B 的文件系统是隔离的。

     - 如果张三装了 PyTorch (2GB)，李四也装了 PyTorch (2GB)，硬盘上确实会占用 4GB。

   - 为什么这么做？

     - 以空间换自由： 这种隔离保证了张三把 PyTorch 里的文件改坏了，完全不会影响李四。
     - LXD 的黑科技 (ZFS)： 虽然 Conda 包会重复，但基础操作系统文件（那几百兆的 Ubuntu 系统文件）是共享且不占空间的（Copy-on-Write 技术）。只有当张三修改文件时，才会真正占用新空间。

   - na公共数据集如何操作?

     #### 方案 A：公共数据集 (The "Library" Strategy) 

     对于 MNIST, CIFAR, ImageNet, COCO 这种大家都用的数据集，千万不要每个人传一份！

     做法： 在宿主机建一个公共仓库，然后“挂载”到每个人的容器里。

     操作步骤：

     1. 在宿主机建立仓库：

        Bash

        ```
        # 在宿主机找个大硬盘位置
        mkdir -p /home/ubuntu/shared_datasets
        
        # 下载数据进去 (比如把刚才的 MNIST 放进去)
        cp -r ./data/MNIST /home/ubuntu/shared_datasets/
        ```

     2. 挂载给所有容器 (只读模式！)：

        为了防止张三手滑把公共数据删了，我们要以 Read-Only (ro) 模式挂载。

        - 挂载给现有容器 (如 liren)：

          **Bash**

          ```
          # 语法：lxc config device add <容器名> <设备名> disk source=<宿主机路径> path=<容器内路径> readonly=true
          lxc config device add liren common-data disk source=/home/ubuntu/shared_datasets path=/mnt/datasets readonly=true
          ```

        - 挂载给模板 (Founding Titan V2)：

          这样以后新建的“王五”、“赵六”天生自带这个数据盘！

          Bash

          ```
          lxc config device add founding-titan-v2 common-data disk source=/home/ubuntu/shared_datasets path=/mnt/datasets readonly=true
          ```

     3. 学生如何使用？

        告诉学生：“代码里读取数据的路径改成 /mnt/datasets/MNIST 即可，不用下载！”

     ------

     #### 方案 B：私有数据集 (The "Ant Move" Strategy)

     对于学生自己的小作业、私有数据。

     做法： 也就是我们之前用的 SCP。

     1. 学生在自己电脑上操作：

        **PowerShell**

        ```
        # 把本地的 my_data.zip 传给 liren
        scp -P 10001 my_data.zip liren@192.168.137.203:~/
        ```

     2. 在容器里解压：

        Bash

        ```
        unzip my_data.zip
        ```

     

4. **Q: 给不同容器分配显卡，这个操作是后续可以调整？权限是普通用户还是管理员？**

   **A: 非常灵活，可以随时调，推荐“全给+口头协调”模式。**

   1. 分配是否固定？
      - 不是固定的。 是一行命令的事。
      - 你现在给张三分了所有卡 (`gpu gpu`)。
      - 如果张三太霸道，你可以随时在宿主机改：`lxc config device add zhangsan gpu0 gpu id=0`（只给他第 0 张卡）。无需重启，立即生效。
   2. 我们想要的效果（每个人看得到所有卡，自己决定用哪张）：
      - 目前的状态就是这样！
      - 因为我们用了 `lxc config device add ... gpu gpu`（把所有 GPU 映射进去）。
      - 现象： 张三输入 `nvidia-smi`，能看到 3 张卡。李四输入，也能看到 3 张卡。
      - 使用逻辑（君子协定）：
        - 大家在群里喊一声：“我用卡 0 跑两天哈！”
        - 然后在代码里指定：`CUDA_VISIBLE_DEVICES=0 python train.py`。
        - 如果不指定，代码可能会默认跑在卡 0 上，两人就会抢资源导致变慢。