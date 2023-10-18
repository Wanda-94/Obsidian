Iwiki:https://iwiki.woa.com/p/4008262205
## AuxRHI（Android Only）
AuxRHI线程主要负责将CreateBuffer，UpdateTexture，ShaderLink三类会造成CPU卡顿的GL API调用与RHI线程进行分离，其工作模式是RHI线程将以上任务分配给AuxRHI，待资源需要真正使用时根据情况等待AuxRHI或者直接忽略部分DrawCall
代码主要参考commit：3acdeab087b5c2b73c12eea46742e5b8ed8c72a0
以及后续AuxRHI相关改动
## Shader Compile & Link
commit：3acdeab087b5c2b73c12eea46742e5b8ed8c72a0
MainPass中DrawCall所使用的Shader如果在其使用时未被编译（尚未准备好，例如PSO未预热该Shader且第一次使用的情况）则需要原地编译，耗时10+ms~100+ms不等，在开启AuxRHI ShaderLink的情况下MainPass会丢弃本次DrawCall，并将Shader Compile & Link的执行交给AuxRHI，待AuxRHI编译完成返回后RHI线程的DrawCall则正常执行，整个流程不会造成CPU（RHI线程）的卡顿
且AuxRHI目前还负责PSO第一次预热时ShaderBinaryCache的生成
## CreateBuffer
commit：3acdeab087b5c2b73c12eea46742e5b8ed8c72a0
AuxRHI会负责StaticMesh的Index Buffer & Vertex Buffer的创建，这类Buffer的特征是一旦创建则在其生命周期内不会改变其内容，这里有一个小修改可能会有点性能问题（测试了一下200 Buffer创建的情况下内存申请的耗时几乎没有变化）是将Buffer Raw数据的内存申请由RHICommandList自己的内存分配器（一段连续的已经申请好的内存地址，每帧会自动释放）改为全局Malloc分配器，以将其资源所属权从RHI线程分离，后续如果开启该功能在低端机要关注一下这个点，同时现在CreateBuffer会进行 RHI-AuxRHI 同步，也就是在Buffer未准备好时进行Wait而非丢弃DrawCall（同步方式与下面Texture一致，都是GPU Sync）
```
Engine/Source/Runtime/OpenGLDrv/Public/OpenGLDrv.h
Line:821-845
```
## UpdateTexture
commit：716ed2c6ec4a8c4144defd7813db5e822212d068
Texture的更新和Buffer类似（只会处理运行过程中不会改变内容的Texture2D，具体判断方式是创建Texture时是否有传入InitData），但是相对Buffer，Texture在使用前需要Wait AuxRHI将Texture上传完毕（ShaderLinke和CreateBuffer未执行完成的情况下则可以选择直接舍弃该DrawCall，Texture由于设计TextureStreaming，为了防止场景中的Mesh闪烁需要Wait资源创建完成），Wait功能需要拓展UE的GL RHI层实现，原本的GL RHI并没有提供在GPU上Wait Sync的功能（只有CPU Wait），具体Commit参见：05c2cdaacc8ea3f0447825f297f581ae13e40008
如果不做Sync，则会导致资源上传失败（Texture显示错误）
## UserBinaryCache
commit：fdd572b4d2fec77f6486aeb56a9c10215e366a8b
GLShaderBinaryCache只会在开启PSO且PSO第一次预热时生成，后续不会更改其内容，UserBinaryCache则是允许在生成前者后如果遇到未被Cache的Shader可以增量收集，并在下次运行时和之前的GLShaderBinaryCache合并，简单的功能没啥好说的
## Todo
目前上述功能都已完备，之前一些奇奇怪怪的崩溃问题基本上都解决了，项目中现在只开启了ShaderLink相关的功能，如果后续低端机性能测试时发生了RHI线程的资源上传卡顿则可以尝试开启Buffer和Texture相关功能（该功能开发初衷就是参考光子的分享，解决中低端机的资源创建导致的卡顿问题）