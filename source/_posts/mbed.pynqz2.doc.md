---
title: Pynq-Z2 开发指南与实例（Linux系统方式）
date: 2019-12-22 01:08:00
categories: mbed
tags: [mbed]
---

## 前言
本教程分三个部分
1. 介绍使用官方镜像文件，使用Python进行开发的一个简单小例子，并且简单了解底层是如何运作的。
2. 记录使用自制镜像文件，使用C/C++开发基于Linux的Pynq-Z2。完成一个将3.5毫米耳机输入口的音频采集，并输出到一个USB音频设备的任务。
3. 自行编译Pynq-Z2镜像文件的步骤（高级）

## 使用Python开发Pynq的简单介绍
### 概述
本节使用了Pynq官方镜像v2.5，基于Ubuntu 18.04，Linux 4.19.0。将简单介绍安装流程，Python程序样例简介与简单认识底层实现原理。
### 前期准备
一张 microSD卡（推荐16GB以上）
一个 microSD卡读卡器
一根 网线
一个 路由器
一根 3.5mm公对公音频线（内录线）（如果需要体验实验结果）
下载 [Pynq-Z2 磁盘镜像（本节基于v2.5）](https://github.com/Xilinx/PYNQ/releases)
安装 [Win32DiskImager](https://sourceforge.net/projects/win32diskimager/)
安装 [DiskGenius](http://www.diskgenius.cn/download.php)
安装 [WinSCP](https://winscp.net/eng/download.php)（如果需要访问完整文件系统）

下载完成后请安装下载的软件
### 安装配置
将SD卡插入电脑，打开Win32DiskImager，选择下载的镜像文件，点击写入
![](https://timg.reito.fun/archive/schwarzer-blog/2019/12/f86f9830-8fe8-43f9-b21f-ee481c936ddb.png)
![](https://timg.reito.fun/archive/schwarzer-blog/2019/12/62d03938-93e4-4f11-9a51-4b0827b18aa6.png)
待完成后，**不要**按系统提示进行格式化，打开DiskGenius。
![](https://timg.reito.fun/archive/schwarzer-blog/2019/12/370a72e9-8e15-4c7e-9e89-5c7addada6b4.png)
在左侧找到SD卡磁盘，选择较大的分区（大概5GB那个），右键，扩容分区。
![](https://timg.reito.fun/archive/schwarzer-blog/2019/12/edf2163f-3158-4bd4-b7ef-7133a1cc058c.png)
点击开始，等待约5分钟即可完成。
![](https://timg.reito.fun/archive/schwarzer-blog/2019/12/1b2f32a9-948d-46f0-9d9d-27091f6a90b3.png)
![](https://timg.reito.fun/archive/schwarzer-blog/2019/12/1c8d8100-e070-4b16-b527-09d89fa43b1d.png)
剩余时间是假的，不必理会。
完成后，将SD卡从电脑上取下，插入Pynq-Z2的卡槽里（背部）。
![](https://timg.reito.fun/archive/schwarzer-blog/2019/12/89054c3f-2bb9-4179-954d-a4596cc7911c.png)
将启动方式跳线帽选择SD。
![](https://timg.reito.fun/archive/schwarzer-blog/2019/12/474f5f44-c9a7-429c-9326-2c1352bd95bd.png)
使用microUSB线连接到电脑，开启电源。
等待`4个绿色LED，2个蓝色LED`高亮闪烁后，代表启动完成。
![](https://timg.reito.fun/archive/schwarzer-blog/2019/12/39fbccd7-ca37-4e55-b8c1-7881aa6fde01.png)
使用网线连接Pynq-Z2到电脑的局域网中，将自动获取IP地址。
以小米路由器为例，可以看到开发板的IP地址为`192.168.31.69`。
![](https://timg.reito.fun/archive/schwarzer-blog/2019/12/7e20f9d2-c5f8-43ec-bc98-8dec3a8a75a4.png)
浏览器访问该IP，将自动跳转到`Jupyter Notebook`。
需要输入密码，该linux系统的账号为`xilinx`，密码为`xilinx`。
![](https://timg.reito.fun/archive/schwarzer-blog/2019/12/3544937e-3983-402a-8091-518e794bd4ba.png)
点击登录
![](https://timg.reito.fun/archive/schwarzer-blog/2019/12/600cabdf-96d0-4da2-9ba3-ffd71f1b13a9.png)
### 文件系统
可以通过文件管理器访问`\\<你的Pynq-Z2 IP地址>\xilinx`，来访问其文件系统（无法访问根目录）。
![](https://timg.reito.fun/archive/schwarzer-blog/2019/12/dab611a4-42cb-4144-8f30-06d1de76f40d.png)
![](https://timg.reito.fun/archive/schwarzer-blog/2019/12/0f9207e4-70d8-4b43-8114-74d5d4aadde7.png)
如果需要访问完整文件系统，请按以下步骤进行，如不需要请跳过。
下载安装 [WinSCP](https://winscp.net/eng/download.php)
按如下方法配置
![](https://timg.reito.fun/archive/schwarzer-blog/2019/12/8caa09f6-6f43-4983-9707-43870bbd2769.png)
点击保存（保存密码），即可以后在左侧选择。
![](https://timg.reito.fun/archive/schwarzer-blog/2019/12/430e2346-6c32-45b1-bdc8-9ceb6274c205.png)
点击登录，即可访问完整目录。
![](https://timg.reito.fun/archive/schwarzer-blog/2019/12/9e5163a4-c69a-4efa-a881-de439f59de12.png)
### 程序分析
`base`文件夹中包含了很多python的例子，可以自行参考学习。
本文以`base/audio/audio_playback.ipynb`为研究对象。
![](https://timg.reito.fun/archive/schwarzer-blog/2019/12/b1efe18f-c02f-481d-8748-79f0d45e1c01.png)
该例程使用python控制板载的`ADAU1761`芯片，采集从耳机接口输入的音频。
![](https://timg.reito.fun/archive/schwarzer-blog/2019/12/67f29ae8-b173-46e8-8477-01faff35f47a.png)
`LINE_IN`代表双声道输入接口，`HP MIC`代表耳麦（传统4段式3.5毫米耳机接口，2声道输出，1声道输入）
首先来看其第一个功能
![](https://timg.reito.fun/archive/schwarzer-blog/2019/12/0e627dfd-271f-477d-98ef-92724ca44e3b.png)
具体实现了选择`LINE_IN`作为输入，直接输出到`HP MIC`的输出，不储存。
你需要一根"内录线"，即3.5mm公对公音频线，一端插入`LINE_IN`，一端插入一个输出源（如手机耳机口），再插入一个耳机到`HP MIC`中，即可听见声音。
可以自己点击上方的运行查看效果，你将听到从`LINE_IN`输入的声音。
![](https://timg.reito.fun/archive/schwarzer-blog/2019/12/a0a0f37b-64e2-4040-a2b1-7eea6be36f47.png)
下面我们来分析他具体做了什么
访问文件系统`\\192.168.31.69\xilinx\pynq\lib`，将IP替换为你自己的。
![](https://timg.reito.fun/archive/schwarzer-blog/2019/12/b91bba12-eb6f-4d79-a45e-4455d66787e6.png)
这里就是其具体调用的python文件。
![](https://timg.reito.fun/archive/schwarzer-blog/2019/12/a665ee13-500d-4eb2-a254-61b41dbd809a.png)
这两部是加载Pynq开发者为Pynq-Z2写的底层PL文件`base.bit`，和加载`libaudio.so`类库。下文删除了注释，请自己打开文件查看。

```python
class AudioADAU1761(DefaultIP):
	def __init__(self, description): 
        super().__init__(description)

        self._ffi = cffi.FFI()
        self._libaudio = self._ffi.dlopen(LIB_SEARCH_PATH + "/libaudio.so")
        self._ffi.cdef("""void config_audio_pll(int iic_index);""")
        self._ffi.cdef("""void config_audio_codec(int iic_index);""")
        self._ffi.cdef("""void select_line_in(int iic_index);""")
        self._ffi.cdef("""void select_mic(int iic_index);""")
        self._ffi.cdef("""void deselect(int iic_index);""")
        self._ffi.cdef("""void bypass(unsigned int audio_mmap_size,
                          unsigned int nsamples, 
                          int uio_index, int iic_index) ;""")
        self._ffi.cdef("""void record(unsigned int audio_mmap_size,
                          unsigned int * BufAddr, unsigned int nsamples,
                          int uio_index, int iic_index);""")
        self._ffi.cdef("""void play(unsigned int audio_mmap_size,
                          unsigned int * BufAddr, unsigned int nsamples,
                          int uio_index, int iic_index);""")

        self.buffer = numpy.zeros(0).astype(numpy.int32)
        self.sample_rate = None
        self.sample_len = len(self.buffer)
        self.iic_index = None
        self.uio_index = None
        self.configure()
```
这里是将native层（原生、C/C++层）的函数暴露给python层使用，根据函数定义去解析`libaudio.so`的符号，查找地址，并建立调用封送约定。

```python
def configure(self, sample_rate=48000,
                  iic_index=1, uio_name="audio-codec-ctrl"): 
        self.sample_rate = sample_rate
        self.iic_index = iic_index
        self.uio_index = get_uio_index(uio_name)
        if self.uio_index is None:
            raise ValueError("Cannot find UIO device {}".format(uio_name))

        self._libaudio.config_audio_pll(self.iic_index)
        self._libaudio.config_audio_codec(self.iic_index)
```
这里是设置采样率，配置时钟，让底层去查找在linux系统中注册的IO口。

```python
    def select_line_in(self): 
        self._libaudio.select_line_in(self.iic_index)
	def bypass(self, seconds): 
        if not 0 < seconds <= 60:
            raise ValueError("Bypassing time has to be in (0,60].")

        self.sample_len = math.ceil(seconds * self.sample_rate)
        self._libaudio.bypass(self.mmio.length,
                              self.sample_len, self.uio_index, self.iic_index)
```
其余函数都是对native层的封装调用，可以说，做的并不好，逻辑全在native层里，python也没封装的咋样。
为此需要进一步查看`libaudio.so`中是如何实现的，可以查看源码
访问目录`\\192.168.31.69\xilinx\pynq\lib\_pynq\_audio`，即为`libaudio.so`的源码。由于我们是Pynq-Z2，所以查看`audio_adau1761.cpp`即可。

首先查看两个配置函数
```python
        self._libaudio.config_audio_pll(self.iic_index)
        self._libaudio.config_audio_codec(self.iic_index)
```
由于过长，全文请自行查看文件。
```cpp
extern "C" void config_audio_pll(int iic_index) {
    ...
    // Poll PLL Lock bit
    u8TxData[0] = 0x40;
    u8TxData[1] = 0x02;
    do {
        if (writeI2C_asFile(iic_fd, u8TxData, 2) < 0){
            printf("Unable to write I2C %d.\n", iic_index);
        }
        if (readI2C_asFile(iic_fd, u8RxData, 6) < 0){
            printf("Unable to read I2C %d.\n", iic_index);
        }
    } while((u8RxData[5] & 0x02) == 0);
    ...
}

extern "C" void config_audio_codec(int iic_index) {
	...
    // Mute Mixer1 and Mixer2 here, enable when MIC and Line In used
    write_audio_reg(R4_RECORD_MIXER_LEFT_CONTROL_0, 0x00, iic_fd);
    write_audio_reg(R6_RECORD_MIXER_RIGHT_CONTROL_0, 0x00, iic_fd);
    // Set LDVOL and RDVOL to 21 dB and Enable left and right differential
    write_audio_reg(R8_LEFT_DIFFERENTIAL_INPUT_VOLUME_CONTROL, 0xB3, iic_fd);
    write_audio_reg(R9_RIGHT_DIFFERENTIAL_INPUT_VOLUME_CONTROL, 0xB3, iic_fd);
    // Enable MIC bias
    write_audio_reg(R10_RECORD_MICROPHONE_BIAS_CONTROL, 0x01, iic_fd);
    // Enable ALC control and noise gate
    write_audio_reg(R14_ALC_CONTROL_3, 0x20, iic_fd);
    // Put CODEC in Master mode
    write_audio_reg(R15_SERIAL_PORT_CONTROL_0, 0x01, iic_fd);
    ...
}
```
可见，是根据[数据手册](https://www.analog.com/cn/products/adau1761.html#)，使用IIC接口对ADAU1761进行配置。
再查看具体功能函数
```cpp
extern "C" void select_line_in(int iic_index) {
    int iic_fd;
    iic_fd = setI2C(iic_index, IIC_SLAVE_ADDR);
    if (iic_fd < 0) {
        printf("Unable to set I2C %d.\n", iic_index);
    }

    // Mixer 1  (left channel)
    write_audio_reg(R4_RECORD_MIXER_LEFT_CONTROL_0, 0x01, iic_fd);
    // Enable LAUX (MX1AUXG)
    write_audio_reg(R5_RECORD_MIXER_LEFT_CONTROL_1, 0x07, iic_fd);

    // Mixer 2
    write_audio_reg(R6_RECORD_MIXER_RIGHT_CONTROL_0, 0x01, iic_fd);
    // Enable RAUX (MX2AUXG)
    write_audio_reg(R7_RECORD_MIXER_RIGHT_CONTROL_1, 0x07, iic_fd);

    if (unsetI2C(iic_fd) < 0) {
        printf("Unable to unset I2C %d.\n", iic_index);
    }
}
```
这一步是将`LINE_IN`选作采样端口。

```cpp
extern "C" void bypass(unsigned int audio_mmap_size,
                       unsigned int nsamples, 
                       int uio_index, int iic_index) {
    int i, status;
    void *uio_ptr;
    int DataL, DataR;
    int iic_fd;

    uio_ptr = setUIO(uio_index, audio_mmap_size);
    iic_fd = setI2C(iic_index, IIC_SLAVE_ADDR);
    if (iic_fd < 0) {
        printf("Unable to set I2C %d.\n", iic_index);
    }
```
这里是取得linux对iic以及设备的封装
```cpp
    // Mute mixer1 and mixer2 input
    write_audio_reg(R23_PLAYBACK_MIXER_LEFT_CONTROL_1, 0x00, iic_fd);
    write_audio_reg(R25_PLAYBACK_MIXER_RIGHT_CONTROL_1, 0x00, iic_fd);
    // Enable Mixer3 and Mixer4
    write_audio_reg(R22_PLAYBACK_MIXER_LEFT_CONTROL_0, 0x21, iic_fd);
    write_audio_reg(R24_PLAYBACK_MIXER_RIGHT_CONTROL_0, 0x41, iic_fd);
    // Enable Left/Right Headphone out
    write_audio_reg(R29_PLAYBACK_HEADPHONE_LEFT_VOLUME_CONTROL, 0xE7, iic_fd);
    write_audio_reg(R30_PLAYBACK_HEADPHONE_RIGHT_VOLUME_CONTROL, 0xE7, iic_fd);
```
这里是配置输入输出
```cpp
    for(i=0; i<nsamples; i++){
        //wait for RX data to become available
        do {
            status = \
            *((volatile unsigned *)(((uint8_t *)uio_ptr) + I2S_STATUS_REG));
        } while (status == 0);
        *((volatile unsigned *)(((uint8_t *)uio_ptr) + I2S_STATUS_REG)) = \
            0x00000001;

        // Read the sample from the input
        DataL = *((volatile int *)(((uint8_t *)uio_ptr) + I2S_DATA_RX_L_REG));
        DataR = *((volatile int *)(((uint8_t *)uio_ptr) + I2S_DATA_RX_R_REG));

        // Write the sample to output
        *((volatile int *)(((uint8_t *)uio_ptr) + I2S_DATA_TX_L_REG)) = DataL;
        *((volatile int *)(((uint8_t *)uio_ptr) + I2S_DATA_TX_R_REG)) = DataR;
    }
```
这里是关键，将音频数据取样，并直接发送。
```cpp
    write_audio_reg(R23_PLAYBACK_MIXER_LEFT_CONTROL_1, 0x00, iic_fd);
    write_audio_reg(R25_PLAYBACK_MIXER_RIGHT_CONTROL_1, 0x00, iic_fd);
    write_audio_reg(R22_PLAYBACK_MIXER_LEFT_CONTROL_0, 0x00, iic_fd);
    write_audio_reg(R24_PLAYBACK_MIXER_RIGHT_CONTROL_0, 0x00, iic_fd);
    write_audio_reg(R29_PLAYBACK_HEADPHONE_LEFT_VOLUME_CONTROL, 0xE5, iic_fd);
    write_audio_reg(R30_PLAYBACK_HEADPHONE_RIGHT_VOLUME_CONTROL, 0xE5, iic_fd);

    if (unsetUIO(uio_ptr, audio_mmap_size) < 0){
        printf("Unable to free UIO %d.\n", uio_index);
    }
    if (unsetI2C(iic_fd) < 0) {
        printf("Unable to unset I2C %d.\n", iic_index);
    }
}
```
这里是收尾。
音频数据就这么简单地被获取了，可以按照他的方法，实现我们自己的应用程序。
读者可以按图索骥，分析其他函数与例程。
## 进阶开发
### 概述
本节使用笔者自己编译的Pynq-Z2镜像文件（基于v2.5），如果你也想自己编译，请看第三节。自己编译的原因是，由于本节的目标是将ADAU1761的数据输出到一个USB音频设备，但是官方镜像的Linux内核没有编译ALSA（底层音频库），也没有USB Audio的驱动，如果手动进行开发非常麻烦。所以自己将内核配置为支持上述选项。
### 前期准备
一个 USB音频设备（本文以华为Type-C降噪耳机3为例）
一个 [USB转接头（USB A转Type-C母）](https://detail.tmall.com/item.htm?spm=a230r.1.14.51.2b18e426KepXXd&id=570945658553&ns=1&abbucket=20&skuId=4451476047437)（如果你的设备是Type-C耳机）
安装 [Putty](https://the.earth.li/~sgtatham/putty/latest/w64/putty.exe)（可选）
下载 [Pynq镜像](https://pan.baidu.com/s/1acneXYi1OyPsPkulzymzXg) fole
下载 [环境配置脚本](https://pan.baidu.com/s/1FNbnq1OvSHBltLC6nwypeQ) 6ffd
### 安装配置
请参考第一节进行安装镜像，并插入SD卡，上电启动。
打开WinSCP，并连接到开发板。
解压环境配置脚本，将`pynq_setup`文件夹拖入`/home/xilinx`中
浏览器访问`http://192.168.31.69:9090`，点击右上角的**New-Terminal**，打开一个终端。
![](https://timg.reito.fun/archive/schwarzer-blog/2019/12/ea5a83c5-18c1-4cc9-a7ba-550551f6aa08.png)
![](https://timg.reito.fun/archive/schwarzer-blog/2019/12/f20f2cb6-d3db-4225-b312-2da3144b6787.png)
你也可以使用`putty`
![](https://timg.reito.fun/archive/schwarzer-blog/2019/12/eb20e521-f188-4a68-b7fc-392f2894491f.png)
点击`Save`可以保存配置供下次使用，点击`Open`。
![](https://timg.reito.fun/archive/schwarzer-blog/2019/12/13c32920-6ab0-479a-ac4e-c31217e21f08.png)
用户名密码都为`xilinx`。
```powershell
cd pynq_setup
chmod 777 ./setup.sh
./setup.sh
```
![](https://timg.reito.fun/archive/schwarzer-blog/2019/12/f9fdee89-cd0b-49af-b597-7cf72bbdfb11.png)
运行过程中会提示
![](https://timg.reito.fun/archive/schwarzer-blog/2019/12/0de8f79e-a03f-4755-97d8-57fb6e5868fb.png)
请输入`y`，回车。
由于`Jupyter Notebook`拥有root权限，因此`sudo`时貌似不用输入密码，而如果使用`putty`，在提示输入`xilinx`的密码时请输入`xilinx`。
运行完成后会自动重启，稍等半分钟即可启动完成。
将USB音频设备插入，开启终端，输入`aplay -l`
![](https://timg.reito.fun/archive/schwarzer-blog/2019/12/38a9710e-6dbf-4cf6-9f0e-77607bfa33c6.png)
如果一切顺利，此处会显示你的设备。
再输入`aplay ./pynq_setup/test.wav`
如果顺利，你可以从耳机中听见一段优美的旋律（音量可能较大）。
如果播放失败，请参考下面的进阶安装配置。
### 进阶安装配置
此部分是针对你个人的USB音频设备的配置调优。 
由于`.asoundrc`仅对当前用户生效，使用浏览器`Jupyter Notebook`打开的终端是`root`用户
![](https://timg.reito.fun/archive/schwarzer-blog/2019/12/1cc07946-6848-4d28-9536-b3edb2745eaf.png)
而我们的用户名是`xilinx`，所以请使用`putty`。
你也可以在浏览器终端中输入`su xilinx`切换![](https://img-blog.csdnimg.cn/2019122014302728.png)
**请注意：如果当前用户名不对，音频配置文件不会被加载。如果发生了各种错误，请先核对你的用户名（@前面的是否为`xilinx`）。**

请打开`.asoundrc`文件，可以用WinSCP打开编辑，或是在本地编辑后，拖到WinSCP里进行上传替换。 
文件是针对各个音频格式做的详细配置，由于我的耳机支持4种配置，你可以看见
```bash
pcm.at24b96k {
	type plug
	slave.pcm "hw-24b-96k"
}
pcm.hw-24b-96k {
    type hw
    card 0
	device 0
	rate 96000
	format S24_3LE      
}
```
就代表这个配置是24位，96000采样率的，第一个`pcm.at24b96k`中，`slave.pcm`引用了下面的`hw-24b-96k`，这一步是必要的，这样就可以让alsa（音频驱动）自己进行格式转换，如播放一个16位48000采样率的音频，和播放设备不匹配，会自动进行转换。
配置了这个如何使用呢？有两种办法
第一，在`aplay -D 24b96k test.wav`中声明自己的配置名称。`-D`后面跟着名称。
![](https://timg.reito.fun/archive/schwarzer-blog/2019/12/41ebe01a-5424-4805-8122-062f6220bfef.png)
可见正常播放，你也可以增加`-v`来看当前的详细配置。
![](https://timg.reito.fun/archive/schwarzer-blog/2019/12/9db8ec56-0bb3-44a3-946b-fbdbaca690b9.png)
你可以尝试`aplay -D hw-24b-96k test.wav`，会提示格式不匹配，无法播放，因此需要这一层转换（实际是调用了alsa的格式转换插件）。
![](https://timg.reito.fun/archive/schwarzer-blog/2019/12/0934700e-29f2-4eac-98d9-d8dc3026e57c.png)
以此类推，还有`16bit 48k, 16bit 96k, 24bit 48k`三种配置，除此之外，文件最上面还有
```bash
ctl.!default {
	type hw
	card 0
}

pcm.!default {
	type plug
	slave.pcm "hw-16b-48k"
}
```
代表默认配置，如果你不在`aplay`时输入`-D`指定，则默认使用`default`，前面的感叹号是必要的，起警示作用。
那么我的设备有这四种配置，读者的配置可能有所不同，需要您自己修改后替换。具体步骤如下
输入`cat /proc/asound/card0/stream0`，查看设备属性
![](https://timg.reito.fun/archive/schwarzer-blog/2019/12/ea9f0a27-e7b5-4c83-979f-a8931079af7a.png)
可见华为降噪耳机3(Type-C)叫CM-Q3，USB Audio设备，有2个`Format`，每个`Format`有2个采样率，共有`16b48k，16b96k，24b48k，24b96k`四种输出格式，只有一种输入格式`16b48k`。
其中`Rates`表示采样率，`Format`需要和配置文件中对应，设备之间有差异请仔细核对。
根据这里的信息，再去调整你的`.asoundrc`，最后重启设备`sudo reboot`即可。

还有一个是控制功能。
输入`amixer contents`
![](https://timg.reito.fun/archive/schwarzer-blog/2019/12/66c8c2dc-e331-4479-9b46-35fe28bd41f3.png)
可以获得配置项，可以看到我的设备，Volume设置的id是4，所以可以
`amixer cset numid=4 15`
来调整音量。
### 程序编写
对于笔者来说，已经研究了4天，有两个好消息：USB音频设备可以正常播放音频了，ADAU1761也可以正常采集音频了。剩下的就是写一个程序把他们连起来。我花了一天时间调试了一个可以用的程序，开源出来给大家参考。
```bash
su xilinx 
cd
git clone https://gitee.com/Schwarzer/adausb.git
cd adausb
make
sudo ./adausb
```
需要注意由于`uio`设备无法被非`root`用户访问，所以需要使用`sudo ./adausb`。
如果正常的话，你将从USB耳机中听到`LINE_IN`输入的声音。
你可以指定声卡，如`./adausb at24b48k`，如不填则等同于`./adausb default`
你可以指定延时，如`./adausb at24b48k 10000`，为10ms，延时越大，缓冲区越大，越不容易发生错误。
![](https://timg.reito.fun/archive/schwarzer-blog/2019/12/18a34a79-3957-4508-9c80-8cdd74c5dded.png)
如果出现`underrun occurred`，说明输入速度慢于输出了，这是由于，官方的PL层可能出现了问题，应该为48000的采样率，实际测试为48740，太快了，所以代码里做了调整。听上去可能有很短暂的一个空白声，几乎无影响。下面来看代码，我的代码都在`adausb.cpp`和`log.h`里。

```cpp
#include "alsa/asoundlib.h"
#include "log.h" 
#include "i2cps.h"
#include "uio.h"
#include "audio_adau1761.h" 
#include <memory.h>
#include <stdint.h>
#include <stdlib.h>   
#include <pthread.h>
#include <signal.h> 
#include <time.h> 
```
所要使用的所有头文件
```cpp
struct adau_ctx
{ 
	uint32_t* opt_buf; 
	uint32_t opt_buf_idx;
	uint32_t opt_buf_size;  
};
```
传递给adau采样线程的上下文
```cpp
adau_ctx ctx;
pthread_t adau_thread;
snd_pcm_t *pcm; 
volatile uint8_t running;

extern "C" void int_handler(int dummy) { 
	loge("\ninterrupted!\n");
	running = 0; 
} 
```
捕获Ctrl+C，不过我没有多做处理。
```cpp
static timespec ts;
uint64_t getns() {
	clock_gettime(CLOCK_MONOTONIC_RAW, &ts);
	uint64_t ret = ts.tv_sec;
	ret *= 1000000000;
	ret += ts.tv_nsec;
	return ret;
}
```
获取纳秒级的时间，这是由于采样率为48740，需要手动修正到48000
```cpp
void* adau_sampling(void* pctx)
{
	adau_ctx* ctx = (adau_ctx*)pctx;
	void* uio_ptr = setUIO(0, 65536);
	if (uio_ptr < 0) {
		loge("unable to mmap uio\n");
		exit(-1);
	}
	int iic_fd = setI2C(1, IIC_SLAVE_ADDR);
	if (iic_fd < 0) {
		loge("unable to set i2c %d\n", 1);
		exit(-1);
	}

	uint32_t status, l, r; 
	 
	uint64_t start = getns();
	uint64_t sample_cnt = 1;
	double sample_rate = 48000.0;
	double second_ns = 1000000000.0;
	double next_sample_time = second_ns / sample_rate;

	while (running)
	{ 
		do {
			status =  *((volatile unsigned*)(((uint8_t*)uio_ptr) + I2S_STATUS_REG));
		} while (status == 0); 
		*((volatile unsigned*)(((uint8_t*)uio_ptr) + I2S_STATUS_REG)) = 0x00000001; 
		l = *((volatile int*)(((uint8_t*)uio_ptr) + I2S_DATA_RX_L_REG));
		r = *((volatile int*)(((uint8_t*)uio_ptr) + I2S_DATA_RX_R_REG)); 
```
初始化`I2C,UIO`，初始化各项时间限制数据，采集一个样本
```cpp
		uint64_t t = getns() - start; 
		while(t < next_sample_time)
		{
			t = getns() - start;
		}
```
等待`1/48000`秒
```cpp
		ctx->opt_buf[ctx->opt_buf_idx++] = l;
		ctx->opt_buf[ctx->opt_buf_idx++] = r;
		if (ctx->opt_buf_idx >= ctx->opt_buf_size)
		{ 
			ctx->opt_buf_idx = 0; 
		}
		next_sample_time = second_ns * ++sample_cnt / sample_rate; 
	}
```
放到缓冲区，计算下一个采样时间（保证精度）
```cpp
	if (unsetUIO(uio_ptr, 65536) < 0) {
		loge("unable to free UIO %d\n", 0);
	}
	if (unsetI2C(iic_fd) < 0) {
		loge("unable to unset I2C %d\n", 1);
	}
}   

int main(int argc, char ** argv)
{
	char * pcm_name = (char *)"default";
	if(argc >= 2)
	{
		pcm_name = argv[1];
		logd("pcm_name = %s\n", pcm_name);
	}

	int ret = snd_pcm_open(&pcm, pcm_name, SND_PCM_STREAM_PLAYBACK, 0);
	if(ret < 0){
		loge("audio open error %d\n", ret);
		return -1;
	}

	logd("initialize\n");
	running = 1;
	signal(SIGINT, int_handler);
 
	int latency = 10000;
	if(argc >= 3)
		sscanf(argv[2], "%d", &latency);
	ret = snd_pcm_set_params(pcm, SND_PCM_FORMAT_S24_LE, SND_PCM_ACCESS_RW_INTERLEAVED, 2, 48000, 1, latency);
	if(ret < 0){
		loge("cannot set params\n"); 
		return -1;
	} 
```
初始化`ALSA API`，设置`PCM`的格式
```cpp
	snd_pcm_uframes_t framesize, periodsize;
	snd_pcm_get_params(pcm, &framesize, &periodsize);
	logd("frame size = %d period size = %d\n", framesize, periodsize);
	
	uint32_t blocks = 16;
	ctx.opt_buf_size = periodsize * 2 * blocks; 
	ctx.opt_buf = new uint32_t[ctx.opt_buf_size];  
	ctx.opt_buf_idx = 0; 
	
	config_audio_pll(1);
	config_audio_codec(1);
	select_line_in(1);
```
使用官方的`adau1761.cpp`中初始化函数
```cpp
	ret = pthread_create(&adau_thread, NULL, adau_sampling, &ctx);
	if (ret) {
		loge("create adau thread failed");
		exit(-1);
	}   
 
	logd("adau priority max\n");
	sched_param param; 
	pthread_getschedparam(adau_thread, &ret, &param);
	param.sched_priority = sched_get_priority_max(ret);
	pthread_setschedparam(adau_thread, ret, &param);  
```
创建采样线程，给他优先级拉满
```cpp
	uint32_t block_idx = 0;
	uint32_t block_cnt = blocks / 4; 

	uint64_t start_time = getns();
	double sample_rate = 48000.0;
	double block_time_numerator = periodsize * 1000000000 ; //need divide by sample_rate
	double next_feed_time = block_time_numerator * block_cnt / sample_rate; 

	snd_pcm_prepare(pcm);

	while (running)
	{
		uint64_t t = getns() - start_time;
		if(t >= next_feed_time)
		{ 
			uint32_t * buf = ctx.opt_buf + periodsize * 2 * block_idx++;
			if(block_idx == blocks) block_idx = 0;

			ret = snd_pcm_writei(pcm, buf, periodsize);
			if(ret == -EPIPE){
				snd_pcm_recover(pcm, ret, 0);
			}
			else if(ret < 0){ 
				loge("writei err %s\n", snd_strerror(ret));
			}
			else if(ret != periodsize){
				loge("short %d %d\n", ret, periodsize);
			}
			next_feed_time = block_time_numerator * (++block_cnt) / sample_rate; 
		} 
	}
```
当采样到一定个数时，向ALSA提交缓冲区。  
当然，笔者的程序非常不完美。需要修改才能保证持久稳定运行。
## 编译Pynq-Z2镜像文件
其他的不用我多说什么，如果你想自己调整内核，我把我的经验分享在这里。
### 前期准备
需要下载大量文件！
建议你去学校机房下载。笔者就是去学校机房把网线拔下来插自己电脑上下的，1000M网比较快，建议准备一个代理。
本文是基于2.5的，如果后续有更新，请参考[这篇文章](https://github.com/Xilinx/PYNQ/blob/master/docs/source/pynq_sd_card.rst)，和[这篇文章](https://github.com/Xilinx/PYNQ/blob/master/sdbuild/README.md)。

笔者电脑为Windows 10 64位，16GB内存，i7-6700HQ，请预留至少100GB硬盘

下载 [Vivado + SDK 2019.1](https://www.xilinx.com/member/forms/download/xef-vivado.html?filename=Xilinx_Vivado_SDK_2019.1_0524_1430.tar.gz) 需要注册xilinx账号 22GB
下载 [Petalinux 2019.1](https://www.xilinx.com/member/forms/download/xef.html?filename=petalinux-v2019.1-final-installer.run) 7GB
下载 [VirtualBox](https://www.virtualbox.org/wiki/Downloads) 选择自己的架构，并且安装
下载 [Ubuntu 16.04.6镜像](https://mirrors.tuna.tsinghua.edu.cn/ubuntu-releases/16.04.6/ubuntu-16.04.6-desktop-amd64.iso)
原本，编译非常耗时，完整需要大概一天。不过我们参考文档可知，可以加速编译，请先下载
下载 [arm架构的通用文件镜像](https://www.xilinx.com/member/forms/download/xef.html?filename=pynq_rootfs_arm_v2.5.zip) 2GB
下载 [预先编译的Pynq文件](https://github.com/Xilinx/PYNQ/releases/download/v2.5/pynq-2.5.tar.gz) 50MB
### 配置
下载完成后，应该有如下这些文件
![](https://timg.reito.fun/archive/schwarzer-blog/2019/12/da419f9c-3ac9-456e-960d-bcd73d62b0e5.png)
首先安装VirtualBox，请自行完成安装运行
![](https://timg.reito.fun/archive/schwarzer-blog/2019/12/0ea52095-2652-4058-8635-1f3d560e9f56.png)
点击新建
![](https://timg.reito.fun/archive/schwarzer-blog/2019/12/59de16a8-8c30-4f8d-a99d-f422a31b8aa4.png)
输入信息
![](https://timg.reito.fun/archive/schwarzer-blog/2019/12/c6fbdb6c-ab7f-45b7-9566-53db913e860a.png)
建议为自己电脑的一半
![](https://timg.reito.fun/archive/schwarzer-blog/2019/12/ae6eafeb-005b-4491-85e4-3794cd6ecaaf.png)
创建磁盘
![](https://timg.reito.fun/archive/schwarzer-blog/2019/12/6a9aa287-d3a9-4cff-b8cf-b5295c081f8f.png)
选择动态创建，大小选择200GB（按个人喜好，不低于100GB）
再打开设置
![](https://timg.reito.fun/archive/schwarzer-blog/2019/12/7951584c-7dfd-4f09-a365-69da5180293b.png)
系统-处理器，处理器数量选择自己电脑核心数。
![](https://timg.reito.fun/archive/schwarzer-blog/2019/12/54ef6c1b-5ec7-412a-aa73-672708dac28e.png)
后续步骤请参考[这篇文章](https://blog.csdn.net/scene_2015/article/details/83025750)，或自行搜索，直到进入系统。
![](https://timg.reito.fun/archive/schwarzer-blog/2019/12/fcbb6ff2-372e-4320-8c63-802eeb38b24f.png)
如果你的桌面很小，记得点击顶部的`设备-安装增强功能`，并重启。
现在点击`设备-共享文件夹`
![](https://timg.reito.fun/archive/schwarzer-blog/2019/12/b4c63910-fea2-4721-980c-2df4e9c7eb20.png)
点击固定分配，再点击右侧的加号，把自己刚才下载的那些文件所在的文件夹添加进来。
我的是`H:\_IDM`，因此是
![](https://timg.reito.fun/archive/schwarzer-blog/2019/12/c2df81c0-b31e-413f-bc1e-78347ddb7d9b.png)
点击确定即可在根目录看到。
![](https://timg.reito.fun/archive/schwarzer-blog/2019/12/9ecd88fb-ed08-4d67-852f-58b9b0a1f6ae.png)
不过现在还没完，请输入
`sudo adduser <你的用户名> vboxsf`
这样才有权限访问它，需要重启才能生效
`sudo reboot`

右键打开终端 
```bash
sudo apt install git
```
这里需要先克隆PYNQ的仓库，我帮大家镜像到了gitee上，并做了修改，请克隆我的仓库。
我做的修改有两处，第一，就是把`boards`中只留下Pynq-Z2的文件，第二就是把预编译文件放在了`dist`下，不然后续会报错。如果你克隆我的仓库有问题，可以自行完成上述更改。
你需要先注册一个gitee账号才能克隆这个仓库，建议注册，以后github速度慢，可以先克隆到gitee。
`git clone https://gitee.com/Schwarzer/PYNQ.git` 
![](https://timg.reito.fun/archive/schwarzer-blog/2019/12/fe11711c-5c8b-47d3-8014-458f85c6fa33.png)
提示输入用户名密码，请输入。

随后就开始安装了，由于笔者已经安装过了，不能再次安装，所以可能有遗漏之处。
我的用户名是`schwarzer`，安装文件放在`/idm`，请根据自己情况修改！
安装Vivado和SDK，请一路下一步，选WebPack，安装路径为/home/<你的用户名>/xilinx
```bash
cd
mkdir downloads
cd downloads
tar -xf /idm/Xilinx_Vivado_SDK_2019.1_0524_1430.tar.gz
cd Xilinx_Vivado_SDK_2019.1_0524_1430
./xsetup
```
安装Petalinux
```bash
sudo apt install tofrodos iproute gawk xvfb make net-tools libncurses5-dev tftpd
sudo apt install zlib1g-dev zlib1g:i386 libssl-dev flex bison libselinux1 gnupg wget diffstat
sudo apt install chrpath socat xterm autoconf libtool tar unzip texinfo gcc-multilib
sudo apt install build-essential libsdl1.2-dev libglib2.0-dev screen pax gzip
/idm/petalinux-v2019.1-final-installer_2.run /home/schwarzer/xilinx
```
安装完成后，你的xilinx路径下应该是这样，**请把`petalinux`文件夹改名为`petalinux-v2019.1-final`**
![](https://timg.reito.fun/archive/schwarzer-blog/2019/12/039cd7dd-8a3c-4b45-8e44-7d687472a1d1.png)
然后需要把arm架构通用镜像复制进来
请先解压`pynq_rootfs_arm_v2.5.zip`得到`bionic.arm.2.5.img`
```bash
mkdir ~/PYNQ/sdbuild/prebuilt
cp /idm/bionic.arm.2.5.img ~/PYNQ/sdbuild/prebuilt
```

如果你没有用我的gitee仓库：
1、还需要把预编译文件复制，用了就不要了
```bash
mkdir ~/PYNQ/dist
cp /idm/pynq-2.5.tar.gz ~/PYNQ/dist
```
2、添加环境配置文件，将以下内容保存为`xilinx.sh`，保存在`~\PYNQ\sdbuild\xilinx.sh` 
```bash
export XILINX_VIVADO=/home/${USER}/xilinx/Vivado/2019.1
export PATH=/home/${USER}/xilinx/Vivado/2019.1/bin:/home/${USER}/xilinx/Vivado/2019.1/lib/lnx64.o:$PATH
export PATH=/home/${USER}/xilinx/SDK/2019.1/bin:/home/${USER}/xilinx/SDK/2019.1/gnu/microblaze/lin/bin:/home/${USER}/xilinx/SDK/2019.1/gnu/arm/lin/bin:/home/${USER}/xilinx/SDK/2019.1/gnu/microblaze/linux_toolchain/lin64_le/bin:/home/${USER}/xilinx/SDK/2019.1/gnu/aarch32/lin/gcc-arm-linux-gnueabi/bin:/home/${USER}/xilinx/SDK/2019.1/gnu/aarch32/lin/gcc-arm-none-eabi/bin:/home/${USER}/xilinx/SDK/2019.1/gnu/aarch64/lin/aarch64-linux/bin:/home/${USER}/xilinx/SDK/2019.1/gnu/aarch64/lin/aarch64-none/bin:/home/${USER}/xilinx/SDK/2019.1/gnu/armr5/lin/gcc-arm-none-eabi/bin:/home/${USER}/xilinx/SDK/2019.1/tps/win64/cmake-3.3.2/bin:/home/${USER}/xilinx/SDK/2019.1/gnuwin/bin:$PATH
export PATH=/home/${USER}/xilinx/DocNav:$PATH
export PYNQ_UBUNTU_REPO=https://mirrors.tuna.tsinghua.edu.cn/ubuntu-ports/
```
还有环境配置脚本，将以下内容保存为`env.sh`，保存在`~\PYNQ\sdbuild\env.sh`
```bash
source ~/xilinx/petalinux-v2019.1-final/settings.sh
source ./xilinx.sh
petalinux-util --webtalk off
```
3、删除
```bash
rm -rf ~/PYNQ/boards/Pynq-Z1
rm -rf ~/PYNQ/boards/ZCU104
```
4、保存更改
```bash
cd ~/PYNQ
git add .
git commit -m 'update'
```
如果你用我的，就不用修改上述内容了

随后准备开始编译
如果你要修改内核配置，请打开`~/PYNQ/sdbuild/scripts/`，在如图所示位置添加
```bash
petalinux-config --get-hw-description=$BSP_BUILD/hardware_project -c kernel
```
![](https://timg.reito.fun/archive/schwarzer-blog/2019/12/74ce5b01-f415-47ef-8714-7865bd3cbb54.png)
如果你不想修改Pynq-Z2的硬件，可以通过修改加速编译过程：
通过手动make，预先完成编译，然后修改makefile的文件名，脚本就可以跳过这个步骤。
```bash
cd ~/PYNQ/boards/Pynq-Z2/petalinux_bsp/hardware_project
make
mv ./makefile ./makefiled
```
同样，完成上述修改后，记得
```bash
cd ~/PYNQ
git add .
git commit -m 'update'
```
环境配置
```bash
cd ~/PYNQ/sdbuild/scripts
./setup_host.sh
```
万事俱备，准备编译
```bash
cd ~/PYNQ/sdbuild
source env.sh 
```
使用以下命令以加速编译，如果想体验完整编译，直接`make`。
```bash 
make PREBUILT="../prebuilt/bionic.arm.2.5.img" PYNQ_SDIST="../dist/pynq-2.5.tar.gz" BOARDS="Pynq-Z2"
```
![](https://timg.reito.fun/archive/schwarzer-blog/2019/12/3c9b51fa-1f3f-4ad2-aa61-aff921bdfad8.png)
![](https://timg.reito.fun/archive/schwarzer-blog/2019/12/cbf443f7-e784-4e2b-ac37-12f460e24c99.png)
如果你想要配置内核，那么中途会弹出配置图形界面，请按需配置。会弹出两次，第二次直接退出即可。
![](https://timg.reito.fun/archive/schwarzer-blog/2019/12/164d7660-8f7a-46e4-aa7c-1a0db7adcc15.png)
如果意外发生中断，建议你重新编译。
这里要注意，如果发生下载中断，可以通过删除`~/PYNQ/sdbuild/build/PYNQ`来重试
![](https://timg.reito.fun/archive/schwarzer-blog/2019/12/c27842aa-a6be-4d49-a8f2-6259e8dbe2ad.png)
![](https://timg.reito.fun/archive/schwarzer-blog/2019/12/1cafff72-b06d-4161-acbf-827528d63443.png)
如果一切顺利，一个小时内将完成编译，文件如下所示
![](https://timg.reito.fun/archive/schwarzer-blog/2019/12/05158f87-3a66-4e4d-9282-aa84089c3bed.png)
如果想重新编译，请`make clean`
如果遇到困难，欢迎留言，或联系邮箱：cnschwarzer@qq.com。谢谢！
