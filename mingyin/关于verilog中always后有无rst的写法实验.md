---
title: 关于verilog中always后有无rst的写法实验
description: 此文是关于网上非常常见的always @(posedge clk or posedge rst)这种写法的实验研究
published: true
date: 2025-04-12T16:03:35.392Z
tags: 
editor: markdown
dateCreated: 2025-04-12T16:00:53.539Z
---

> <img src="https://s2.loli.net/2024/09/03/zxBRC9dnIPTswZh.jpg" alt="洺音" style="position: absolute; top: 0; right: 0px; width: 207px; height: 207px;">
> <br>
> 
> ### 作者 洺音 
> 
> <i class="fab fa-qq fa-sm"></i> **QQ** 469480405 &emsp; &emsp; &emsp; <i class="fab fa-weixin fa-sm"></i> **微信** \_ming_yin_
> <br>
> - 懂一点数字IC，会画一点PCB。
> - 喜欢搞oc，画画。

## 引言

&emsp; &emsp;我在学习FPGA的过程中，在网上查阅过大量代码，也使用AI生成过很多代码。我发现几乎所有的代码包括AI生成的代码都会出现这样一段：
```verilog
    always @(posedge clk or posedge rst) begin
        if (rst)  xxxxx;
        else      xxxxx;
    end
```
&emsp; &emsp;而我的习惯一般是这样的：
```verilog
    always @(posedge clk) begin
        if (rst)  xxxxx;
        else      xxxxx;
    end
```
&emsp; &emsp;那么就产生了一个问题，在一个时序系统中，rst信号既然已经作为了判断条件，为什么还要专门把rst信号作为触发信号之一写上去呢，这样到底会有什么影响，有什么必要性吗？
&emsp; &emsp;因此我专门针对这个问题研究了一下VIVADO到底是怎么去综合触发器的，在此基础上做一些改动，又会有什么影响。

## 同步和异步

&emsp; &emsp;首先我们就用最基本代码来试试这两种结构有什么区别：
```verilog
    always @(posedge clk_ibuf or posedge rst) begin
        if (rst) p <= 0;  // 异步复位
        else     p <= c;      // 正常数据输入
    end
    
    always @(posedge clk_ibuf) begin
        if (rst) q <= 0;  // 同步复位
        else     q <= d;      // 正常数据输入
    end
```
&emsp; &emsp;综合出来的结果长这样

<br>
<div style="text-align: center;">
<img src="https://s2.loli.net/2025/04/12/Ei3kNr2WRlYQXt5.png" alt="fb0bc72f1a588f15fb65ba2528d4061b" style="zoom:80%;"/>
</div>

&emsp; &emsp;\*补充一下几种基本结构的名字：
&emsp; &emsp;异步复位（FDCE）
&emsp; &emsp;异步置位（FDPE）
&emsp; &emsp;同步复位（FDRE）
&emsp; &emsp;同步置位（FDSE）
&emsp; &emsp;可以看到上下两种情况分别被综合成了异步复位和同步复位。
&emsp; &emsp;同理将代码从复位改成置位：
```verilog
    always @(posedge clk_ibuf or posedge rst) begin
        if (rst) p <= 1;  // 异步复位
        else     p <= c;      // 正常数据输入
    end
    always @(posedge clk_ibuf) begin
        if (rst) q <= 1;  // 同步复位
        else     q <= d;      // 正常数据输入
    end
```
&emsp; &emsp;即可得到另外两种置位结构

<br>
<div style="text-align: center;">
<img src="https://s2.loli.net/2025/04/12/CzEkrXp4VWR829s.png" alt="fb0bc72f1a588f15fb65ba2528d4061b" style="zoom:80%;"/>
</div>

&emsp; &emsp;观察引脚可知，在同步结构中rst分别引入作为S或者R引脚。异步结构中则被命名为PRE（预写入）和CLR（清除）。从功能性上看起来区别不大，但是从结构上应该是有区别的。
&emsp; &emsp;所以在继续测试之前，要先猜测一下综合的电路结构。

## 电路结构

&emsp; &emsp;首先我们都知道，最基础的触发器是两个与非门组成的RS触发器，像是这样

