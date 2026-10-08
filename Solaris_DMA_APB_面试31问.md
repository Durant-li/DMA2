# Solaris 面经：DMA-APB 项目 31 问

> 使用方法：每题先说“面试回答”，被追问时再展开“代码边界”。本文件以当前仓库为准，不把验证计划中的目标当作已实现结果，也不推断代码作者。原始设计规格、真实仿真日志和个人职责记录若与本文件不同，应以实际证据为准。

## 先记住项目的四条主线

1. **DUT**：单通道DMA；上游APB Slave接CPU配置/Direct访问，下游APB Master访问统一Memory空间；固定32-bit数据，COPY长度`N`对应`N`读加`N`写。
2. **激励**：Master Agent主动发CPU事务；Reactive Slave Agent从Monitor请求经analysis FIFO取得下游访问，按Memory内容及`ready_delay`返回`PRDATA/PREADY`。
3. **检查**：Env里的Scoreboard收上游Setup、上游完成、下游完成和IRQ事件，维护寄存器Shadow、Reference Memory、Expected Transfer Queue、COPY Read Data Queue。
4. **边界**：当前无UVM RAL、Virtual Sequencer/Sequence、独立Memory Model或`PSLVERR`；计划里有些Reset/Busy/非法命令场景尚未形成完整checker，不要说成已完成Sign-off。

## 一、项目与UVM验证环境

### 1. 从陌生人的视角介绍模块、接口和验证环境

**面试回答：** CPU通过上游APB写`SRC/DST/LEN`并下发`INIT/COPY`命令；DUT通过一个下游APB Master端口访问外部Memory。`INIT`向`SRC`起始区域写入指定值，`COPY`按读Source、写Destination的顺序搬运。Direct窗口则把CPU读写转成下游访问。环境中Master Agent模拟CPU，Reactive Slave Agent模拟Memory，Scoreboard做上下游端到端检查，Coverage Collector统计功能和时序场景。两侧`PREADY`分别由DUT和Slave Agent提供。

**边界：** SRC、DST是一个下游地址空间里的逻辑地址，不是两个物理Memory端口；地址每次`+1`，byte/word编址语义仍需原始规格确认。

### 2. 哪些是已有的，哪些由你补全？

**面试回答模板：** “接手时已有的部分是`[按实际填写：RTL / 基础APB Agent / 简单Sequence等]`。我具体补充了`[按实际填写：哪几个Sequence、Scoreboard检查、Coverage、回归脚本、Debug修正]`。我能举出改动的类、接口连接和验证效果。”

**不能替你确认：** 仓库只能证明“现在有什么”，不能证明每一行由谁写、接手前版本是什么。准备二面时，最好对照你自己的提交、任务记录和实习沟通，把“独立完成/参与完善/复用已有”分清。当前代码里有Master/Slave Agent、Scoreboard、Coverage、测试及脚本，但不能据此全部认领。

### 3. 除RTL学习和环境搭建，还做了什么？发现问题后如何处理？

**面试回答：** 我把功能拆成Direct、INIT、COPY、异常中断、Wait-state、Busy和Reset等场景，补定向/随机激励以及端到端预测。失败时先固定`TEST+SEED`重现，查看UVM首次报错，再把上游Setup/完成、下游Setup/完成、IRQ时间线对齐；结合波形判断是DUT行为、验证模型预测时机，还是激励/规格理解问题。修正后先跑最小复现，再跑相关回归。

**可讲的两个具体观察：** Direct在上下游Wait-state不对称时，若等上游完成才建期望，下游先完成会让Scoreboard误报；LEN读事务的无意义`PWDATA`可能参与RTL的非法长度判断。后者在当前代码中有规避激励和RTL条件证据；若无实际日志/团队记录，不要说“我已提交并关闭RTL Bug”。

### 4. 验证计划怎么功能分解？覆盖哪些点？

**面试回答：** 先按**控制面**（寄存器、Direct、命令、IRQ/W1C）、**数据面**（INIT/COPY地址、方向、数量、数据、顺序）、**时序面**（上下游Wait、Reset、Busy期间CPU访问）拆分；每个点写清“如何激励—谁检查—哪个Coverage bin证明测到”。功能覆盖包括命令×长度、COPY长度×地址关系、两侧读写×等待周期、异常状态位及Busy×CPU访问类型。

**边界：** Coverage命中只能证明场景发生，正确性由Scoreboard/SVA判断；仓库没有可核实的最终覆盖率数字。`DMA_APB_Verification_Testplan.md`含目标和待确认规格，不等于全部close。

