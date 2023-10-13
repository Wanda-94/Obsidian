``` c++ 
bool IncrementalDestroyGarbage(bool bUseTimeLimit, float TimeLimit)
{
	const bool bMultithreadedPurge = !ShouldForceSingleThreadedGC() && GMultithreadedDestructionEnabled;
	if (!GAsyncPurge)
	{
		GAsyncPurge = new FAsyncPurge(bMultithreadedPurge);
	}
	else if (GAsyncPurge->IsMultithreaded() != bMultithreadedPurge)
	{
		check(GAsyncPurge->IsFinished());
		delete GAsyncPurge;
		GAsyncPurge = new FAsyncPurge(bMultithreadedPurge);
	}
```

![[GC Crash 2023-08-04 15.08.04.excalidraw]]
![[Pasted image 20230808144004.png]]
GC时同一个UObject在PurgePendingKillArray里存在复数，导致会被Destroy了多次，第二次Destroy时会因为UObject的Flag Check不过而崩掉
存在复数的情况是因为，该UObject所属的Cluster中包含多次同一个UObject，Cluster被Destroy加入PurgePendingKillArray时加入了多次
而多次包含的UObject为UFunction类，出现这种情况是由于Unlua在Replace Function时
![[Pasted image 20230808191941.png]]
![[Pasted image 20230808192009.png]]
![[Pasted image 20230808193830.png]]
