```C TI:"Texture2D.h:17" HL:"7"
UCLASS(hidecategories=Object, MinimalAPI, BlueprintType)
class UTexture2D : public UTexture
{
	GENERATED_UCLASS_BODY()

public:
virtual FTextureResource* CreateResource() override;
```

```C TI:"Texture2DResource.h" HL:"18"
class FTexture2DResource : public FStreamableTextureResource
{
public:
	/**
	 * Minimal initialization constructor.
	 *
	 * @param InOwner			UTexture2D which this FTexture2DResource represents.
	 * @param InPostInitState	The renderthread coherent state the resource will have once InitRHI() will be called.		
 	 */
	FTexture2DResource(UTexture2D* InOwner, const FStreamableRenderResourceState& InPostInitState);

	/**
	 * Destructor, freeing MipData in the case of resource being destroyed without ever 
	 * having been initialized by the rendering thread via InitRHI.
	 */
	virtual ~FTexture2DResource();

	virtual void CreateTexture() final override;
```

```C TI:"OpenGLTexture.cpp:2589"
void FOpenGLDynamicRHI::UnlockTexture2D_RenderThread(class FRHICommandListImmediate& RHICmdList, FRHITexture2D* Texture, uint32 MipIndex, bool bLockWithinMiptail, bool bNeedsDefaultRHIFlush)
{
	check(IsInRenderingThread());
	static auto* CVarRHICmdBufferWriteLocks = IConsoleManager::Get().FindTConsoleVariableDataInt(TEXT("r.RHICmdBufferWriteLocks"));
	bool bBuffer = CVarRHICmdBufferWriteLocks->GetValueOnRenderThread() > 0;
	FTextureLockTracker::FLockParams Params = GLLockTracker.Unlock(Texture, 0, MipIndex);
	if (!bBuffer || Params.LockMode != RLM_WriteOnly || RHICmdList.Bypass() || !IsRunningRHIInSeparateThread())
	{
		GLLockTracker.TotalMemoryOutstanding = 0;
		RHITHREAD_GLCOMMAND_PROLOGUE();
		this->RHIUnlockTexture2D(Texture, MipIndex, bLockWithinMiptail);
		RHITHREAD_GLCOMMAND_EPILOGUE();
	}
	else
	{
		auto GLCommand = [=]()
		{
			uint32 DestStride;
			uint8* TexMem = (uint8*)this->RHILockTexture2D(Texture, MipIndex, Params.LockMode, DestStride, bLockWithinMiptail);
			uint8* BuffMem = (uint8*)Params.Buffer;
			check(DestStride == Params.Stride);
			FMemory::Memcpy(TexMem, BuffMem, Params.BufferSize);
			FMemory::Free(Params.Buffer);
			this->RHIUnlockTexture2D(Texture, MipIndex, bLockWithinMiptail);
		};
		ALLOC_COMMAND_CL(RHICmdList, FRHICommandGLCommand)(GLCommand);
	}
}
```
OpenGL Lock Texture
```C TI:"OpenGLTexture.cpp : 2563"
void* FOpenGLDynamicRHI::LockTexture2D_RenderThread(class FRHICommandListImmediate& RHICmdList, FRHITexture2D* Texture, uint32 MipIndex, EResourceLockMode LockMode, uint32& DestStride, bool bLockWithinMiptail, bool bNeedsDefaultRHIFlush)
{
	check(IsInRenderingThread());
	static auto* CVarRHICmdBufferWriteLocks = IConsoleManager::Get().FindTConsoleVariableDataInt(TEXT("r.RHICmdBufferWriteLocks"));
	bool bBuffer = CVarRHICmdBufferWriteLocks->GetValueOnRenderThread() > 0;
	void* Result;
	uint32 MipBytes = 0;
	if (!bBuffer || LockMode != RLM_WriteOnly || RHICmdList.Bypass() || !IsRunningRHIInSeparateThread())
	{
		RHITHREAD_GLCOMMAND_PROLOGUE();
		return this->RHILockTexture2D(Texture, MipIndex, LockMode, DestStride, bLockWithinMiptail);
		RHITHREAD_GLCOMMAND_EPILOGUE_GET_RETURN(void *);
		Result = ReturnValue;
		MipBytes = ResourceCast_Unfenced(Texture)->GetLockSize(MipIndex, 0, LockMode, DestStride);
	}
	else
	{
		MipBytes = ResourceCast_Unfenced(Texture)->GetLockSize(MipIndex, 0, LockMode, DestStride);
		Result = FMemory::Malloc(MipBytes, 16);
	}
	check(Result);

	GLLockTracker.Lock(Texture, Result, 0, MipIndex, DestStride, MipBytes, LockMode);
	return Result;
}
```
![[Pasted image 20230707155907.png]]
1. Lock Texture
	创建LockTracker，并且为其创建一个缓冲Buffer，用来Copy要上传的Texture的Data
2. Copy Data
	将Texture的Data Copy到LockTracker的缓冲Buffer中
3. Unlock Texture
	将缓冲Buffer里的数据上传到GPU
	![[Pasted image 20230707162819.png]]
	![[Pasted image 20230707163646.png]]
	![[Pasted image 20230707163654.png]]
![[Pasted image 20230707172607.png]]