### 5. 寄存器、地址、长度应有哪些定向和边界测试？

**面试回答：** 寄存器测复位默认值、写后读回、重复改写、W1C写0/写1；地址测Direct映射、SRC/DST相等、前后Overlap、相邻、分离及保留地址；LEN测`0、1、2～15、16、17`及超位宽值，特别检查写入原始值与5-bit寄存器截断之间的关系。数据可选0、全1、交替位、one-hot和普通随机值。再交叉Wait、Busy和Reset阶段。

**边界：** 当前`LEN`寄存器只有5 bit；`LEN=33`会按原始写数据报非法长度，但寄存器低5位实际为1。地址回绕、高位截断的期望未由正式规格确认。

### 6. 除基础项目外补什么场景？有限test怎样覆盖？

**面试回答：** 优先补风险交叉而不是为每个bin建一条test：一次寄存器/Direct测试覆盖方向和数据类；一次INIT/COPY测试覆盖`LEN=1/16`、地址关系和数据搬运；一次异常/中断测试覆盖四个原因及W1C；独立Wait、Busy、Reset测试覆盖时序；少量不同Seed的随机测试补未命中的组合。看Coverage hole和失败原因定向补测，不单纯堆Seed。

**边界：** 当前Reset测试主要在Idle/Setup/Access对一个无副作用INT读插入Reset；“DMA搬运中Reset”尚缺Scoreboard清空未完成期望的建模。Busy时重新下INIT/COPY、LEN=0命令也在README里列为风险/waiver，不能宣称已完整验证。

## 二、中断机制与并发激励

### 7. 为什么额外有Interrupt Handler，不直接在Sequencer中做？

**面试回答：** Sequencer的analysis `write`回调是`function`，只能快速记录IRQ事件，不能等待时钟或执行耗时的APB读写。Monitor检测边沿、Sequencer保存`pending/level/count`，Handler在自己的`run_phase`等待`pending`，再启动IRQ Sequence。Sequencer管仲裁和状态，Handler管耗时的软件响应流程。

### 8. 中断test如何构造？

**面试回答：** 先制造明确的原因：正常INIT/COPY写完成、COPY地址Overlap、写`LEN=0/>16`、访问保留区域或读只写命令地址。DUT置位`DMA_INT`并拉高IRQ；Monitor通知Sequencer，Handler启动IRQ Sequence读取`DMA_INT`，记录读到的状态，再写相同的1位做W1C。Scoreboard独立预测“应当是哪一位”，比较读回并在最终检查未清状态。

**边界：** 当前RTL的`done`位在每个INIT下游写或COPY目的写完成时即可置位，并非严格“整笔DMA结束”才置位；正式语义需规格确认。

### 9. Monitor怎么通知Sequencer/Handler？

**面试回答：** Master Monitor在每个采样时钟比较IRQ与上一拍，检测0→1/1→0，调用`interrupt_ap.write(irq)`广播。Agent把该Port连到Master Sequencer的`interrupt_imp`；其`write_apb_irq_seqr`回调更新`interrupt_level`，上升沿置`interrupt_pending`并计数。Handler用`wait(sequencer.interrupt_pending)`异步等待，启动IRQ Sequence；普通业务Sequence通过`p_sequencer`读取共享状态做同步。

### 10. 怎样验证IRQ“应该拉高”而不只是看到拉高？

**面试回答：** 必须有独立预测：Scoreboard根据上游非法操作/长度/Overlap，以及下游INIT/COPY写完成，更新`expected_interrupt`；IRQ Sequence随后读`DMA_INT`，Scoreboard比较实际状态位和期望状态位。不能仅因Monitor看见IRQ上升沿就判成功。

**当前缺口：** 当前Scoreboard保存IRQ引脚电平并在测试结束检查未清零，但没有对“每个期望事件后，IRQ必须在规定延迟内上升/下降”建立独立时间窗检查。面试可以说这是后续要加的SVA/时序checker，不能说已完整检查IRQ时限。

### 11. 检测到IRQ后，IRQ Sequence做什么？

**面试回答：** 在同一个Master Sequencer上高优先级发`DMA_INT`读事务；Driver返回的读数据写回Item，IRQ Sequence取低4位`serviced_status`；若不为0，再写同样的1位做W1C。这样不会盲写`0xF`，能够保留“实际触发了哪些原因”的证据供Scoreboard核对。

### 12. 怎么证明中断清除了？

