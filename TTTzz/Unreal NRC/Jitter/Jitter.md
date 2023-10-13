# GameThread
GameEngineTick中NPCModule:OnNPCLeave耗时较高
![[Pasted image 20230717145612.png|1175]]
Lua GC 和 GameGC都有很高的耗时
Lua
![[Pasted image 20230717150724.png|1125]]
![[Pasted image 20230717150234.png|1150]]
![[Pasted image 20230717150405.png|1300]]
GameGC总是接在一个LevelStreaming后面
![[Pasted image 20230717150610.png|1325]]
![[Pasted image 20230717150629.png|1300]]
![[Pasted image 20230718163819.png]]
# RenderThread
PostInitViewsFlushDel等待一个事件
![[Pasted image 20230717150512.png|1325]]
![[Pasted image 20230717150643.png|1275]]
![[Pasted image 20230717150657.png|700]]
# RHIThread
Present time中总是有一个Stall
![[Pasted image 20230717145745.png|1200]]![[Pasted image 20230717150528.png|1025]]