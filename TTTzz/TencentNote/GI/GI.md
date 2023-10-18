在LPV的基础上进行改版实现了一波能跑在移动端上的GI
https://iwiki.woa.com/p/1692568654
branch：NRC_Mobile_LPV_Refactoring_2022_12_01
# LPV
LPV是通过在体素化（Volume）的空间中生成虚拟点光源（Light），然后在体素中对光源的传播进行模拟（Propagation），传播的结果以球谐的方式存储在体素中，后续场景中任意顶点计算GI时只需采样球谐并计算一个球谐光照即可
LPV的流程主要分步
1. RSM渲染
	从光源角度渲染一张BaseColor，Normal，Depth，这个后续会用于生成VPL(Virtual Point Light)的Color，Direction，Position，对应引擎的修改则是在Mobile阶段开启RSM渲染（PC端本身有这个）
2. 虚拟点光源生成（VPL Gen）
	根据1中生成的RSM生成VPL，需要注意的是VPL的强度计算，需考虑该点的影响范围（根据Shadow Project范围，深度等计算）
	<font color=red>ComputeShader,List</font>
3. 点光源注入（VPL Inject）
	将2生成的点光源注入到Volume中，以球谐存储，为了避免自光照一般会将虚拟点光原朝Normal方向偏移一个Voxel存入Volume
	<font color=red>ComputeShader,SH,Accumulate</font>
4. 传播（Propagation）
	以3为起始状态，每个Voxel累加计算相邻Voxel对其的光照结果，以球谐存储，重复以上步骤多次待Volume状态稳定（UE的实现则是进行固定次数3/4次，然后下一帧以前一帧的计算结果为起始状态，重复进行以降低单次的计算复杂度，但是其未考虑能量守恒导致加大传播次数会导致Volume的值无限增大），并且UE自带的AO遮蔽效果由于Volume太大而效果很差（类似于用Voxel化的场景做遮蔽）
	<font color=red>ComputeShader,Propagation</font>
5. 采样
	根据Position，Normal采样4计算得到的Volume3DTexture，并计算一次球谐光照
	<font color=red>SH</font>
# 遇到的问题及解决方案
1. RingArtifact
产生的原因和一些解决方案可以参考Stupid SH，这里是使用了基于拉普拉斯算子梯度下降的方式避免了RingArtifact（计算效率可接受，效果很好）
2. 光照效果平
UE的LightMap里有一种DirectionalLightMap，不仅存了一个光照结果还保存了最强光照方向
![[Pasted image 20231013155238.png]]
![[Pasted image 20231013155248.png]]
而保存光照信息的球谐很容易计算出最强光照方向，所以LPV计算光照时可以将最强光照方向加入考虑，调整最后计算的GI强度
3. Propagation计算耗时
https://developer.nvidia.com/content/basics-gpu-voxelization
有尝试过生成场景的体素化信息（以PropagationNum * Light Volume Size为Voxel Size）来避免一些对光照结果没有太大影响且不会采样的Voxel中的Propagation计算
4. 范围问题
这个是比较严重的缺陷，如果要保证GI计算精度那么Voxel的精度不能太低，而精度高带来的是能覆盖的空间变小，移动端的计算负载是一定的，32x32x32大小Volume的计算量勉强是移动端可以带动的一个分辨率，如果一个Voxel为1x1x1那么整体也只能覆盖很短的距离（玩家一般位于Volume中心，也就是只有面前16m有GI效果），原版LPV使用了Cascade来解决，但是多级之间存在一个接缝问题（参考Cascade Shadow），而且这个接缝问题还有点难解决，两级Cascade计算的光照结果会有较大的差距，如果采样两个Volume进行平滑这个消耗又有点太大了，后面选用了类似Clipmap的方式来扩大影响范围，同时在XYZ轴上调整了Voxel的数量，将更多的Voxel分布在XY上
![[Pasted image 20231013164443.png]]