**面试回答：** 先核对W1C前的状态读回与预测一致，再根据清除掩码更新预测状态；随后读回`DMA_INT=0`并确认IRQ引脚下降。在当前项目中，Scoreboard已做状态读回比较、W1C预测和结束时IRQ未保持为高的检查；若要求精确证明“W1C后几拍内下降”，应补即时读回/断言。

### 13. 加自动中断响应，需要改哪些组件？

**面试回答：** Monitor增加IRQ边沿采样和`interrupt_ap`；Master Sequencer增加`interrupt_imp`及`pending/level/count`状态；Master Agent把Port连到Sequencer，并持有Handler；Handler启动IRQ Sequence；IRQ Sequence实现INT读与W1C；Scoreboard独立预测状态并检查。业务Sequence还要等待Handler服务结束，避免两股Master流无意交叉。

### 14. 为什么放在Master Agent，不全放Test里？

**面试回答：** 中断响应是“CPU Master这一侧的可复用服务行为”，与具体哪个Test触发IRQ无关，封装在Master Agent可避免每个Test重复写ISR。Test负责选择触发场景，Handler负责按策略服务。

**权衡：** 永远自动清IRQ会妨碍测试中断保持、屏蔽、多个原因累积。因此更成熟的环境应做`auto_service/hold/poll_only`等可配置策略；当前代码默认自动服务，尚未实现模式开关。

### 15. Driver忙时Handler事务如何调度？

**面试回答：** Handler启动的IRQ Sequence与普通Sequence共享Master Sequencer，设置高优先级；Master Sequencer使用`SEQ_ARB_STRICT_FIFO`。但高优先级只影响**下一次仲裁**，不能中断Driver已经通过`get_next_item()`拿走、正在等待`PREADY`的APB事务。当前事务先完成，Driver调用`item_done()`后，IRQ读写才能获得总线；应检查普通访问、IRQ到来、W1C之间的时序和可能的状态变化。

### 16. 中断后必须立刻清吗？会不会过度模拟软件？

**面试回答：** 不一定。自动服务适合大量功能回归，防止sticky IRQ阻塞后续激励；但中断保持、多个原因叠加、W1C优先级、忙期间读状态都需要延迟服务或手动服务。当前Handler是默认自动读后清，适合作为基础模式；扩展时应让Test配置响应延迟或关闭自动服务，而不是让IRQ机制写死所有场景。

## 三、Scoreboard、预测模型与数据通路

### 17. Scoreboard怎么预测COPY次数、地址和数据？

**面试回答：** 上游完成SRC/DST/LEN写后，Scoreboard保存Shadow值。COPY命令完成时，按`i=0…LEN-1`依次生成`READ SRC+i`、`WRITE DST+i`，所以预期是`2N`笔下游事务。Slave Monitor每报告一笔完成事务，就从`expected_slave_q`队首比较读写方向和地址。读数据与Reference Memory比较，实际读值进入`copy_read_data_q`；下一笔写数据须等于该读值。每笔写后更新参考Memory；末尾还检查队列是否清空。

### 18. COPY发起时不知道源数据，怎样建数据期望？

**面试回答：** 命令时先预测访问**结构**，即次数、方向、地址和顺序，不必提前把每笔写数据都算进队列。测试先通过Direct Write预填Source，Scoreboard维护自己的Reference Memory；下游Read完成时先检查返回值是否等于参考值，再把本次实际读值暂存，下一笔Destination Write与它比较。

**重点：** 如果Read错而Write跟着错，Read对Reference Memory的检查仍能报错；如果Read对但Write错，读写传递检查报错。当前代码即使Read比较失败，仍会暂存实际读值，以便继续定位后续数据通路。

### 19. RTL怎样与Memory通信，为什么是N读+N写？

**面试回答：** DUT只有一个下游APB Master端口，Source和Destination是同一地址空间中的不同地址。COPY分两阶段：在`dma_copy_active_p1`向`SRC`发读请求、接收`slave_prdata`；在`dma_copy_active_p2`向`DST`发写请求、写出先前读值。一个32-bit元素对应一读一写，LEN=N即N对、共2N笔下游APB传输，不是两个Memory接口同时搬运。

### 20. Memory Model应放哪一层？

**当前实现：** Slave Basic/Coverage Sequence里的关联数组`memory[address]`是**响应Memory**，为下游读返回数据，写完成后更新；Scoreboard里的`reference_memory`是**检查用参考状态**。两者用途不同，不能共用同一对象，否则DUT/响应模型的错误可能一起污染期望。

**可改进设计：** 把响应Memory抽成Slave Agent内独立对象，由Reactive Sequence查询并更新，便于多Sequence共享、预载、复位策略和复用；Scoreboard的预测Memory仍独立。Driver只负责Pin级握手，通常不拥有业务Memory状态。

