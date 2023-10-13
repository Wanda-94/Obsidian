No DCV
![[Pasted image 20230911185919.png]]
![[Pasted image 20230911185915.png]]
![[Pasted image 20230911185926.png]]
![[Pasted image 20230911185909.png]]
![[Pasted image 20230911185903.png]]
![[Pasted image 20230911185853.png]]

![[Pasted image 20230911185604.png]]
![[Pasted image 20230911185722.png]]
![[Pasted image 20230911185732.png]]
![[Pasted image 20230911185747.png]]
![[Pasted image 20230911185801.png]]
![[Pasted image 20230911185808.png|1775]]
![[Pasted image 20230911185816.png]]
![[Pasted image 20230911185831.png]]
草
![[Pasted image 20230911185655.png]]
![[Pasted image 20230911185640.png]]

![[Pasted image 20230911185612.png|1025]]
![[Pasted image 20230911185645.png|800]]
r.NumBufferedOcclusionQueries
![[Pasted image 20230913105728.png]]
缓存N Frame Occlusion Query信息，1则直接等待当前帧结果，N则等待N帧前结果
r.Mobile.AdrenoOcclusionMode 
修改Occlusion Query的时机
![[Pasted image 20230912152557.png]]
![[Pasted image 20230913105636.png]]
这个修改在Oppo Reno 10上压测场景好像有点用 RHI 31~32->28~30
幻塔极致画质MSAA 2X
![[Pasted image 20230913105821.png]]
![[Pasted image 20230913111545.png]]
![[Pasted image 20230913112945.png]]
![[Pasted image 20230913113001.png]]
L1 L2 Texture Cache Miss
![[Pasted image 20230913111600.png]]
Per Pixel Texture Miss

![[Pasted image 20230913111646.png]]
![[Pasted image 20230913112042.png]]
宠物NPC没有LOD，DC多，Animation更新耗，（高等级LOD直接用StaticMesh？DrawInstance，减少Animation Update）
![[Pasted image 20230913160542.png]]

# 数据
开启GPUScene的情况下
r.mobilemsaa 
	4->2->1
	frame time : 85ms->60ms->40ms（非RenderThread/RHIThread瓶颈）
r.AdrenoMode & r.NumBuffered
在Oppo Reno 10上AdrenoMode RHI -2ms，其余无变化
压测场景 msaa 4x & r.allowocclusionquery 1->0
	frame time : 100ms->80ms
压测场景 msaa 2x & r.allowocclusionquery 1->0
	frame time : 75ms->81ms

关闭GPUScene RHIThread
msaa 2 adrenomode 1 backbuffer 3 ->25ms
msaa 4 adrenomode 0 backbuffer 1 ->28ms
