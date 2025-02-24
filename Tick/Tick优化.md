## 原生Tick系统

### 流程

1. 每帧选出需要执行的函数
2. 构造Task
3. 放入TaskGraph执行

![[TickSystem.drawio.png]]

### 开销

除去Tick函数执行本身的开销以外，可优化的开销有：

- QueueTickFunction调用链
	- 依赖关系遍历
	- Task构造
	- ...
- ReleaseTickGroup调用链
	- Task在指定线程入队出队
	- Task执行完毕后通知Subsequents
	- ...

## LogicTick系统

### 调用链简化

DS环境下，所有Tick函数都在GameThread执行；因此可以绕过Task和TaskGraph，直接调用TickFunction；

现有的LogicTick方案已经实现了直接调用，TickFunction放入数组，每帧遍历执行；

### 行为一致性

简化后的Tick系统需要和原生系统行为一致；

- TickFunction在两种系统内，执行时间点应该一致
	- 考虑Overrun
	- 考虑动态设置TickInterval
	- 考虑动态开启关闭
- TickFunction在两种系统内，TickGroup应该一致
	- 考虑预设的TickGroup和EndTickGroup
	- 考虑依赖关系
	- 考虑Demotion
- 有依赖关系的TickFunction在两种系统内，执行的先后顺序应该一致
	- 优先业务层处理，业务想办法干掉函数的依赖关系
	- 如果函数有干不掉的依赖关系，引擎层再来做保证

> 如何验证上述一致性？

- 考虑增加交叉比对，验证总开关开启前后的执行结果是否满足上述的一致性；

## 数据采集工具

增加类似NetworkProfiler的TickProfiler，用于抓取TickFunction的耗时、次数、依赖关系等数据，并加以分析；

![[TickProfileChart.png]]

![](TickProfileSample.png) 

## 优化思路

### 框架层修改

基于现有的LogicTick继续优化，考虑：

- 逻辑正确性
- 缓存友好性
- 算法优化

### 业务层修改

为了缩小影响面，考虑先仅对小范围的TickFunction实验启用LogicTick；

总开关和FTickFunction上的子开关共同控制是否开启LogicTick。主开关仅在部分模式的DS开启，子开关需要开发主动勾选。子开关可热更；

结合TickProfile数据和代码：

- 确认函数没有依赖，则直接改成LogicTick
- 如果函数有依赖
	- 首先考虑逻辑层主动消除依赖(主动调用、委托、修改TickGroup...)
	- 如果依赖无法消除，考虑两种方案
		- Plan A: 函数和它的上下游依赖进行拓扑排序，决定好执行顺序后再由LogicTick处理
			- 所有Tick都走LogicTick系统，但是拓扑排序会引入额外消耗
		- Plan B: 函数和它的上下游依赖仍然走原生Tick系统，LogicTick系统不保证先后顺序
			- 和原生系统妥协，好处是不引入额外消耗

### 工具搭建

- TickProfiler
- AB Tester

## 问题

### 数据采集问题

TickProfiler录制的内网压测数据和外网真实数据存在差异：

- 数量差异
- 质量差异
	- 部分Tick需要特殊条件才能激活
	- 部分依赖需要特殊条件才能建立

只能在没有外网数据的情况下，尽可能丰富自动化用例的行为；

### 交叉比对问题

Tick系统是有状态系统，受环境的影响较大，游戏逻辑的随机因素和DeltaTime的不稳定性都让同一个用例产出的结果不能完全一致；

考虑通过 手动编写 或 录制 的方式，创造稳定的环境来执行单元测试；