### 21. 两个Agent与Scoreboard怎么连？属于哪层？要同步吗？

**面试回答：** Scoreboard属于Env，与两个Agent并列。Env连接Master的`request_ap`到Scoreboard的`master_request_imp`、`completed_ap`到`master_complete_imp`、`interrupt_ap`到`interrupt_imp`；Slave的`completed_ap`到`slave_complete_imp`。这些是Analysis Port到Analysis Implementation的事务通知，不要求两个Monitor在同一周期“同步”。Scoreboard按因果事件建队列并匹配；Direct在Master Setup就建期望，正是为了解决下游先于上游完成的时序差异。

**区分：** `uvm_tlm_analysis_fifo`在Slave Sequencer中连接Slave Monitor请求，供Reactive Sequence阻塞`get()`；Scoreboard本身没有用Analysis FIFO。

### 22. Golden Memory存期望还是实际？更新链路是什么？

**理想回答：** Golden/Reference Memory应代表预测的架构状态，不能简单复制DUT结果。上游Direct Write给出预期地址和写数据；下游完成时先检查实际方向/地址/数据，再按定义更新预测Memory。COPY Read与该Memory比较，COPY Write既要与前次读值比较，也要反映写后状态。

**当前代码的真实边界：** `reference_memory`在下游写完成后保存的是**Monitor观察到的实际写值**，即便比较失败也继续更新；这使它兼有参考状态与已观察状态的角色，可减少后续级联报错，但独立性不够强。若面试官深挖，应坦诚说改进方式是维护`predicted_memory`与`observed_memory`两份，或只在匹配后按期望写值更新Golden Memory，避免错误数据覆盖期望。

### 23. APB Monitor在什么阶段采样？

**面试回答：** Setup是`PSEL=1 && PENABLE=0`，可取得地址、方向、写数据并发布`request_ap`；完成是Access阶段的`PSEL=1 && PENABLE=1 && PREADY=1`，发布`completed_ap`。Read的`PRDATA`只在完成握手时有效；Write数据也以完成事务为准。Wait期间控制、地址和写数据应保持稳定。当前Master/Slave Monitor均按这两类条件采样。

### 24. APB error response要不要验证？

**面试回答：** 如果接口有`PSLVERR`，必须验证它仅在完成握手时有效、哪些地址/访问触发、主设备是否正确处理，以及错误事务对Memory/寄存器是否有副作用。但**当前DUT与Interface没有`PSLVERR`引脚**；项目的非法操作、非法LEN通过`DMA_INT`状态报告，不是APB error response。若规格将来引入`PSLVERR`，要同步扩展Item、Driver/Slave响应、Monitor、Scoreboard和Coverage；现在不能说已验证它。

### 25. COPY期望数据从哪来？如何保证模型一致？

**面试回答：** 预填Source时，Master发Direct Write，DUT转成下游Write，Slave Memory在握手后保存该值；Scoreboard同时核对该下游Write与上游期望，并更新独立Reference Memory。COPY的Source Read应由Slave Memory返回预填数据，Scoreboard用自己的Reference Memory核对，之后Destination Write再与本次Read数据核对。两份Memory不能直接共享同一个数组。

**当前风险：** 两份Memory都基于已观察下游写事务更新，若前面写错，Scoreboard会先报错，但后续状态可能同时跟随错误值；需要看**首次报错**，不能只看最后Memory相同。

## 四、长度计数、Busy与异常边界

### 26. LEN非法时应拒绝、报中断还是继续执行？

**面试回答：** 先问规格，不能凭经验下结论。较稳妥的设计是非法值不启动DMA、给出错误状态/中断，必要时保留旧LEN；是否拒绝寄存器写由规格定义。**当前RTL**在写`LEN=0`或原始`PWDATA>16`时置`invalid_length`，但同时仍把`PWDATA[4:0]`写入5-bit LEN，没有明确禁止之后的INIT/COPY。因此“报了中断就不会搬运”对这个项目不成立。

### 27. LEN每beat递减还是结束时清零？怎样测？

**当前RTL：** INIT每完成一笔下游写，`dma_len`减1；COPY每完成一笔Destination Write减1，Source Read不减。COPY过程中SRC和DST也按各自阶段推进。因此LEN是**剩余元素计数**，不是完成时一次清零。可配置`LEN=3`，在每笔下游写完成后观察3→2→1→0；插入长`PREADY`等待，确认未握手时不多减、不重复传。

