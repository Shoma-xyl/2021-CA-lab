### 体系结构LAB4实验报告

JL19110004  徐语林

#### 一.实验目标

1.实现BTB和BHT两种动态分支预测器

2.体会动态分支预测对流水线性能的影响

#### 二.实验环境和工具

语言：verilog/systemverilog

环境：vivado/windows操作系统

#### 三.实验内容,过程及结果

##### BTB部分：

![image-20210530201122001](C:\Users\10062\AppData\Roaming\Typora\typora-user-images\image-20210530201122001.png)

结构如上图所示，相当于LAB3给出的直接映射的cache。

对于上图的结构的代码，如下：

`reg [            31:0] predictpc    [SET_SIZE]; // SET_SIZE个line，每个line有LINE_SIZE个word
reg [TAG_LEN-1:0]      pctag        [SET_SIZE];            // SET_SIZE个TAG
reg                    state        [SET_SIZE];            // SET_SIZE个valid(有效位)`

BTB是先进行一个TAG比对，并且在预测位为1的时候才更新使能信号；如果写信号为1，则进行写入，关键代码如下：

`always @ (*) begin              // 进行一个比对,比对完后根据实际的情况对使能信号以及预测pc进行一个赋值
    if(pctag[rd_btb_addr] == rd_tag_addr && state[rd_btb_addr])   // 先是进行一个tag的比对，然后判断预测位是否为1
    begin
        en=1'b1;
    end
    else
    begin
        en=1'b0;
    end
     predict_pc=predictpc[rd_btb_addr];
end
//写入buffer
always@(posedge clk or posedge rst)
begin
    if(rst)
    begin
        for(integer i=0;i<SET_SIZE;i=i+1)
        begin
            predictpc[i]<=0;
            pctag[i]<=0;
            state[i]<=0;
        end
        en<=0;
        predict_pc<=0;
    end
    else
    begin
        if(wr_req)
        begin
            predictpc[wr_btb_addr]<=wr_predict_pc;
            pctag[wr_btb_addr]<=wr_tag_addr;
            state[wr_btb_addr]<=predict_bit;
        end
    end
end`

完成了BTB部分后，将其加入CPU。加入CPU后主要的运行过程是：把BTB加入到IF/ID段，从IF段输出的PCF进入BTB,如果命中，则输出的地址进入NPC，选择预测的PC作为下一个PC；如果不命中则使用PCF+4，因为不知道此时的结果是否正确，于是预测的使能位(表示是否进行了预测)以及预测的PC要随着流水线一起往下传，到了EX段后，由于BranchE信号在此段生成，于是可以判断之前的预测是否正确，如果实际跳转但是之前并没有预测，或者实际不跳转但是之前进行了预测，都要清空ID和EX段寄存器，且NPC模块会根据情况得到下一个PC。对于BTB的更新也需要在EX段确定了是否跳转后进行修改。

大致做了修改的部分有RV32core,HazardUnit,IDSegReg,EXSegReg,NPC_Generator.

具体的代码见附件。



##### BHT部分：

对于BHT部分，分支PC和预测的PC仍然保存在了BTB中，只不过此时BTB的作用只是进行一个确认当前指令是否是分支指令。BHT部分则是预测是否跳转，维护一个表，同样采取的是直接映射，用的是2bit的预测位来预测是否发生跳转，参考下图：

![image-20210530205255479](C:\Users\10062\AppData\Roaming\Typora\typora-user-images\image-20210530205255479.png)

对于BHT部分的代码则是用上图来显示状态的转移，如下所示：