<br>
<div style="text-align: center;">
<img src="https://s2.loli.net/2025/04/12/SsvFGclCe4U3V9y.png" alt="fb0bc72f1a588f15fb65ba2528d4061b" style="zoom:60%;"/>
</div>

&emsp; &emsp;通过R和S可以完成置1置0和不变三种操作，但是显而易见，这样的结构没有时钟参与，不是一个很好的时序结构。
&emsp; &emsp;接下来我们加入时钟，那么现在这就是一个时钟控制的RS触发器，但是这两个信号的控制方式不太直观，也很不方便。

<br>
<div style="text-align: center;">
<img src="https://s2.loli.net/2025/04/12/aoTjQqnvSCpsy2k.png" alt="fb0bc72f1a588f15fb65ba2528d4061b" style="zoom:60%;"/>
</div>

&emsp; &emsp;作为存储器，我们显然希望在每个时钟的时候直接将里面的值变更为新的值，因此我们对其进行一点基本的改动得到D触发器，到这一步就是FPGA中每一个触发器的基本结构了

<br>
<div style="text-align: center;">
<img src="https://s2.loli.net/2025/04/12/BFwN7SKtbrxv8uY.png" alt="fb0bc72f1a588f15fb65ba2528d4061b" style="zoom:60%;"/>
</div>


&emsp; &emsp;常规的D触发器只能在时钟上升沿把值改成输入的值，但是我们还需要一个根据复位信号置零或者置一的功能，可以通过加上或门或者或非门来加上rst信号，这样我们就做出了同步复位/置位触发器，对应always @(posedge clk)

<br>
<div style="text-align: center;">
<img src="https://s2.loli.net/2025/04/12/h8dWBilEr7ApxNf.png" alt="fb0bc72f1a588f15fb65ba2528d4061b" style="zoom:60%;"/>
</div>

&emsp; &emsp;显然我们也可以在输出的地方直接加上rst判断，这样就形成了与clk完全无关的rst控制，即异步复位/置位触发器，对应always @(posedge clk_ibuf or posedge rst) 这种情况

<br>
<div style="text-align: center;">
<img src="https://s2.loli.net/2025/04/12/Nvrqw6xzcRpP9tn.png" alt="fb0bc72f1a588f15fb65ba2528d4061b" style="zoom:60%;"/>
</div>

## 基本区别

&emsp; &emsp;现在我们继续实验，测试如下代码：
```verilog
always @(posedge rst or posedge clk_ibuf) begin
    if (rst) p <= 0;  
    else     p <= c;      
end

always @(posedge rst or posedge clk_ibuf) begin
    if (clk_ibuf) q <= 0; 
    else     q <= d;     
end
```
&emsp; &emsp;可以看到综合结果里，clk与rst调换了位置，证明两个触发信号的功能性是不同的。

<br>
<div style="text-align: center;">
<img src="https://s2.loli.net/2025/04/12/JlYvcjbkaCzPrMS.png" alt="fb0bc72f1a588f15fb65ba2528d4061b" style="zoom:70%;"/>
</div>

&emsp; &emsp;直到这里，我们清楚了VIVADO对这两种写法的综合逻辑，但是上述的例子资源占用量都相同，对实际代码编写影响不大，那么接下来来测试一些有影响的情况。
试试这段代码，不用rst信号复位而引入第三个en信号作复位信号。
```verilog
always @(posedge clk_ibuf or posedge rst) begin
    if (en) p <= 0; 
    else     p <= c;   
end
```
&emsp; &emsp;综合结果直接报错，说代码有时序问题。

<br>
<div style="text-align: center;">
<img src="https://s2.loli.net/2025/04/12/9uoBzFEOAjm4efi.png" alt="fb0bc72f1a588f15fb65ba2528d4061b" style="zoom:200%;"/>
</div>

&emsp; &emsp;事实上，在有两个时序信号的模块中如果不将其中一个时序作为特殊情况列出来就会报错。这是因为当两个上升沿相邻很近的时候，硬件可能会先执行A上升沿操作再执行B上升沿，也可能先B再A，也可能只执行一次，而数字设计中不允许出现这种无法确定结果的情况，所以会报错。
&emsp; &emsp;而我们如果只用一个时序信号就不会出现这种问题，像这样写就不会有问题
```verilog
always @(posedge clk_ibuf) begin
    if (en) q <= 1; 
    else     q <= d; 
end
```
&emsp; &emsp;可以看到en取代了此前rst的位置作为新的置位信号

