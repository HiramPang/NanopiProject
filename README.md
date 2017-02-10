# Smart Car Project
##1	视频部分
###1.1	摄像头选择
####1.1.1	DVP接口摄像头
Matrix - CAM500A
需求：驱动，视频服务器，
实现困难，暂不采取
####1.1.2	USB摄像头
需UVC driver/tool（tool中自带视频服务器功能）
→motion使用：
安装：sudo apt-get install motion
  配置：sudo vi /etc/motion/motion.conf
  找到”control_localhost on “和”webcam_localhost on“这两行，改为以下两行后，保存退出
  control_localhost off
  webcam_localhost off
   调试：配置端口8080 监控端口8081
        →mjpg_streamer
	安装：
$sudo apt-get install libv4l-dev
$sudo apt-get install libjpeg8-dev 
$sudo apt-get install imagemagick 

mjpg_streamer源码包进行编译安装，这里说明一下，直接编译安装程序会报错，需要先运行一下命令，创建一个软连接

sudo ln -s /usr/include/libv4l1-videodev.h /usr/include/linux/videodev.h  
创建完成后开始下载编译安装mjpg_streamer源码包
$wget http://sourceforge.net/code-snapshots/svn/m/mj/mjpg-streamer/code/mjpg-streamer-code-182.zip 
$unzip mjpg-streamer-code-182.zip 
$cd mjpg-streamer-code-182/mjpg-streamer  
&sudo make USE_LIBV4L2=true clean all 
$sudo make DESTDIR=/usr install  
编译安装完成后，运行程序目录下的start..sh命令启动服务
监控端口: 8080 

###1.2	视频服务器建立
####1.2.1	Nanopi2取消热点模式
实现方法：用vi或在图形界面下用gedit编辑文件 /etc/wpa_supplicant/wpa_supplicant.conf, 在文件末尾填入路由器信息如下所示：
network={
        ssid="YourWiFiESSID"
	psk="YourWiFiPassword"
}
其中，YourWiFiESSID和YourWiFiPassword请替换成你要连接的无线AP名称和密码。
保存退出后，执行以下命令即可连接WiFi: 
ifdown wlan0
ifup wlan0
如果你的WiFi当前处于无线热点模式，你需要先退出该模式方可连接到路由器，使用以下命令退出无线热点模式：
su
turn-wifi-into-apmode no


####1.2.2	UDP协议来传输
###1.3	WIFI视频传输

##2	Android app 实现
###2.1	APP访问视频服务器并显示
###2.2

##3	避障
###3.1	使用HC-SR04超声波测距模块
####3.1.1	硬件说明
（1）	采用IO口TRIG触发测距。给至少10us的高电平新号
（2）	模块制动发送8个40khz的方波，自动检测是否有信号返回
（3）	有信号返回，通过IO口ECHO输出一个高电平，高电平持续时间就是超声波从发射到返回的时间。测试距离=（高电平时间*声速（340M/S））/2.

###3.2	解决方案
####3.2.1	10us电平输出。
技术实现：
1.PWM的使用方法 （不行，因为S5P4418的PWM的范围是30HZ ） 
→ 使用matrix-pwm，需要编译module，然后安装。
（1）但这个编译需要在板子上进行。老版matrix编译需要gccX64（64位，不能在板子运行）。目前下载新版matrix，通过U盘拷贝到板子上，再次尝试。
TODO：在板子上编译
（2）在drivers/char中有matrix_pwm.c和matrix_gpio_int.c，
在drivers/char中的Makefile有
obj-$(CONFIG_MATRIX_GPIO_INT)	+= matrix_gpio_int.o
obj-$(CONFIG_MATRIX_PWM)	+= matrix_pwm.o
在arch/arm/configs中的nanopi2_linux_deconfig中有
CONFIG_MATRIX_GPIO_INT=m
CONFIG_MATRIX_PWM=m
以上说明nanopi2中已经将gpio_sensor和pwm模块做好，但是没有直接编入内核，需要手动使用insmod 加载。
TODO：在板子的drivers中检查是否有相应的ko文件？可行
*在/lib/modules/3.4.39-s5p4418/kernel/drivers/char/中有相应的ko
TODO：修改nanopi2_linux_deconfig，和将/drivers/char/Kconfig中的相应的config （最后几行）中的default m改为y
→ 参照matrix，使用内核空间的控制方法。
使用gitsmart查看Nanopi2的更新history，找到和PWM相关的Module。
→已有相应的程序 /drivers/char/中，在Makefile中有相应的obj，在/drivers/char/Kconfig最后几行中也有相应的配置，但都是设为m（Build as a module, to be loaded if needed.）
必要时可以更改以上程序进行优化。
2. 使用GPIO来实现控制
（1）Timer控制
Delay实现：#include <unistd.h>
usleep(20); //delay 20us