`always@(posedge clk or posedge rst) 
begin//写入buffer,有一个状态机的转移
	if(rst) 
	begin
		for(integer i = 0; i < SET_SIZE; i = i + 1) begin
			state[i] = 2'b00;
		end
		rd_predicted_taken = 1'b0;
	end 
	else 
	begin
		if(wr_req) 
		begin
			    case(state[wr_table_addr])
			    2'b00:
			    begin
			        if(wr_taken)
			            state[wr_table_addr]<=2'b10;
			    end
			    2'b01:
			    begin
			        if(wr_taken) 
			        begin
                        state[wr_table_addr] <= 2'b11;
                    end 
                    else 
                    begin
                        state[wr_table_addr] <= 2'b00;
                    end
			    end
			    2'b10:
			    begin
			        if(wr_taken) 
			        begin
                        state[wr_table_addr] <= 2'b11;
                    end 
                    else 
                    begin
                        state[wr_table_addr] <= 2'b00;
                    end
			    end
			    2'b11:
			    begin
			        if(!wr_taken)
			            state[wr_table_addr]<=2'b10;
			    end
			    endcase 
		end
	end
end`

加入CPU后，由于是在BTB部分的基础上进行添加的BHT,因此对于之前的BTB部分要进行一个修改，主要是加入BHT后运用的是BHT中2bit的预测位来进行判断，而不再利用BTB中的1bit的预测位。处理的大致思路和BTB类似，主要的不同在于在EX段得到了真实的跳转结果后，要根据BTB Hit ，BHT hit以及真实的结果来进行处理NPC以及Harzard部分。

修改的模块和BTB一样。



##### 实验结果：

在对BTB和BHT的cpu进行仿真后，快速排序和矩阵相乘的testbench已经成功，由于已经检查过，在此略过了这一步的展示，下面仅显示各CPU的分支指令预测的次数，和预测错误次数的截图。

###### 无分支预测CPU：

Quicksort:

<img src="C:\Users\10062\AppData\Roaming\Typora\typora-user-images\image-20210531132004844.png" alt="image-20210531132004844" style="zoom:50%;" />

Matrix:

<img src="C:\Users\10062\AppData\Roaming\Typora\typora-user-images\image-20210531131621901.png" alt="image-20210531131621901" style="zoom:50%;" />

btb:

<img src="C:\Users\10062\AppData\Roaming\Typora\typora-user-images\image-20210531132318732.png" alt="image-20210531132318732" style="zoom:50%;" />

bht:

<img src="C:\Users\10062\AppData\Roaming\Typora\typora-user-images\image-20210531132550875.png" alt="image-20210531132550875" style="zoom:50%;" />

###### BTB：

Quicksort:

<img src="C:\Users\10062\AppData\Roaming\Typora\typora-user-images\image-20210531125110386.png" alt="image-20210531125110386" style="zoom:50%;" />

Matrix:

<img src="C:\Users\10062\AppData\Roaming\Typora\typora-user-images\image-20210531125432875.png" alt="image-20210531125432875" style="zoom:50%;" />

btb:

<img src="C:\Users\10062\AppData\Roaming\Typora\typora-user-images\image-20210531125804024.png" alt="image-20210531125804024" style="zoom:50%;" />

bht:

<img src="C:\Users\10062\AppData\Roaming\Typora\typora-user-images\image-20210531130124674.png" alt="image-20210531130124674" style="zoom:50%;" />

###### BHT：

Quicksort:

<img src="C:\Users\10062\AppData\Roaming\Typora\typora-user-images\image-20210531122301393.png" alt="image-20210531122301393" style="zoom:50%;" />

Matrix:

<img src="C:\Users\10062\AppData\Roaming\Typora\typora-user-images\image-20210531123306315.png" alt="image-20210531123306315" style="zoom:50%;" />

btb:

<img src="C:\Users\10062\AppData\Roaming\Typora\typora-user-images\image-20210531123719412.png" alt="image-20210531123719412" style="zoom:50%;" />

bht:

<img src="C:\Users\10062\AppData\Roaming\Typora\typora-user-images\image-20210531124005279.png" alt="image-20210531124005279" style="zoom:50%;" />

#### 四.实验分析

##### 分析分支收益和分支代价：

###### 无分支预测的cpu:

假定分支指令是不跳转的，NPC选择的是PC+4；但是当EX段发现要跳转，那么就要Flush流水线，这里要两个周期的代价；当CPU不跳转时，CPI为1；要跳转是相当于CPI为3.