<br>
<div style="text-align: center;">
<img src="https://s2.loli.net/2025/04/12/68vBN7eCwmDoWTG.png" alt="fb0bc72f1a588f15fb65ba2528d4061b" style="zoom:80%;"/>
</div>

&emsp; &emsp;继续测试让rst信号和en信号都作为置位信号的情况
```verilog
always @(posedge clk_ibuf or posedge rst) begin
    if (rst) p <= 1;
    else if (en) p <= 1; 
    else     p <= c;   
end

always @(posedge clk_ibuf) begin
    if (rst) q <= 1;
    else if (en) q <= 1; 
    else     q <= d; 
end
```
&emsp; &emsp;可以看到，在异步系统中，可能是由于rst信号被写在时序信号上的缘故，VIVADO优先保证了rst信号在PRE位置上，将en与输入信号c通过LUT2结合起来作为D输入。
&emsp; &emsp;而在同步系统中，VIVADO把en和rst两个置位信号通过LUT2结合后作为置位信号输入，保持了输入信号d不变。

<br>
<div style="text-align: center;">
<img src="https://s2.loli.net/2025/04/12/K2cx1abGgenhPA8.png" alt="fb0bc72f1a588f15fb65ba2528d4061b" style="zoom:80%;"/>
</div>

&emsp; &emsp;当把en信号也写到时序信号里的时候异步系统才会把en和rst绑定起来
```verilog
always @(posedge clk_ibuf or posedge rst or posedge en) begin
    if (rst) p <= 1;
    else if (en) p <= 1; 
    else     p <= c;   
end
```
<br>
<div style="text-align: center;">
<img src="https://s2.loli.net/2025/04/12/S8khzRs3F6gqwXt.png" alt="fb0bc72f1a588f15fb65ba2528d4061b" style="zoom:80%;"/>
</div>

## 进阶区别

&emsp; &emsp;上面的测试都区别不大，但是当rst信号不复位也不置位，而是作为赋值的时候区别就很大了，比如这一段代码

```verilog
    always @(posedge clk_ibuf or posedge rst) begin
        if (rst) p <= en;
        else     p <= c;   
    end
    
    always @(posedge clk_ibuf) begin
        if (rst) q <= en;
        else     q <= d; 
    end
```
&emsp; &emsp;两种写法的资源占用区别大的惊人

<br>
<div style="text-align: center;">
<img src="https://s2.loli.net/2025/04/12/tTcbPhZqOX8vsVF.png" alt="fb0bc72f1a588f15fb65ba2528d4061b" style="zoom:80%;"/>
</div>

&emsp; &emsp;由于异步的触发器需要将rst从输出端接入，这段代码中根据en的值rst信号既要能够置一又要能够置零，显然没有办法在触发器内部生成LUT电路，所以只能生成两个异步触发器，一个是置零一个是置一，然后分别用两个LUT2生成置零和置一信号输送给CLR和PRE，再生成一个Latch接收两个LUT2的结果来判断采用哪个输出值，最后将三个信号通过LUT3选择后输出，过程可谓是相当的复杂。
&emsp; &emsp;而同步的触发器就没这么多事了，因为是从输入端接入，所以直接提前控制输入的值即可，用rst对en和d进行选择，经过一个LUT3之后直接输入同步触发器即可实现功能。

&emsp; &emsp;异步结构需要两个FF，一个Latch，两个LUT2，一个LUT3实现
&emsp; &emsp;同步结构需要一个FF，一个LUT3实现

## 结论

&emsp; &emsp;综上所述可以得出结论，异步写法中，编译器会优先保证复位/置位信号从输出端附近输入，因此必须提前确定是复位还是置位信号，如果是复杂的条件赋值，那么只能复位和置位都生成一个，然后生成更多结构去选择。而在同步的写法中，编译器会在输入端提供多一个接口，就算有很复杂的结构也可以通过前置的LUT来实现，能够节省很多资源。

&emsp; &emsp;因此在设计的过程中，使用同步的写法设计起来更加从容随意，如果使用异步的写法设计的时候就要严格注意把每个信号在rst条件中变成确定数值。