**验证边界：** 当前Scoreboard在Slave Completion处更新Shadow LEN，偏事务级；忙期间CPU精确读LEN与RTL内部采样时点的对应关系尚未完整建模，README对此有waiver。不要说已经精确检查每拍寄存器值。

### 28. DMA Busy时再次发操作会怎样？怎样验证？

**面试回答：** 正确行为取决于规格：可等待、拒绝、排队，或允许某些状态寄存器访问，但不应静默丢命令或破坏当前搬运。当前RTL上游`PREADY`通常受`is_dma_busy`抑制，表现为等待，但不同读写/命令路径仍需分别验证。测试应在INIT/COPY Busy时发Direct、寄存器访问、新INIT/COPY，检查握手时刻、旧任务数据是否被覆盖、新命令是否只执行一次及IRQ状态。

**当前实现范围：** 有Busy定向Sequence在INIT期间发多类CPU访问，并用`COV_PROBE.dma_busy`确认Setup发生在忙期；Busy期间再次写INIT/COPY被列为风险/waiver，不能说已有完整重入命令验证。

### 29. LEN归零后不重配再COPY，为何可能下溢？

**面试回答：** 合法COPY的最后一笔目的写完成后，5-bit `dma_len`从1减到0。如果软件不重写LEN就再次写COPY命令，当前启动条件没有检查`LEN>0`，会进入COPY读写流程；第一笔目的写后`0-1`在5-bit寄存器中回绕为31，可能继续产生大量非预期事务。定位时看第二次命令前的LEN读回、COPY active状态、每次下游写握手时LEN变化和下游地址序列；加超时及Unexpected Transfer检查防止仿真挂住。

**边界：** 这属于从RTL可推导的高风险场景；若没有实际复现日志，不要给出“实测搬了多少笔”或声称已修复。

### 30. 显式写LEN=0与操作后遗留LEN=0一样吗？

**面试回答：** 对后续命令而言两者寄存器数值都是0，但**成因和错误报告路径不同**。显式写`LEN=0`会命中当前RTL的非法长度检测并置`invalid_length`；正常搬运结束自然减到0，不会因此自动置该位。若此时再次发COPY，当前RTL也没有在COPY命令处重新做LEN合法性检查，所以存在第29题的下溢风险。验证必须分成“非法LEN写入”和“完成后未重配LEN就重发命令”两个独立场景。

### 31. 0减1回绕应如何防护和验证？

**建议设计：** 启动INIT/COPY前检查`LEN`有效且非0；只在有效下游写完成且剩余LEN>0时递减；非法命令按规格返回错误状态或IRQ且不得产生下游访问。最好另设“LEN已配置/任务已消费”状态，区分软件显式写0与上次任务结束遗留0，避免只看数值导致语义混淆。

**建议验证：** 两条独立测试链：① Reset→配置SRC/DST→写LEN=0→发COPY；② 配置合法LEN→完成COPY使LEN归零→不重配直接再发COPY。分别检查错误位/IRQ、下游请求数为0、地址和Memory不变、不会超时。当前Scoreboard对`model_len==0`命令会报不支持，但没有完整的“发非法命令后预测拒绝+有限时间无下游访问”专项用例；这是明确的后续增强点。

## 面试结束前的三条边界提醒

- **区分事实与建议**：RTL当前行为、现有验证检查、你建议的规格修正是三层，不能混说。
- **区分项目与个人贡献**：仓库不能证明作者身份；请把第2、3题中的“我”替换为真实职责。
- **区分覆盖目标与结果**：有Covergroup、Test Plan和回归脚本，不等于已经获得可引用的覆盖率百分比或正式Sign-off。

## 代码索引

- DUT行为：`apb_device.v`
- 环境连接：`apb_device_src/apb_device_env.sv`
- 端到端模型：`apb_device_src/apb_device_scoreboard.sv`
- 覆盖：`apb_device_src/apb_device_coverage_full.sv`
- 中断路径：`src/apb_master_monitor.sv`、`src/apb_master_sequencer.sv`、`src/apb_interrupt_handler.sv`、`src/apb_interrupt_sequence.sv`
- Reactive Slave：`src/apb_slave_monitor.sv`、`src/apb_slave_sequencer.sv`、`src/apb_slave_basic_sequence.sv`、`src/apb_slave_driver.sv`
- 激励和Test：`sequences/dma_apb_coverage_sequences.sv`、`tests/dma_apb_tests.sv`
- 计划和waiver：`DMA_APB_Verification_Testplan.md`、`README.md`
