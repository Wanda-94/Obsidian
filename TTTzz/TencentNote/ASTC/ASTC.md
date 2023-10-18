Iwiki:https://iwiki.woa.com/p/4007769630
## 运行时RVT ASTC压缩
Iwiki:https://iwiki.woa.com/p/4007686450
commit：b938c36885d298919d94156b13296259de604d74，27b852cad06c8df2c81002eeda61498e8a291ccf，及后续相关修改
改动为
1.合入了NextStudio那边的运行时RVT ASTC压缩
2.在1的基础上增加了4x4 Dual Plane模式的压缩，以提高压缩质量，经测试BaseColor，Normal等的压缩效果均优于原始
3.<font color=red>还增加了6x6压缩的模式，但是应该用不到，效果不太好，但是效率很高，后续低端机上可以考虑使用（6x6对比4x4压缩单Block计算消耗是一样的，这个是由ASTC的格式决定的，但是内存占用和Block数都要小）</font>
## ASTC压缩工具替换
Iwiki:https://iwiki.woa.com/p/4007686450
UE4 默认的ASTC压缩工具是Intel ISPC，相比另一个ARM的压缩工具encode astc可控配置的参数较少，ARM的astcenc可以实现更高质量的压缩，但相对的压缩耗时增高，目前做了两个改动
1.将UE4的ARM astcenc版本从1.0升级成最新版本（压缩效率更高，质量更高）
2.将UE4 默认的压缩工具改成ARM
## 运行时ASTC格式Texture合并
Iwiki:https://iwiki.woa.com/p/4009015337
为Avatar开发的NRC定制运行时ASTC纹理合并，支持各种Block压缩格式且PC端移动端显示效果一致，合并耗时目前看来可接受（Android XiaoMi 9 - 1024x1024 4ms），大部分时间消耗在从磁盘上加载Raw Data，而这个消耗不能很好的避免，即使做Cache，Cache的利用率也不会很高（不太会出现反复合并一张图的情况）

