![[Pasted image 20230717193344.png|1500]]

![[ Pasted image 20230718102853.png|700]]
两者的GC都会造成卡顿，引擎的GC会在LevelStreaming的时候定时触发，Lua则是每隔数帧触发
## Lua
现在我们用的是garbagecollection("collect")是一种全量GC，而garbagecollection("step")可以每帧只执行一次GC的最小步骤
我们以Editor下场景为例
- garbagecollection("collect") / 150 frames
	![[Pasted image 20230718110101.png]]
	50~60ms每次GC
-  garbagecollection("step") / 150 frames
	![[Pasted image 20230718110204.png]]
	20~40ms每次GC
-  garbagecollection("step") / 50 frames
	![[Pasted image 20230718110303.png|500]]
	10~20ms每次GC
- garbagecollection("step") / 1 frames
	![[Pasted image 20230718110428.png]]
	1~6ms每次GC
通过修改GC的方式，从每次全量GC改为多次多步GC可以降低其峰值（60ms->6ms），同时上面的对比也说明了需要被GC的Lua对象大多数都是新产生的（如果存在大量常驻内存的Lua对象那么GC耗时应当每次都维持在一个较高的耗时，这种情况需要针对常驻内存的Lua对象进行标记，GC时跳过）
### 真机测试
#### XiaoMi 13
- garbagecollection("collect") / 150 frames
	![[Pasted image 20230717193344.png|1000]]
	40ms~60ms
- garbagecollection("step") / 30 frames
	![[Pasted image 20230718111819.png|1175]]
	每次还是有10ms以上的耗时
- garbagecollection("step") / 10 frames
	![[Pasted image 20230718113101.png|1250]]
	每次10ms以内
- garbagecollection("step") / 1 frames
	![[Pasted image 20230718115830.png]]
	1~6ms，基本2ms左右
Lua的GC耗时最好是定一个期望值，然后根据实际GC耗时和期望值的差距动态调整GC的间隔，因为同样的全量GC在不同机型上耗时应该是不同的，高端机也许不会卡但是低端机就不一定了
``` lua TI:"Lua GC" HL:""
    --- Common/UpdateManager.lua line:118
    if(_G.StartAutoGCByTick and _G.StartAutoGCByTick > 0)then
        UpdateManager.CurGCTickIdx = UpdateManager.CurGCTickIdx + 1
        if(UpdateManager.CurGCTickIdx >= _G.StartAutoGCByTick)then
            UpdateManager.CurGCTickIdx = 0
            _G.NeedToBeGC = true
        end
        if(_G.NeedToBeGC and not _G.BlockGC)then
            collectgarbage("step")
            if(_G.StartAutoGCByTickTwice)then
                collectgarbage("step")
            end
            _G.NeedToBeGC = false
        end
    end
```
修改为
```lua HL:"8,9,11,12"
    if(_G.StartAutoGCByTick and _G.StartAutoGCByTick > 0)then
        UpdateManager.CurGCTickIdx = UpdateManager.CurGCTickIdx + 1
        if(UpdateManager.CurGCTickIdx >= _G.StartAutoGCByTick)then
            UpdateManager.CurGCTickIdx = 0
            _G.NeedToBeGC = true
        end
        if(_G.NeedToBeGC and not _G.BlockGC)then
	        local TimeCounter = CreateTimeCounter()
	        TimeCounter.Start()
            collectgarbage("step")
            TimeCost = TimeCounter.Stop()
            _G.StartAutoGCByTick =
        AdjustGCHeartBeat(TimeCost,_G.GCTimeLimit)
            _G.NeedToBeGC = false
        end
    end
```
跑了下修改GC的频率内存也还维持在60~70mb左右