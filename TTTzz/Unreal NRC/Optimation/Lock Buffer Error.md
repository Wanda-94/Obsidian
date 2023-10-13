UE表现为，lock buffer时会返回null（glMapBufferRange返回null，且gl驱动不报错）
![[Pasted image 20230908160625.png|900]]
同时android log中有
![[Pasted image 20230908160652.png|875]]
这个似乎在高通低版本GPU+低版本驱动有几率出现
UE/Unity都有人遇到过这个问题
https://answer.uwa4d.com/question/60195c1310a17c6c2b09de1c
https://udn.unrealengine.com/s/question/0D52L00004luws6SAA/msaa-on-android
https://udn.unrealengine.com/s/question/0D54z000080E0zHCAS/fopenglmapbufferrange%E8%BF%94%E5%9B%9E%E4%BA%86nullglgeterror0
![[Pasted image 20230908161803.png]]
之前jchan也在低端机上遇到过加载landscape grass相关height map时触发类似bug，改动方法是降低了NRC渲染压力（降低Quality）
![[Pasted image 20230908161027.png]]
- 产生原因
	- 似乎是高通自身的Bug，特定的渲染功能或同一时间GPU渲染压力大时（MSAA，PS中使用Depth Offset，使用float frame buffer format，compute shader执行）有一定概率引起GPU Freeze，画面整个卡住，此时lock buffer就会返回null，导致后续崩溃
- 解决方法
	- 官方是建议将渲染特性一个个关闭，找到是使用了什么特性导致的，然后去掉
		![[Pasted image 20230908161850.png]]
		![[Pasted image 20230908161845.png]]
		![[Pasted image 20230908161946.png|825]]
		也有人提到适时调用glFlush，避免一次性提交太多指令导致GPU卡住
		![[Pasted image 20230908162129.png]]
		NRC这里的情况是在Oppo Reno 9上播放一个特定技能会稳定触发这个Bug，尝试不使用MapBufferRange来更新PixelBuffer，似乎可以避免稳定崩溃（还是有概率崩掉）
		![[Pasted image 20230908162532.png]]
		![[Pasted image 20230908162443.png]]
# 后续
在清风山战斗释放水波术之后crash
从官方给出的信息来看是GPU停止工作，卡住了，需要排查是那些操作卡住了GPU
在分支包上经过测试
在清风山选择完宠物任务附近使用水蓝蓝技能水波纹会有大概率触发GPU卡死
在不同位置战斗则不会触发，且在该位置
show.staticmesh 1
show.landscape 1
之一存在必崩
调整MSAA依然会崩
技能中也没有使用特殊操作
r.ScreenPercentage 调低100->50不会崩
综上引起这个现象的原因可能是在该场景下渲染压力较大，且该技能也很费导致一时渲染压力过大，使得GPU卡死，并非是使用了某些特定的GPU特性（Depth Offset，MSAA等）
而主干上同样条件则不会崩溃，很离谱（应该是某些地方做了优化？）
最新消息，提高r.ScreenPercentage 100->150 主干也会崩了😊
