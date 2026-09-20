# Ubuntu基本设置
## Windows与Ubuntu跨系统复制粘贴
实现Windows与Ubuntu跨系统复制粘贴板上内容，也实现Ubuntu窗口自适应
```
sudo apt-get autoremve open-vm-tools
sudo apt-get install open-vm-tools
sudo apt-get install open-vm-tools-desktop
```
## 安装C语言基本软件
安装Vim
```
sudo apt install vim
```
安装gcc
```
sudo apt install gcc
```
## Ubuntu与主机互传文件
打开某一个文件夹，然后从主机文件夹将文件拖动过去，即可将文件从主机复制到虚拟机上

### 共享文件夹
选择虚拟机配置设置，点击"选项"。先点击"共享文件夹"，再点击"总是启用"，最后点击添加。按步骤建立共享文件夹
这一步要选择一个Windows路径，也就是一个共享文件夹，在这个路径里的文件，虚拟机和Windows都能访问
启动虚拟机之后, 点击 虚拟机-> 安装VMware tools。打开文件管理，书签栏这里多了一个VMware Tools, 进入文件夹, 有五个文件, 其中一个是压缩包。将把压缩包移动到任意一个目录下，解压压缩包。**进入到目录**之后执行 .pl 文件
```text
# 执行 .pl 文件
$ ./ vmware-install.pl
```
程序安装完成之后，末尾会出现"Enjoy, --the VMware team"
配置好之后, 在根目录有个mnt文件夹, mnt文件夹里有个hgfs文件夹, hgfs文件夹就是共享文件夹

# Ubuntu问题处理
## 连不上网络
* 选择与主机使用Nat连接
* 确认服务中Vmware Nat正在运行
* 在网络适配器中Vmware 两个虚拟网卡都有正常运行
* 重启NetworkManager网络
	* 使用nmcli重启
		```
		sudo nmcli networking off
		sudo nmcli networking on
		```
		或者是
		```
		sudo service network-manager restart
		```
	* 编辑NetworkManager文件
		```
		sudo vi /etc/NetworkManager/NetworkManager.conf
		```
		将其中的managed=false改为manafed=true
		修改文件时，用delete键删除false，再按 i 键进入编辑模式，编辑好后，Esc推出编辑，Shift+冒号，光标定位到最下面，输入wq(退出并保存），再回车就行
		重启网络
		```
		sudo service network-manager restart
		```
* 删除NetworkManager缓存文件
	```
	service NetworkManager stop
	rm /var/lib/NetworkManager/NetworkManager.state
	service NetworkManager start
	```