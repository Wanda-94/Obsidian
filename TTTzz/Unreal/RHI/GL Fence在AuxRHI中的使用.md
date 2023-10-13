![[Pasted image 20230728112302.png]]
Fence用于GPU上的多线程资源同步，关键函数如下
- glFlush：发送指令到GPU，但不等待其返回，CPU无阻塞执行
- glFinsh：发送指令到GPU，CPU端等待指令在GPU上执行完成
- glFenceSync：创建Fence
- glWaitSync：GPU上等待Fence，不阻塞CPU
- glClientWaitSync：等待Fence，阻塞CPU

之前一直有个问题，在AuxRHI线程上会进行ShaderLink，Buffer Create，Texture Update/Create三个操作，但是Texture Update/Create在部分机型上（Adreno）存在一些贴图更新失败的样子
![[Pasted image 20230728114322.png|825]]
很离谱的是只有固定的贴图会出现这个问题，比如上面的UI和树干部分，一开始怀疑到贴图自身数据问题，但是这些数据在RHI线程上是没有问题的，后面怀疑到数据同步上，一开始同步是这样做的
```c++
//AuxRHIThread
glUpdateTexture2DData(texture,data);
CPUFence.Set()

//RHIThread
CPUFence.Wait()
glBindTexture2D(texture)
```
当时认为即使是不同线程的gl command，也会依照其提交先后顺序执行，所以只需要在RHI Thread使用Texture前在CPU处调用CPUFence，等待AuxRHI Thread执行完上传指令即可，其结果就是上图，有些图明显是没有上传成功的
这里需要先说明一下OpenGL的执行逻辑
![[GL Fence在AuxRHI中的使用 2023-07-28 12.00.32.excalidraw|1000]]
在CPU端调用的GL指令并不会立刻送往GPU执行，而是会先记录到一个Command Buffer中，当合适的时机（GPU空闲，Command Buffer指令过多，CPU强制推送，GL Driver控制？）会一口气上传到GPU上执行，这里有几个指令可以起到GPU强制推送的效果
- glFlush
- glFinsh
- glSwapBuffer

一般不需要我们调用到glFlush和glFinsh，只需要在帧渲染结束时调用glSwapBuffer即可，所有的Command的执行时机交给GL Driver去控制，但是对于上述的资源同步情况有所不同
在多线程的情况下，每个线程拥有自己的Command Buffer，运行情况是这样的
![[GL Fence在AuxRHI中的使用 2023-07-28 12.20.48.excalidraw|1000]]
所以对于下面这段代码
```c++
//AuxRHIThread
glUpdateTexture2DData(texture,data);
CPUFence.Set()
// 未强制推送command，推送时机交由GL Driver判断

//RHIThread
CPUFence.Wait()
// AuxRHIThread的指令可能还未被推送到GPu
glBindTexture2D(texture)
```
执行失败的原因就是CPU的Fence并不起作用，glUpdateTexture2DData的执行时间并没有被做同步，那么如果按如下方式执行
```c++
//AuxRHIThread
glUpdateTexture2DData(texture,data);
glFlush();
CPUFence.Set()
// 未强制推送command，推送时机交由GL Driver判断

//RHIThread
CPUFence.Wait()
// AuxRHIThread的指令可能还未被推送到GPu
glBindTexture2D(texture)
```
结果是否回正确，猜测还是可能会出问题，因为GL提供了一个SyncObject功能，如果用Flush就能同步的话SyncObject是可以被完全替代的，并且SyncObject在插入Fence后也需要Flush
```c++
//AuxRHIThread
glUpdateTexture2DData(texture,data);
glFlush();
CPUFence.Set()
// 未强制推送command，推送时机交由GL Driver判断

//RHIThread
CPUFence.Wait()
// AuxRHIThread的指令可能还未被推送到GPu
glBindTexture2D(texture)
```
所以正确的做法应该是
```c++
glUpdateTexture2DData(texture,data);
fence = glFenceSync();
glFlush();
CPUFence.Set()

CPUFence.Wait()
glWaitSync();
glBindTexture2D(texture)
```
完成上述改动后图片就完全显示正确了，下面是Sync使用时需要注意的点
1. GL ES下WaitSync的参数
	![[Pasted image 20230728152652.png]]
	两个参数都需要使用默认参数，0以及TIMEOUT_IGNORED
2. 创建SyncObject后需要Flush
	![[Pasted image 20230728152703.png]]
	在创建SyncObject后需要Flush到GPU，否则可能出现GPU上等待一个未被推送到GPU上的SyncObject导致死锁（未验证）
附一个Mult-Thread Context资源删除行为描述
![[Pasted image 20230728153354.png]]