（2）GPIO控制 
I．用户态监听GPIO 
解决方案：
https://www.kernel.org/doc/Documentation/gpio/sysfs.txt
http://blog.csdn.net/gqb_driver/article/details/8620809
利用/sys/class/gpio/gpio进行gpio操作
ultrasonicTriger管脚定为GPIO12
ultrasonicEcho管脚定为GPIO18
使用exportGPIOPin设置管脚
使用setGPIODirection设定方向
设置中断时需要定义edge，即/sys/class/gpio/gpio18/edge
读取管脚值使用/sys/class/gpio/gpio18/edge
管脚的gpio18中的18要用到itoa，但此函数linux中不支持，故使用sprintf进行转换。
使用strcat进行路径的拼接。
技术难点：
us级别的时间检测 → Linux中计时器的使用
时间计算：http://www.cnblogs.com/clover-toeic/p/3845210.html
_#include <sys/times.h>
	struct timespec time_start={0,0},time_end={0,0};
	long fCostTime;
	clock_gettime(CLOCK_REALTIME,&time_start);
	clock_gettime(CLOCK_REALTIME,&time_end);
	fCostTime = (long)(time_end.tv_nsec - time_start.tv_nsec);
这种方案可以测出距离，但很不稳定，会有错误的距离算出，会有负值的距离算出，而且是一个正确的，后面一个负值，在后面一个正确值。并且程序会卡死，原理待查
II. 内核态监听GPIO： 参照matrix-ultrasonicranger使用matrix_hcsr04模块
难点：
①matrix-ultrasonicranger的硬件管脚只有三个，VDD/GND/Controller其中Controller管脚是发出Triger信号和接受返回信息。
②matrix-ultrasonicranger调用/sys/class/hcsr04模块来实现的。Hcsr04由/lib/modules/3.4.39-sp54418/kernel/driver/char/matrix_hcsrc04.ko 来加载，该模块的源代码是linux/drivers/char/matrix_hcsr04.c
设置linux编译时加载方法：在/drivers/char/Kconfig 中
config MATRIX_HCSR04
tristate "Matrix hcsr04"
default m
---help---
Driver for Matrix hcsr04.
将default m  改为 default y

代码实现途径 Matrix_ultrasonic_ranger.c → iio.c →matrix_hcsr04.c
解决方法：根据手中的板子修改iio.c 和 matrix_hcsr04.c增加一个PIN的操作。
TODO：先将matrix_hcsr04.c中的发出Triger信号的PIN口固定。
修改方法：
1.修改iio.c中的Hcsr04Init，增加一个pin初始化
int Hcsr04Init(int Pin) →  Hcsr04Init(int cont, int echo)
libfahw-iio.h 中HCSR04_resource 中增加一个pin
2.修改matrix_hcsr04.c
HCSR04_resource()中增加pin定义
hcsr04_value_write()中获取pin值
hcsr04_hwinit()中获取pin资源
hcsr04_value_read()中设置trig输出和echo中断响应
gpio_isr（）中更改gpio的变量名
hcsr04_hwexit()增加io口释放
使用demo中的matrix_ultrasonicranger做测试（修改Hcsr04Init的调用）。
发生：Fail to get distance
Debug：
1.在readValueFromFile中添加printf → 运行时没有打印出来，
   在matrix_hcsr04的hcsr04_value_read()中增加对ECHO管脚的输入设置 
   →使用dmesg查看，发现有hcsr04_value_read timeout 错误发生
   原因：在/lib/modules/3.4.39-sp54418/kernel/driver/char/中的.ko都是原先就存在的并不是修改过的文件编译而成的。
2.先编译模块.ko http://salomi.blog.51cto.com/389282/362444
  在drivers/char目录下的Kconfig中把要编译的模块default值改为m
  然后make modules 则在相应的文件目录下可见.ko
  将matrix_hcsr04.ko拷贝到rootf/lib/modules/3.4.39-sp54418/kernel/driver/char/
  再次上电运行，按照模块后，修改过的printk（…..add）出现
  但运行matrix_ultrasonicranger, 仍旧报Fail to get distance 
  查看dmesg 在模块中所加的其他printk都未出现。
