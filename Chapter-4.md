#### DHCP与PEX:IP怎么来的，又是怎么没的

&nbsp;&nbsp;&nbsp;&nbsp; 通过上一章我们了解到，如果需要与其它机器通信，我们就需要一个通信地址，分配给对应的网卡。那么，如何将IP分配给网卡呢？学完这一张
你就可以使用ifcongfig 或 ip addr给机器手动分配一个地址了。先看配置命令：
```
sudo ifconfig eth1 10.0.0.1/24
sudo ifconfig eth1 up

如果使用iproute2

sudo ip addr 10.0.0.1/24 dev
sudo ip link set up eth1

```

配置的命令其实很简单，但是要注意的是分配正确的IP才能实现互通。例如，旁边的机器都是：192.168.1.x 你自己非要配置：16.158.23.6 根据上一张讲到的包出去的过程
我们可以知道，他会找不到这个IP，那么就收不到包，就实现不了互通。第二节我们说过：网络包一定是完整的，可以有下层没上层，但不能相反。因为这两个IP的网段是不一样
样的，不一样则不会发送ARP请求，那么就不会加MAC头，这就形成跨网段调用，就不会把包发送到网络中，而是企图发送到网关，虽然网关能找到，能确定目标IP就是自己，但
是却没有填写MAC地址，所以网关拒收。如果没有网关，包根本发布出去，因为它要发送给网关，但又没有网关，这就冲突了。所以手动配置的时候，一定要配置合适的IP，不同
的系统配置文件的格式不同，但是无非就是CIDR，子网掩码，广播地址和网关地址。

&nbsp;&nbsp;&nbsp;&nbsp; 动态主机配置协议（DHCP），配置ip的门道还是挺多的，ip配置以后一般不变，这是服务端机器还好，一直在一个网段内。但客户端的机器经常
在不同的环境，连不同的wifi，那不可能去一个环境就配置一次IP吧，那估计要疯了。所以，我们需要一个自动配置的协议，就是DHCP。有了这个协议，网管就非常轻松了，
他只需要配置一段共享的IP，每台机器都通过DHCP协议去共享IP中申请IP地址，然后自动配置，走了后又还回去，供其他机器使用。打个比方：数据中心的服务器，ip一旦配
置，基本不会变，就像自己买了房，自己装修。DHCP就像租房，不用装修，都帮你配好，租完后把房退了就行。

&nbsp;&nbsp;&nbsp;&nbsp;那么DHCP的具体工作方式是怎样的呢？当一台新机器加入一个网络后，什么都没有，也什么都不知道，只有自己的MAC地址，这时，通信基本靠
吼，“我来了，有人嘛”，这就是DHCP Discover。新来的机器使用0.0.0.0 发送一个广播包，目标地址为255.255.255.255。广播包封装了UDP，UDP封装了BOOTP，DHCP就是
BOOTP的增强版，如果去抓包，你可能看到包里面的协议还是BOOTP协议。在广播包里：我是Boot request,我的MAC地址是XXXX，能否租一个IP给我，我要上网。基本包括：
MAC头，IP头(新人使用0.0.0.0)，UDP头，BOOTP头，content(租IP)。如果网络中有DHCP server，他就会知道来了一个新人，需要给他一个IP，这里MAC地址的唯一性就非常
重要了，因为DHCP server是通过MAC地址去判断是否是新人的。分配的过程称为DHCP Offer，同时，DHCP server会保存提供的IP给哪个MAC的信息，保证不会重复分配IP。
DHCP server发送的广播包：MAC头，IP头，UDP头，BOOTP头（boot reply），包括分配的IP，子网掩码等各种信息。如果有多个DHCP server，很可能同时收到多个广播包，
新机器会选择其中一个，一般是先到达的那个，然后想网络发送一个DHCP request广播包，包括MAC，租用的IP，是那个DHCP server提供的IP，这样其余的DHCP server就
撤销分配，选中的就记录此IP已经分配，但这个过程中，IP租用还没得到DHCP server 的二次认可，所以会继续使用0.0.0.0，当server 收到后，回复一个ACK，表示接受客户
的选择，最终达成一致，同时广播一下，让网段的机器都知道。

&nbsp;&nbsp;&nbsp;&nbsp;  在客户机使用离开或使用期限到期后，管理员就会回收IP，如果到期了，你还要继续使用，那么客户机会提前一段时间告诉DHCP-server，还需要继续使用当前IP，客户机一般在租期过去50%的时候，向DHCP-server发送DHCP request，客户机会等待回来的ACK，会根据包里提供的新租期和其他TCP/IP
参数更新自己的配置，这就完成了IP租用更新。一切看起来很完美，DHCP大部分人也知道，但里面还有一个细节，很多人不会注意，但很有意思：网管不仅能自动分配IP
地址，还能帮你自动安装操作系统。

&nbsp;&nbsp;&nbsp;&nbsp; 接下就来了解一下PXE，预启动执行环境。普通的笔记本一般不会有这种需求，因为拿到的时候，OS已经安装好了，但数据中心不一样，网管可能一下子拿到几百上千台空机器，这要一个个的去安装OS，那肯定会疯的。所以网管不仅仅分配IP，还需要自动安装操作系统。先安装OS，然后分配IP，这就可以登上直接使用了。那怎么自动安装OS呢？这个过程和操作系统的启动过程有点像，首先，启动BIOS。这是一个特别小的小系统，只能干特别小的一件事，其实就是读取硬盘的MBR启动扇区，将GRUB启动起来，GRUB加载内核，加载作为根文件系统的inittramfs文件，然后将全力交给内核，最内核启动，初始化整个操作系统。那么，我们安装操作系统的过程，只能在BIOS启动之后，因为没有安装系统前，扇区都还有，所以这个过程称为预启动执行环境（PXE）。

&nbsp;&nbsp;&nbsp;&nbsp;PXE协议分为客户度和服务端，由于没有OS，只能把客户端放BIOS里面，当计算机启动时，BIOS把PXE客户端调入内存中，就可以连接到服务端做一些操作了。开始，PXE客户端也需要个IP，客户端启动后，会发送一个DHCP的请求，让DHCP server给它分配一个IP，那PXE客户端怎么知道服务器在哪里呢？主要依靠DHCP server除了分配IP外，还可以配置next-server，指向PXE服务器地址，另外还配置初始化启动文件filename。这样在PXE客户端发送DHCP请求后，不仅仅能得到一个IP，还可以知道PXE的服务器地址，也可以知道去下载哪个文件，用来初始化操作系统。下载文件使用TFTP协议，因此，除了PXE服务器，还有TFTP服务器，PXE客户端向TFTP服务器请求下载这个文件，TFTP服务器说好啊，于是就将这个文件发给他。然后，PXE客户端执行这个文件，这个文件会指示PXE客户端，向TFTP服务器请求
pxelinux.cfg。TFTP服务器会给PXE客户端一些配置文件，里面会说内核在哪里，inittramfs在哪里。PXE会继续请求这些文件。最后，启动Linux内核，一旦操作系统
起来了，接下就是DHCP分配IP的过程，动态配置IP后就可以使用了。