###### BTB:

预测错误的代价是两个周期，因此相当于预测错误的时候的CPI为3，预测成功的时候的CPI为1

###### BHT:

与BTB相似，但是从实验结果看，BHT各方面的预测水平都要高于BTB;BHT预测错误的时候的CPI为3，预测成功的时候的CPI为1.



##### 统计未使用分支预测和使用分支预测的总周期数及差值

###### quicksort:

|                 | 总时间(ns) | 总周期 | 差值   |
| --------------- | ---------- | ------ | ------ |
| 无分支预测的cpu | 261120     | 65280  |        |
| BTB             | 263080     | 65770  | +490   |
| BHT             | 204584     | 51146  | -14134 |

###### matrix:

|                 | 总时间(ns) | 总周期 | 差值  |
| --------------- | ---------- | ------ | ----- |
| 无分支预测的cpu | 1418404    | 354601 |       |
| BTB             | 1387980    | 346995 | -7606 |
| BHT             | 1382300    | 345575 | -9026 |



###### btb:

|                 | 总时间(ns) | 总周期 | 差值 |
| --------------- | ---------- | ------ | ---- |
| 无分支预测的CPU | 2044       | 511    |      |
| BTB             | 1260       | 315    | -196 |
| BHT             | 1260       | 315    | -196 |



###### bht:

|                 | 总时间(ns) | 总周期 | 差值 |
| --------------- | ---------- | ------ | ---- |
| 无分支预测的CPU | 2148       | 537    |      |
| BTB             | 1532       | 383    | -154 |
| BHT             | 1460       | 365    | -172 |



##### 统计分支指令数目、动态分支预测正确次数和错误次数

###### quicksort:

|                 | 分支指令数目 | 动态分支预测正确次数 | 错误次数 |
| --------------- | ------------ | -------------------- | -------- |
| 无分支预测的cpu | 7397         | 5782                 | 1615     |
| BTB             | 7397         | 5270                 | 2127     |
| BHT             | 7397         | 6310                 | 1087     |



###### matrix:

|                 | 分支指令数目 | 动态分支预测正确次数 | 错误次数 |
| --------------- | ------------ | -------------------- | -------- |
| 无分支预测的cpu | 4624         | 274                  | 4350     |
| BTB             | 4624         | 4076                 | 548      |
| BHT             | 4624         | 4346                 | 278      |



###### btb:

|                 | 分支指令数目 | 动态分支预测正确次数 | 错误次数 |
| --------------- | ------------ | -------------------- | -------- |
| 无分支预测的cpu | 101          | 1                    | 100      |
| BTB             | 101          | 99                   | 2        |
| BHT             | 101          | 99                   | 2        |



###### bht:

|                 | 分支指令数目 | 动态分支预测正确次数 | 错误次数 |
| --------------- | ------------ | -------------------- | -------- |
| 无分支预测的cpu | 110          | 11                   | 99       |
| BTB             | 110          | 88                   | 22       |
| BHT             | 110          | 97                   | 13       |



##### 对比不同策略并分析以上几点的关系

1.没有预测分支的cpu在大多数时候的性能都是要差于BTB和BHT的，但是由于BHT是两位的预测位，因此具有更佳的弹性，因此性能最优

2.对于快速排序这种分支是随机的样例而言，由于BTB是根据上一次跳转的结果来决定本次跳转与否，因此会出现数据没有无分支预测的CPU好的现象

3.对于for循环，由于for循环是前n次都处于循环状态，而第n次跳出，对于不加分支预测的cpu来说是比较困难的，因此可以看见此时的性能是大大的不如BTB和BHT。

4.推荐使用BHT的分支预测来进行条件转移指令的决策。

#### 五.总结及建议

通过本次动态分支预测的实验，体会到了BTB和BHT这两种动态分支预测器的本质和不同点，以及加入BTB和BHT后对于原CPU的一个性能上的改进。

建议：无