3.增加printk逐步查看
在matrix_hcsr04.c的hcsr04_value_write()添加输入参数监控，发现:len=4,trigPin=58,echoPin=0,而且该函数被调用两次，第二次时是在Hcsr04Deinit()中调用的，在该函数中trigIO被复位到-1，在第二次的Log中 trigPin=-1。这就说明程序的通路没有问题，关键点在echopin的值没有被传过去。
将iio.c中fwrite的sizeof改为8，结果不便。现在问题关键在fwrite如何调用class_attribute中的store接口的？
4.直接在matrix_hcsr04中指定trigPio = 58  echoPio=59
设置irq成功，但仍未收到中断。
示波器检测到器件的输入和输出波形，故trigpin没有问题。
将GPIO27 改为GPIO30后成功过一次，结果为13CM。但后面就再没成功过。
在gpio_isr()中添加printk,运行发现有一条输出结果，说明中断也进去了，但只仅进去了一次。所以是没有符合wake_up_interruptible（）要求。
从网上找hcsr04的驱动程序(https://github.com/tanzilli/hc-sr04/blob/master/hcsr04.c)，更改模块程序，在实验室里仍未成功。
到家后更换hcsr04模块，并使用外接电源，试验成功。但使用友善之臂提供的驱动仍不能成功。原因：1.模块损坏，使用新驱动和老模块，未能成功。→模块损坏
2.友善之臂的驱动有问题，使用老驱动和新模块未能成功。→老驱动有问题
3.在实验室中使用笔记本电源，新驱动新模块未能成功。使用Ipad电源仍未成功。
难道模块又坏了，→ 友善之臂的驱动会损坏hcsr04硬件模块？
在家用所有的hcsr04模块试验了一下，发现近距离无法检测，只有远距离才能检测成功。
原因待查
5.因为FA和网上使用的是class方法，不知道处于什么原因。待查
这里用device char(字符型设备)模式来实现看看看有什么异常.
参考：http://blog.chinaunix.net/uid-25014876-id-59416.html
http://www.latelee.org/embedded-linux/a-simple-char-driver.html
5-1：device char 准备
参照matrix_gpio_int.c 该文件是建立了一个gpio_int_sensor的misc device(杂项设备：因为有些字符设备不符合预先确定的字符设备范畴，所以这些设备采用设备号10，一起归于misc device，其实misc_register就是用设备号10调用register_chrdev()的)。
①	包含一些基本的头文件。<linux/modules.h>
②	编写功能函数 read() write() ioctl()等
③	定义file_operations结构体。
④	设备号申请：手动指定设备号register_chrdev_region()
动态分配设备号alloc_chrdev_region()
⑤	设备注册：分配cdev：直接定义或调用函数cdev_alloc()
初始化cdev:cdev_init()
添加cdev:cdev_add()
⑥	卸载模块：注销设备：cdev_del()
注销设备号
⑦	编译
5-2使用device
①	Insmod加载模块
②	创建设备文件 mknod /dev/hcsr04 c 主设备号 次设备号 (cat /proc/devices 查询主设备号)
运行成功，距离检测正确。
遗留问题：使用一次后再开，出现open device faile 待查 → 没有free_irq。
####3.2.2	在主控程序中调用HCSR04设备。
由于实验中出现误报的情况，故在监听中加入滤波功能，即受到距离在少于或超出某个范围时，等下一次测量来进行确认。确认情况改变后在发出消息。
####3.2.3	避障处理方案


#####3.2.3.1	总体方案
在选择Auto运行状态后，开启一个进程开始控制小车自动行驶，同时再开一个超声波的进程来监控距离。两个进程之间使用消息队列进行通信。超声波进程通知小车距离障碍的安全与危险状态。当小车收到危险状态时，改变车子的行驶状态，直到收到安全状态通知为止。
Linux 线程通信方法：http://blog.csdn.net/ljianhui/article/details/10287879 
msgget(key_t key, int msgflg) 创建访问消息队列
msgsnd(int msgid, const void *msg_ptr, size_t msg_sz, int msgflg) 把消息添加到消息队列中，其中msg_ptr所指向的消息结构一定要以一个长整形的成员变量开始的结构体，接受函数将用这个成员来确定消息的类型。
Struct my_message{
Long int message_type;
/*The data you want to tranfer*/
};
msgrcv(int msgid, void *msg_ptr, size_t msg_st, long int msgtype, int msgflg)
msgctl(int msgid, int command, struct msgid_ds *buf);// command值为IPC_RMID是删除消息队列


#####3.2.3.2	改变行驶状态的解决方案


####3.2.4	模式切换。

解决方案：1.进入MENU模式时，Kill掉AUTO线程。进入AUTO模式时，再次Create。
2.使线程暂停，然后需要时重启。（Linux中不支持进程挂起）
##4	轨迹


##5	手机模拟方向盘控制小车。
###5.1	方案：使用重力加速传感器。通过Y轴和Z轴值的变化来控制小车。
首先手机横置，
手机向左倾斜时（Y轴值<0）：车子左转
手机向水平时（Y轴值=0）：车子直行
手机向右倾斜时（Y轴值>0）：车子左转
手机向前倾斜时（Z轴值>6）：车子前进
手机向后倾斜式（Z轴值<6）：车子后退
###5.2	Android重力加速传感器使用
private void gravityTest(){
	sensorMgr = (SensorManager) getSystemService(SENSOR_SERVICE);
	final Sensor sensor = sensorMgr.getDefaultSensor(Sensor.TYPE_ACCELEROMETER);
	SensorEventListener lsn = new SensorEventListener() {
		@Override
		public void onSensorChanged(SensorEvent event) {
			int  x = (int)event.values[0];
			int y = (int)event.values[1];
			int z = (int)event.values[2];
			action.setText("X="+String.valueOf(x)+"     Y="+String.valueOf(y)+"     Z="+String.valueOf(z));
		}
		@Override
		public void onAccuracyChanged(Sensor sensor, int accuracy) {
		}
	};
	sensorMgr.registerListener(lsn, sensor, SensorManager.SENSOR_DELAY_UI);
}

###5.3	改变小车行驶速度

####5.3.1	方案：使用占空比为50%的方波进行前进与后退控制
难点：占空比方波的实现方法，方波的周期。
实现：使用PWM来实现方波，PWM的实现方法
1.调用/drivers/char/matrix_pwm.c来建立PWM的device
2.调用Matrix/lib/pwm.c中的PWMPlay()和PWMStop()对PWM进行控制。
目前使用HZ=1000 duty=500,当duty<500时无法驱动电机。
####5.3.2	


##6	程序架构设计
设计auto,menu,traction三个操作模式:
难点：模式切换时，线程的终止与重启。
方案：1.使用pthread_kill, 发送SIGKILL. 结果发生进程退出
2. 使用pthread_cancel.只能在创建该线程的线程中调用。
3. 利用全局变量，当发现被置位时，退出循环而退出线程。
设置start，stop 状态模式
##7	附注

###7.1	Linux signal处理
1.	初始化struct sigaction
2.	初始化sa_handler
3.	调用int sigaddset(sigset_t *set,int signum);
4.	调用int sigaction(int signum,const struct sigaction *act ,struct sigaction *oldact);
###7.2	Uboot 编译时 make s5p4418_nanopi2_config 发生 bin/sh 0 illegal option –
1.	所使用的uboot源码是用smartgit pull下来的故里面的文件时dos编码，故在ubuntu上使用bash编译前要使用 dos2unix mkconfig来进行转码到utf8才行
2.	要转变文件夹内所有文件编码：find -type f | xargs dos2unix -o
###7.3	编译linux时，直接使用make 时发生no rule to make target 'net/netfilter/xt_tcpmss.o' , needed by net/netfilter/built-in.o'
解决过程：
1. obj-y生成built-in.o
Kbuild编译所有的$(obj-y)文件，并调用”$(LD) -r”把所有这些文件合并到built-in.o文件。这个built-in.o会被上一级目录的Makefile使用，最终链接到vmlinux中。

2. MSS 作用
MSS表示TCP数据包的每次能够传输的最大数据分段。MSS的主要作用是在TCP建立连接的过程通常要写上对发的MSS值，这个值是TCP协议实现的时候根据MTU换算而得（主要是1500-20个大小的包头-20个大小的TCP数据包头）。因此一般的MSS值大小为1460. 

发现在/net/netfilter/Makefile中有obj-$(CONFIG_NETFILTER_XT_TARGET_TCPMSS) += xt_TCPMSS.o 但与文件名xt_tcpmss不同（大小写不同）。将大写改为小写后通过编译。
###7.4	编译uImage时缺少mkimage命令
解决方法：
从https://launchpad.net/ubuntu/xenial/amd64/u-boot-tools/2016.01+dfsg1-2ubuntu1 
下载u-boot-tools_2016.01+dfsg1-2ubuntu1_amd64.deb 
后使用 dpkg 命令安装
###7.5	SD卡烧写linux系统
1.	烧写Uboot 
1) 在电脑上先用命令 sudo apt-get install android-tools-fastboot 安装 fastboot 工具; 
2) 用串口配件连接NanoPi2和电脑，在上电启动的2秒内，在串口终端上按下回车，进入 u-boot 的命令行模式；
3) 在u-boot 命令行模式下输入命令 fastboot 回车，进入 fastboot 模式； 
4) 用microUSB线连接NanoPi2和电脑，在电脑上输入以下命令烧写u-boot.bin: fastboot flash bootloader u-boot.bin 

*如果是空卡，使用dd命令  http://www.cnblogs.com/humaoxiao/p/4282903.html
烧写后，未能成功启动。→ 原因待查
2.	烧写Uimage:将uImage拷贝到SD卡的boot分区中。

	
   

