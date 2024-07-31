---
title: ByteCTF 2020 Writeup [RE] Easy Flutter 
categories: writeup
tags: [writeup,re]
---

比赛没做出来，继续研究。这题是把代码藏进了VM，Dart是一种谷歌为了跨平台搞的语言，具体的~~没有深入了解~~。

~~略去走了两天的弯路，实际解题过程却挺短的，先来废话几句弯路~~

作为RE新手，第一感觉是这种题的难点在于代码定位，我首先的思路是，捕捉点击事件，并且一路跟下去，总能触发事件回调，找到目标函数。

![image-20201028111203863](https://timg.reito.fun/archive/typora-schwarzer/image-20201028111203863.png)

在这里就不赘述了，总之得出结论，消息会传入队列，然后分发到Dart创建的UI线程，这样跟是跟不到的。随后我又考虑到一般会在Dart的`main`函数中注册按钮回调，在查找了一些Dart屈指可数的[资料](https://flutter.dev/docs/development/add-to-app/performance)后，定位到这里。这一段我还在看64位的版本，后面为了分析方便就换成32位的了。

![image-20201028112301691](https://timg.reito.fun/archive/typora-schwarzer/image-20201028112301691.png)

猜测是通过查找libapp.so的`_runMainZoned`函数，这段代码应该是AOT生成时，直接从Dart转的，因此Dart的VM代码里找不到这一段，一路跟下去，Dart2CPP太牛，根本看不出哪个是目标。

接着注意到libapp.so中存在`checkFlag`，想起应该和C#的Binary类似，这种VM应该都存在反序列化过程，于是找到了Dart的[源码](https://github.com/dart-lang/sdk)。通过分析，总结出以下反序列化相关的函数调用过程

```cpp
main
	Dart_Initialize
		Dart::Init
			FullSnapshotReader::ReadVMSnapshot //读取VM的Snapshot
    	...	
    	Dart::InitIsolateFromSnapshot
    		FullSnapshotReader::ReadProgramSnapshot //读取Isolate的Snapshot
    	...
```

读取的内容，就是libapp.so导出的符号

![image-20201028113639770](https://timg.reito.fun/archive/typora-schwarzer/image-20201028113639770.png)

两两为一组，构成了两个Snapshot，Dart首先读取VM的，初始化一些类型表，随后在初始化好的环境中读取Isolate，就是用户编译的程序。

对于AOT的Dart，一个Snapshot却由3部分组成，我称之为Metadata，ROData，以及Instruction。具体原因下文会说明。

首先来看`FullSnapshotReader::ReadVMSnapshot`

```CPP
SnapshotHeaderReader header_reader(kind_, buffer_, size_);

intptr_t offset = 0;
char* error = header_reader.VerifyVersionAndFeatures(/*isolate=*/NULL, &offset);
```

在这里，首先校验了版本和Features，因为不同版本序列化方法不一样，我在这里也踩过坑，拿着master的代码去研究这题的文件，这题的版本是2.10.0-77.0.dev，注意从仓库拉的时候要找到对应版本。随后进行反序列化。以下代码在`/runtime/vm/clustered_snapshot.cc`

```CPP
VMDeserializationRoots roots;
deserializer.Deserialize(&roots);
```

```CPP
Array& refs = Array::Handle(zone_);
num_base_objects_ = ReadUnsigned();
num_objects_ = ReadUnsigned();
num_clusters_ = ReadUnsigned();
const intptr_t field_table_len = ReadUnsigned();

clusters_ = new DeserializationCluster*[num_clusters_];
refs_ = Array::New(num_objects_ + kFirstReference, Heap::kOld);
```

这里会先读取4个数，决定后续反序列化的对象个数，这里推荐先去看这篇[文章](https://blog.tst.sh/reverse-engineering-flutter-apps-part-1/)，会对后续内容有更深理解。这里还有个坑，做过本题的人可能无法理解为什么直接拿010Editor看，啥都看不出来。是因为本题使用了[LEB128编码](https://en.wikipedia.org/wiki/LEB128)（Metadata与ROData段）。

随后会按这个个数进行分析，Cluster表示一个类，对于VM的Snapshot，存在数个类，按顺序存放，每次读取一个类，并在类中读取一个长度，决定该类有多少个对象，最后分配进入全局对象表。

Dart使用全局对象表存储所有对象，使对象之间通过在该表的位置进行索引，num_objects表示本阶段所有对象的数量。对于每个读取阶段（VM，Isolate，Unit（本题不关心）），首先会读取上一个阶段的全局对象表，当然VM是第一阶段，所作的事情是添加一些基础类型进全局对象表，本文就不贴代码了。

```
roots->AddBaseObjects(this);
```

然后开始读取

```CPP
for (intptr_t i = 0; i < num_clusters_; i++) {
    clusters_[i] = ReadCluster();
    clusters_[i]->ReadAlloc(this);
}
```

对于每一个类，都重载了`ReadAlloc`函数进行读取。这是第一阶段，首先在Dart的全局对象表中Alloc对象，`ReadCluster()`函数实现了通过Class的Id实例化不同类的作用。

![image-20201028115851534](https://timg.reito.fun/archive/typora-schwarzer/image-20201028115851534.png)

随后在`ReadFill`函数中把对象的值域从文件中读取。

```CPP
for (intptr_t i = 0; i < num_clusters_; i++) {
	clusters_[i]->ReadFill(this);
```

对于部分Cluster，还重载了PostLoad进行后处理

```CPP
for (intptr_t i = 0; i < num_clusters_; i++) {
	clusters_[i]->PostLoad(this, refs);
}
```

这样就反序列化完成了，现在我们需要找到最关心的代码位置，注意到`CodeDeserializationCluster`的`ReadFill`中做了这样一件事 

```CPP
void ReadFill(Deserializer* d, intptr_t id, bool deferred) {
    auto const code = static_cast<CodePtr>(d->Ref(id));
    Deserializer::InitializeHeader(code, kCodeCid, Code::InstanceSize(0));

    d->ReadInstructions(code, deferred);
```

```CPP
void Deserializer::ReadInstructions(CodePtr code, bool deferred) {
  if (deferred) {
#if defined(DART_PRECOMPILED_RUNTIME)
    if (FLAG_use_bare_instructions) {
      uword entry_point = StubCode::NotLoaded().EntryPoint();
      code->ptr()->entry_point_ = entry_point;
      code->ptr()->unchecked_entry_point_ = entry_point;
      code->ptr()->monomorphic_entry_point_ = entry_point;
      code->ptr()->monomorphic_unchecked_entry_point_ = entry_point;
      code->ptr()->instructions_length_ = 0;
      return;
    }
#endif
```

注意`code->ptr()->entry_point_ = entry_point;`，这就是我们要找的东西了。而通过上文的文章了解到，Code对象被Function对象引用，再看`FunctionDeserializationCluster`

![image-20201028120611758](https://timg.reito.fun/archive/typora-schwarzer/image-20201028120611758.png)

其中，`ReadFromTo`是按照结构体顺序读取值域

![image-20201028120330043](https://timg.reito.fun/archive/typora-schwarzer/image-20201028120330043.png)

`ReadRef`是从文件读取一个整数，返回该整数索引的对象位置。我们来看Function的结构体

![image-20201028120529873](https://timg.reito.fun/archive/typora-schwarzer/image-20201028120529873.png)

其中有`name`域，因此推断，可以通过name域获取到`code`。

下面开始解题过程，理论上可以通过引用Dart SDK的代码，写程序，直接得到反序列化结果，但本文使用了C#实现了自己的Dart反序列化，[源码](https://github.com/cnSchwarzer/DartDeserialize)（部分实现，题做完了就没往下实现所有Cluster了）。

首先看`Program.cs`，读取ELF文件，解析32/64位，并读取两个Snapshot。

```C#
SerializedDartReaderElf dr = new SerializedDartReaderElf(fs);
            (Snapshot vm, Snapshot isolate, TargetId target) = dr.GetVMIsolateSnapshotAndTarget();
```

创建一个DartEnv，当上下文用，由于32/64位会对反序列化流程产生影响，所以传入参数。

```C#
DartEnv env = new DartEnv(target);

SnapshotReader vmReader = new SnapshotReaderVM(vm);
vmReader.ResolveSnapshot(env);

SnapshotReader isolateReader = new SnapshotReaderIsolate(isolate);
isolateReader.ResolveSnapshot(env);
```

接着仿照Dart，首先解析VM，其次解析Isolate。这里，为了解释上文中提到的Snapshot分三部分，稍作说明，首先运行完`ReadAlloc`阶段，程序给出以下类信息。

![image-20201028121354369](https://timg.reito.fun/archive/typora-schwarzer/image-20201028121354369.png)

前三段，都是从Data，也就是`_kDartVmSnapshotData`这一段读取内容，到了最后一段

![image-20201028121607802](https://timg.reito.fun/archive/typora-schwarzer/image-20201028121607802.png)

注意到对于**AOT**在**这个版本**的情况下，所有这些类都被归到`ROData`类里了，看该类的读取代码了解到

![image-20201028121712947](https://timg.reito.fun/archive/typora-schwarzer/image-20201028121712947.png)

是通过`GetObjectAt`获取一个对象指针，而跟进去发现

![image-20201028121751242](https://timg.reito.fun/archive/typora-schwarzer/image-20201028121751242.png)

是以`data_image_`为偏移量起始的，这里的`data_image_`很有迷惑性，让人以为是`_kDartVmSnapshotData`，实际上是` FullSnapshotReader`在构造函数时，把`_kDartVmSnapshotData`分成了两段，因此我称作一个是Metadata，一个是Data。

![image-20201028121956839](https://timg.reito.fun/archive/typora-schwarzer/image-20201028121956839.png)

上图中，`snapshot`是`_kDart{xxx}SnapshotData`，`instructions_buffer`是`_kDart{xxx}SnapshotInstructions`。而`snapshot->DataImage()`指向了`snapshot`的一个位置

![image-20201028122134046](https://timg.reito.fun/archive/typora-schwarzer/image-20201028122134046.png)

`length()`就是写在`snapshot`头部的一个长度，长度以后就不是metadata了，这是一个坑。这样就能顺利读取完VM，接下来就要读取Isolate，一个一个实现`ReadAlloc`。

![image-20201028122344722](https://timg.reito.fun/archive/typora-schwarzer/image-20201028122344722.png)

得出如下结果

![image-20201028122404696](https://timg.reito.fun/archive/typora-schwarzer/image-20201028122404696.png)

可以看见，我们只需要把`ReadFill`实现到Code，再记录下`checkFlag`在对象池的位置，就可以知道偏移量了。实现完成后，读取32位版本的`libapp.so`。（32位不知道哪里有Bug？比读64位慢多了，可能是不对齐导致的？）

![image-20201028122622096](https://timg.reito.fun/archive/typora-schwarzer/image-20201028122622096.png)

在读取到`checkFlag`时，程序中断

![image-20201028122732287](https://timg.reito.fun/archive/typora-schwarzer/image-20201028122732287.png)

获取到对象的索引是

![image-20201028122805153](https://timg.reito.fun/archive/typora-schwarzer/image-20201028122805153.png)

继续读取，在读完Function后，成功获取到代码位置

![image-20201028122848738](https://timg.reito.fun/archive/typora-schwarzer/image-20201028122848738.png)

![image-20201028122910676](https://timg.reito.fun/archive/typora-schwarzer/image-20201028122910676.png)

接下来就可以愉快的分析目标函数了，打开IDA，定位到`0x2457A0 + 0xB000`

![image-20201028123114879](https://timg.reito.fun/archive/typora-schwarzer/image-20201028123114879.png)

查找引用，发现调用者

![image-20201028123153005](https://timg.reito.fun/archive/typora-schwarzer/image-20201028123153005.png)

得出只要返回table[29]即可获得胜利，推测这里可能是某种类型表，因为Dart源码里有

![image-20201028123244703](https://timg.reito.fun/archive/typora-schwarzer/image-20201028123244703.png)

可能就是和这个比较，接着我们分析目标函数。注意到函数底部

![image-20201028123337375](https://timg.reito.fun/archive/typora-schwarzer/image-20201028123337375.png)

如果判断失败，直接就返回False了，因此我们往上追溯，有一个`idx = v109;`，那么图中`idx`表示的是下文的数组。

![image-20201028123526617](https://timg.reito.fun/archive/typora-schwarzer/image-20201028123526617.png)

再看另一个比较对象（`str_transformed`），注意到有

![image-20201028123723396](https://timg.reito.fun/archive/typora-schwarzer/image-20201028123723396.png)

赋值v77，而v77表示`2 * ((v74 << v75) + v66)`，这里通过动态调试知道v75和v66都是1，可能是Dart的加Tag处理，不做过多猜测。

![image-20201028123818258](https://timg.reito.fun/archive/typora-schwarzer/image-20201028123818258.png)

一路追溯可以找到v72是关键，判断上面还有处理过程，继续追溯

![image-20201028124009656](https://timg.reito.fun/archive/typora-schwarzer/image-20201028124009656.png)

这两个是一个对象，继续往上

![image-20201028124055222](https://timg.reito.fun/archive/typora-schwarzer/image-20201028124055222.png)

因此v57和两个`str_transformed`是一个东西，继续分析

![image-20201028124157338](https://timg.reito.fun/archive/typora-schwarzer/image-20201028124157338.png)

在这里进行了赋值，因此追溯v59

![image-20201028124217159](https://timg.reito.fun/archive/typora-schwarzer/image-20201028124217159.png)

![image-20201028124225680](https://timg.reito.fun/archive/typora-schwarzer/image-20201028124225680.png)

![image-20201028124257503](https://timg.reito.fun/archive/typora-schwarzer/image-20201028124257503.png)

这里，经过动态调试（题外话：MIUI的假root名不虚传，真动态调试还是得刷个原生系统）看了下，v41就是输入的字符串，v44是上文另一个数组

![image-20201028124328920](https://timg.reito.fun/archive/typora-schwarzer/image-20201028124328920.png)

因此，v101就是输入字符串与该数组值的低8位的异或。Dart充满了这种最低位的操作，还是猜测是Tag，不做过多研究。至此， 此题可以写出解题脚本。

```python
ori = [184, 464, 364, 230, 54, 260, 240, 96, 352, 370, 198, 476, 224, 246, 372, 378, 296, 50, 194, 292, 222, 146, 408, 58, 358, 154]
ans = [122, 582, 778, 90, 354, 858, 250, 302, 838, 922, 6, 606, 286, 290, 918, 566, 970, 282, 22, 974, 118, 506, 590, 430, 890, 194]

str = list(('ByteCTF{' + '?'*17 + '}').encode())

for idx in range(26):
    for c in range(32, 127):
        step1 = ori[idx] >> 1
        step2 = c ^ step1
        step3 = 2 * step2
        step4 = step3 >> 1
        step5 = (step4 << 1) + 1
        step6 = 2 * step5

        if step6 == ans[idx]:
            str[idx] = c

print(bytes(str).decode())
```

运行得出结果

```
ByteCTF{a_by73_0f_dar7_vm}
```

贴一张开心图

<img src="https://timg.reito.fun/archive/typora-schwarzer/Screenshot_2020-10-28-10-42-47-153_com.example.ea.jpg" style="zoom: 33%;" />

一点心得：以后VM题还是直接尝试反序列化，按VM的逻辑读取吧，不要试图强行找目标了，太费时间。