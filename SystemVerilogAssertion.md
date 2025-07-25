# SystemVerilogAssertion学习笔记


## 两种断言

### 立即断言

立即被求值，类似于if

```systemverilog
always@(posedge clk) 
begin
    a_ia: assert (a && b);
end
```

### 同时断言

```systemverilog
a_cc:assert property (@(posedge clk) not (a&&b));
```

断言在clk的每个正沿上执行，并使用预置区域中的变量的值来计算表达式，该区域是时钟给定边沿之前的增量周期。因此，在时钟边缘采样，Take for Assertion的值将为时钟边沿之前的稳定值。

## 构造SVA块

时序
```systemverilog
sequence s_ab;
    a ##1 b;
endsequence
```

属性
```systemverilog
property p_expr;
    @(posedge clk) s_ab;
endproperty
```

断言
s_a:assert property(p_expr);

以上三部分构成一个完整的SVA块，但是实际使用可能不需要全部。

```systemverilog
property p_req_gnt;
    @(posedge clk) $rose (req) |->
                ##[2:5] $rose (gnt);
endproperty
```
```systemverilog
a_reg_gnt:
    assert property(p_req_gnt)
    else if(arb_sva_severity) $fatal;
```
```systemverilog
c_reg_gnt:cover property(p_req_gnt);
```

## 边沿定义
```
$rose 0->1  
$fell 1->0  
$stable 未跳变
```  

## 逻辑关系

在语句中使用逻辑关系运算符
```systemverilog
sequence s3;
    @(posedge clk) a || b;
endsequence
```

## 序列形参

```systemverilog
sequence s3_lib (a,b);
    @(posedge clk) a || b;
endsequence
```
```systemverilog
sequence s3_lib_inst1;
    @(posedge clk) s3_lib (req1, req2);
endsequence
```
实际的形参的用法在总的SVA构造module体现。

## 时序关系

SVA在sequence,property,assert其一需要有“时序检查”
通常情况下，在属性（property）的定义中指定时钟，并保持序列（sequence）独立于时钟是一种好的编码风格。这可以提高基本序列定义的可重用性。

## 禁止属性

not 期望某个属性为假，为假时断言成功。

## 蕴含（Implication）
先行算子（antecedent） 类似于if
后续算子（consequent） 类似于then

a 成功 b 成功  结果 成功  
a 成功 b 失败  结果 失败  
a 失败 b 都行  结果 空成功

### 交叠蕴含 “-|>”

```systemverilog
property p8;
    @(posedge clk) a |-> b;
endproperty
```
信号a为高则信号b在相同的时钟边沿也必须为高

### 非交叠蕴含 "|=>"

```systemverilog
property p8;
    @(posedge clk) a |=> b;
endproperty
```
信号a为高则信号b在下一个时钟边沿也必须为高

### 后续算子带固定延迟

```systemverilog
property p10;
    @(posedge clk) a |-> ##2 b;
endproperty
```

### 序列蕴含

```systemverilog
property p11;
    s11a |-> s11b;
endproperty
```

## 时序窗口

### 有限

```systemverilog
property p12;
    @(posedge clk) (a && b) |-> ##[1:3] c;
endproperty

a12: assert property(p12);
```
时序窗口可重叠##[0:3]

### 无限
```
##[1:$]
```
影响性能一般不使用

## ended结构
通常情况下是将多个序列以序列的起始点作为同步点，来组合成时间上连续的检查。ended使用序列的结束点作为同步点。
```systemverilog
sequence s15a;

endsequence

sequence s15b;

endsequence

property p15;
    s15a.ended |-> ##2 s15b.ended;
endproperty

a15: assert property(p15);
```

## 使用参数的SVA检验器

```systemverilog
module generic_chk (input logic a, b, clk);
parameter delay = 1;

property p16;
    @(posedge clk) a |-> ##delay b;
endproperty

a16: assert property(p16);

endmodule
```
后续会有进阶版本

## 使用选择运算符的SVA检验器

```systemverilog
property p17;
    @(posefge clk) c ? d == a: d == b;
endproperty

a17: assert property(p17);
```

## 使用true表达式的SVA检验器

```systemverilog
sequence s18;
    @(posedge clk) a ##1 b ##1 'true;
endsequence
```
延迟一个周期判定

## $past构造

```systemverilog
property p19; 
@(posedge clk) (c && d) |->  
            ($past((a&&b)，2) == 1'b1); 
endproperty

a19：assert property(p19); 
```
带时钟门控的$past构造
```systemverilog
roperty p20; 
@(posedge clk) (c && d) |->  
            ($past((a&&b)，2，e) == 1'b1); 
endproperty 

a20：assert property(p20); 
```
信号e在任意给定时钟上升沿有效时检验才被激活

## 重复运算符

### 连续重复运算符[*]

#### 普通[*]
```systemverilog
property p21; 
@(posedge clk) $rose(start) |->  
                ##2 (a[*3]) ##2 stop ##1 !stop; 
endproperty 
a21： assert property(p21); 
```
#### 用于序列的连续重复运算符[*]
```systemverilog
 property p22; 
@(posedge clk) $rose(start) |->  
                ##2 ((a ##2 b)[*3]) ##2 stop; 
endproperty 
a22： assert property(p22); 
```
#### 用于带延迟窗口的序列的连续重复运算符[*]
```systemverilog
property p23; 
@(posedge clk) $rose(start) |->  
                ##2 ((a ##[1:4] b)[*3]) ##2 stop; 
endproperty 
a23： assert property(p23); 
```
#### 连续运算符[*]和可能性运算符
```systemverilog
property p24; 
@(posedge clk) $rose(start) |->  
                ##2 (a[*1：$]) ##1 stop; 
endproperty 
a24： assert property(p24); 
```

### 跟随重复运算符[->]
```systemverilog
property p25; 
@(posedge clk) $rose(start) |->
                ##2 (a[->3]) ##1 stop; 
endproperty 
a25： assert property(p25); 
```
最后一次信号a ##1（延迟一个周期）stop为高

### 非连续重复运算符[=]
```systemverilog
Property p26; 
@(posedge clk) $rose(start) |->  
                ##2 (a[=3]) ##1 stop ##1 !stop; 
endproperty 
a26： assert property(p26); 
```
信号a第3次匹配成功后不需要信号stop为高不必在下一个时钟发生，往后有发生即可。

## and构造
```systemverilog
sequence s27a; 
    @(posedge clk) a##[1:2] b; 
endsequence 
sequence s27b; 
    @(posedge clk) c##[2:3] d; 
endsequence 
 
property p27; 
    @(posedge clk)  s27a and s27b; 
endproperty 
 
a27: assert property(p27); 
```
当两个序列都成功时整个属性才成功，两个序列必须有相同的起始点，但是可以有不同的结束点。

## intersect构造
```systemverilog
sequence s28a; 
    @(posedge clk) a##[1：2] b; 
endsequence 
sequence s28b; 
    @(posedge clk) c##[2：3] d; 
endsequence 
property p28; 
    @(posedge clk)  s28a intersect s28b; 
endproperty 
a28： assert property(p28); 
```
两个序列必须在相同时刻开始且结束于同一时刻。换句话说，两个序列的长度必须相等。

## or构造
```systemverilog
 sequence s29a; 
    @(posedge clk) a##[1:2] b; 
endsequence 
sequence s29b; 
    @(posedge clk) c##[2:3] d; 
endsequence 
property p29; 
    @(posedge clk)  s28a or s28b; 
endproperty 
a29： assert property(p29); 
```
只要其中一个序列成功，整个属性就成功。

## first_match构造
```systemverilog
sequence s30a; 
    @(posedge clk) a ##[1:3] b; 
endsequence 
sequence s30b; 
    @(posedge clk) c ##[2:3] d; 
endsequence 
property p30; 
    @(posedge clk) first_match(s30a or s30b); 
endproperty 
a30： assert property(p30); 
```
任何时候使用了逻辑运算符(如“and”和“or”)的序列中指定了时间窗，就有可能出现同一个检验具有多个匹配的情况。“first_match”构造可以确保只用第一次序列匹配，而丢弃其他的匹配。当多个序列被组合在一起，其中只需时间窗内的第一次匹配来检验属性剩余的部分时，“first_match”构造非常有用。

## throughout构造
```systemverilog
property p31; 
    @(posedge clk) $fell(start) |->  
        (!start) throughout  
        (##1 (!a&&!b) ##1 (c[->3]) ##1 (a&&b)); 
endproperty 
a31： assert property(p31); 
```
为了保证某些条件在整个序列的验证过程中一直为真，可以使用“throughout”运算符。

## 内建系统函数

$onehot(expression)—— 检验表达式满足“one-hot”，换句话说，就是在任意给定的时钟沿，表达式只有一位为高。  
$onehot0(expression)—— 检验表达式满足“zero one-hot”，换句话说，就是在任意给定的时钟沿，表达式只有一位为高或者没有任何位为高。  
$isunknown(expression)—— 检验表达式的任何位是否是 X或者Z。  
$countones(expression)—— 计算向量中为高的位的数量。  

## disable iff构造
```systemverilog
property p34; 
    @(posedge clk)  
    disable iff (reset)  
    $rose(start) |=> a[=2] ##1 b[=2] ##1 !start ; 
endproperty 
a34： assert property(p34);
```
若信号reset为高则不执行后续检验。

## intersect构造
```systemverilog
property p35; 
    (@(posedge clk)  1[*2:5] intersect  
                        (a ##[1:$] b ##[1:$] c)); 
endproperty 
a35： assert property(p35); 
```
这个intersect 的定义检查从序列的有效开始点(信号“a”为高)，到序列成功的结束点(信号“c”为高)，一共经过2~5个时钟周期。 

## 属性中使用形参
```systemverilog
property arb (a，b，c，d); 
    @(posedge clk) ($fell(a) ##[2:5] $fell(b)) |->
                    ##1 ($fell(c) && $fell(d)) ##0  
                    (!c&&!d) [*4] ##1 (c&&d) ##1 b; 
endproperty 
a36_1： assert property(arb(a1，b1，c1，d1)); 
a36_2： assert property(arb(a2，b2，c2，d2)); 
a36_3： assert property(arb(a3，b3，c3，d3)); 
```

## 嵌套的蕴含
```systemverilog
`define free (a && b && c && d) 
property p_nest; 
    @(posedge clk) $fell(a) |->  
                    ##1 (!b && !c && !d) |->  
                                    ##[6：10] `free;
endproperty 
a_nest： assert property(p_nest); 
```
## 在蕴含中使用if/else
```systemverilog
property p_if_else; 
    @(posedge clk)  
    ($fell(start) ##1 (a||b)) |-> 
        if(a)  
            (c[->2] ##1 e) 
        else  
            (d[->2] ##1 f); 
endproperty 
a_if_else： assert property(p_if_else); 
```
## SVA中的多时钟定义
```systemverilog
sequence s_multiple_clocks; 
    @(posedge clk1) a ##1 @(posedge clk2) b; 
endsequence 
```
当在一个序列中使用了多个时钟信号时，只允许使用“##1”延迟构造。使用“##0”会产生混淆，即在信号“a”匹配后究竟哪个时钟信号才是最近的时钟。这将引起竞争，因此不允许使用。使用##2 也不允许，因为不可能同步到时钟“clk2”的最近的上升沿。
禁止在两个不同时钟驱动的序列之间使用交叠蕴含运算符。因为先行算子的结束和后续算子的开始重叠，可能引起竞争的情况，这是非法的。下面的代码显示了这种非法的编码方式：
```systemverilog
property p_multiple_clocks_implied_illegal; 
    @(posedge clk1) s1 |-> @(posedge clk2) s2; 
endproperty 
```
## matched构造
```systemverilog
sequence s_a; 
    @(posedge clk1) $rose(a); 
endsequence 
sequence s_b; 
    @(posedge clk2) $rose(b); 
endsequence 
property p_match; 
    @(posedge clk2) s_a.matched |=> s_b; 
endproperty 
a_match： assert property(p_match); 
```
匹配最近的clk2
## expect构造
```systemverilog
initial 
begin 
@(posedge clk); 
#2ns cpu_ready = 1'b1; 
expect(@(posedge clk) ##[1：16]  
                    memory_ready == 1'b1) 
    $display("Hand shake successful\n"); 
else 
begin
    $display("Hand shake failed： exiting\n") 
    $finish(); 
end 
for(i=0; i<64; i++) 
begin 
    send_packet(); 
    $display("PACKET %0d sent\n"，i); 
end 
end 
```
## 使用局部变量的SVA
```systemverilog
property p_local_var1; 
int lvar1; 
@(posedge clk)  
($rose(enable1)，lvar1 = a) |->  
                ##4 (aa == (lvar1*lvar1*lvar1)); 
endproperty 
a_local_var1： assert property(p_local_var1); 
```
## 在序列匹配时调用子程序
```systemverilog
sequence s_display1; 
@(posedge clk)  
    ($rose(a)，$display("Signal a arrived at %t\n"，$time)); 
endsequence 
sequence s_display2; 
@(posedge clk)  
    ($rose(b)，$display("Signal b arrived at %t\n"，$time)); 
endsequence 
property p_display_window; 
@(posedge clk)  
    s_display1 |-> ##[2：5] s_display2; 
endproperty 
a_display_window ： assert property(p_display_window); 
```
## 将SVA与设计连接
design RTL
```systemverilog
module inline(clk，a，b，d1，d2，d); 
input logic clk，a，b; 
input logic [7：0] d1，d2; 
output logic [7：0] d;    //个人习惯合并写在module声明里

always@(posedge clk) 
begin 
if(a) 
    d <= d1; 
if(b) 
    d <= d2; 
end 
endmodule
```
SVA检验器
```systemverilog
module mutex_chk(a，b，clk); 
input logic a，b，clk;      //个人习惯合并写在module声明里
property p_mutex; 
    @(posedge clk) not (a && b); 
endproperty 
a_mutex： assert property(p_mutex); 
endmodule 
```
顶层
```systemverilog
module top (..); 
inline u1 (clk，a，b，in1，in2，out1); 
inline u2 (clk，c，d，in3，in4，out2); 
endmodule
```
连接SVA检验器与顶层
```systemverilog
bind top.u1 mutex_chk i1(a，b，clk); 
bind top.u2 mutex_chk i2(c，d，clk); 
```
## SVA与功能覆盖
```systemverilog
c_mutex： cover property(p_mutex); 
```

## 一个例子
设计模块 fifo_ctrl.sv
```systemverilog
module fifo_ctrl #(
    parameter DEPTH = 8,
    parameter ADDR_WIDTH = $clog2(DEPTH)
)(
    input logic clk,
    input logic rst_n,
    input logic wr_en,
    input logic rd_en,
    output logic full,
    output logic empty,
    output logic [ADDR_WIDTH-1:0] wr_ptr,
    output logic [ADDR_WIDTH-1:0] rd_ptr
);

    // 内部信号
    logic [ADDR_WIDTH:0] wr_ptr_reg;
    logic [ADDR_WIDTH:0] rd_ptr_reg;
    
    // 指针更新逻辑
    always_ff @(posedge clk or negedge rst_n) begin
        if (!rst_n) begin
            wr_ptr_reg <= '0;
            rd_ptr_reg <= '0;
        end else begin
            if (wr_en && !full) begin
                wr_ptr_reg <= wr_ptr_reg + 1;
            end
            if (rd_en && !empty) begin
                rd_ptr_reg <= rd_ptr_reg + 1;
            end
        end
    end
    
    // 输出当前指针（去掉最高位）
    assign wr_ptr = wr_ptr_reg[ADDR_WIDTH-1:0];
    assign rd_ptr = rd_ptr_reg[ADDR_WIDTH-1:0];
    
    // 空满标志逻辑
    assign full = (wr_ptr_reg[ADDR_WIDTH] != rd_ptr_reg[ADDR_WIDTH]) && 
                  (wr_ptr_reg[ADDR_WIDTH-1:0] == rd_ptr_reg[ADDR_WIDTH-1:0]);
    assign empty = (wr_ptr_reg == rd_ptr_reg);
    
endmodule
```
断言属性定义 fifo_props.sv
```systemverilog
package fifo_props_pkg;

    // 序列定义
    sequence s_no_wr_when_full;
        !(fifo_ctrl.full && fifo_ctrl.wr_en);
    endsequence
    
    sequence s_no_rd_when_empty;
        !(fifo_ctrl.empty && fifo_ctrl.rd_en);
    endsequence
    
    sequence s_ptr_increment_after_wr;
        fifo_ctrl.wr_en && !fifo_ctrl.full ##1 !fifo_ctrl.wr_en |-> 
        (fifo_ctrl.wr_ptr == ($past(fifo_ctrl.wr_ptr) + 1));
    endsequence
    
    sequence s_ptr_increment_after_rd;
        fifo_ctrl.rd_en && !fifo_ctrl.empty ##1 !fifo_ctrl.rd_en |-> 
        (fifo_ctrl.rd_ptr == ($past(fifo_ctrl.rd_ptr) + 1);
    endsequence
    
    sequence s_full_assertion;
        ((fifo_ctrl.wr_ptr_reg[ADDR_WIDTH] != fifo_ctrl.rd_ptr_reg[ADDR_WIDTH]) && 
         (fifo_ctrl.wr_ptr_reg[ADDR_WIDTH-1:0] == fifo_ctrl.rd_ptr_reg[ADDR_WIDTH-1:0])) |-> 
        fifo_ctrl.full;
    endsequence
    
    sequence s_empty_assertion;
        (fifo_ctrl.wr_ptr_reg == fifo_ctrl.rd_ptr_reg) |-> fifo_ctrl.empty;
    endsequence
    
    sequence s_no_simultaneous_wr_rd_after_reset;
        $rose(fifo_ctrl.rst_n) |-> ##[0:10] (fifo_ctrl.wr_en && fifo_ctrl.rd_en);
    endsequence
    
    // 属性定义
    property p_no_wr_when_full;
        @(posedge fifo_ctrl.clk) disable iff (!fifo_ctrl.rst_n)
        s_no_wr_when_full;
    endproperty
    
    property p_no_rd_when_empty;
        @(posedge fifo_ctrl.clk) disable iff (!fifo_ctrl.rst_n)
        s_no_rd_when_empty;
    endproperty
    
    property p_ptr_increment_after_wr;
        @(posedge fifo_ctrl.clk) disable iff (!fifo_ctrl.rst_n)
        s_ptr_increment_after_wr;
    endproperty
    
    property p_ptr_increment_after_rd;
        @(posedge fifo_ctrl.clk) disable iff (!fifo_ctrl.rst_n)
        s_ptr_increment_after_rd;
    endproperty
    
    property p_full_assertion;
        @(posedge fifo_ctrl.clk) disable iff (!fifo_ctrl.rst_n)
        s_full_assertion;
    endproperty
    
    property p_empty_assertion;
        @(posedge fifo_ctrl.clk) disable iff (!fifo_ctrl.rst_n)
        s_empty_assertion;
    endproperty
    
    property p_no_simultaneous_wr_rd_after_reset;
        @(posedge fifo_ctrl.clk)
        s_no_simultaneous_wr_rd_after_reset;
    endproperty
    
    // 覆盖率属性
    property p_cov_wr_while_full;
        @(posedge fifo_ctrl.clk) fifo_ctrl.full && fifo_ctrl.wr_en;
    endproperty
    
    property p_cov_rd_while_empty;
        @(posedge fifo_ctrl.clk) fifo_ctrl.empty && fifo_ctrl.rd_en;
    endproperty
    
endpackage
```
绑定模块 fifo_bind.sv
```systemverilog
module fifo_ctrl_assertions(
    input logic clk,
    input logic rst_n,
    input logic wr_en,
    input logic rd_en,
    input logic full,
    input logic empty,
    input logic [ADDR_WIDTH-1:0] wr_ptr,
    input logic [ADDR_WIDTH-1:0] rd_ptr,
    input logic [ADDR_WIDTH:0] wr_ptr_reg,
    input logic [ADDR_WIDTH:0] rd_ptr_reg
);
    
    import fifo_props_pkg::*;
    
    // 假设参数传递
    parameter ADDR_WIDTH = 3;
    
    // 直接断言
    always @(posedge clk) begin
        // 立即断言
        assert (!(full && empty)) else $error("FIFO cannot be full and empty at the same time");
    end
    
    // 并发断言
    assert property (p_no_wr_when_full) else $error("Write while FIFO is full");
    assert property (p_no_rd_when_empty) else $error("Read while FIFO is empty");
    assert property (p_ptr_increment_after_wr) else $error("Write pointer not incremented after write");
    assert property (p_ptr_increment_after_rd) else $error("Read pointer not incremented after read");
    assert property (p_full_assertion) else $error("Full flag not set correctly");
    assert property (p_empty_assertion) else $error("Empty flag not set correctly");
    assert property (p_no_simultaneous_wr_rd_after_reset) else $error("No simultaneous write and read after reset");
    
    // 覆盖点
    cover property (p_cov_wr_while_full);
    cover property (p_cov_rd_while_empty);
    
    // 序列覆盖
    sequence s_fifo_transition_empty_to_full;
        fifo_ctrl.empty ##1 fifo_ctrl.full[*1:$] ##1 !fifo_ctrl.full;
    endsequence
    
    cover property (s_fifo_transition_empty_to_full);
    
endmodule

// 绑定模块到设计
bind fifo_ctrl fifo_ctrl_assertions fifo_assert_inst (
    .clk(clk),
    .rst_n(rst_n),
    .wr_en(wr_en),
    .rd_en(rd_en),
    .full(full),
    .empty(empty),
    .wr_ptr(wr_ptr),
    .rd_ptr(rd_ptr),
    .wr_ptr_reg(wr_ptr_reg),
    .rd_ptr_reg(rd_ptr_reg)
);
```
tb顶层 fifo_tb.sv
```systemverilog
`timescale 1ns/1ps

module fifo_tb;
    import fifo_props_pkg::*;
    
    // 参数
    parameter DEPTH = 8;
    parameter ADDR_WIDTH = $clog2(DEPTH);
    
    // 时钟和复位
    logic clk;
    logic rst_n;
    
    // DUT接口
    logic wr_en;
    logic rd_en;
    logic full;
    logic empty;
    logic [ADDR_WIDTH-1:0] wr_ptr;
    logic [ADDR_WIDTH-1:0] rd_ptr;
    
    // 实例化DUT
    fifo_ctrl #(
        .DEPTH(DEPTH),
        .ADDR_WIDTH(ADDR_WIDTH)
    ) dut (
        .clk(clk),
        .rst_n(rst_n),
        .wr_en(wr_en),
        .rd_en(rd_en),
        .full(full),
        .empty(empty),
        .wr_ptr(wr_ptr),
        .rd_ptr(rd_ptr)
    );
    
    // 时钟生成
    initial begin
        clk = 0;
        forever #5 clk = ~clk;
    end
    
    // 测试序列
    initial begin
        // 初始化
        wr_en = 0;
        rd_en = 0;
        rst_n = 0;
        
        // 复位
        #10 rst_n = 1;
        
        // 测试1: 写满FIFO
        $display("Test 1: Filling the FIFO");
        for (int i = 0; i < DEPTH; i++) begin
            @(posedge clk);
            wr_en = 1;
            @(posedge clk);
            wr_en = 0;
        end
        
        // 测试2: 尝试在满时写入
        $display("Test 2: Attempting to write when full");
        @(posedge clk);
        wr_en = 1;
        @(posedge clk);
        wr_en = 0;
        
        // 测试3: 读空FIFO
        $display("Test 3: Emptying the FIFO");
        for (int i = 0; i < DEPTH; i++) begin
            @(posedge clk);
            rd_en = 1;
            @(posedge clk);
            rd_en = 0;
        end
        
        // 测试4: 尝试在空时读取
        $display("Test 4: Attempting to read when empty");
        @(posedge clk);
        rd_en = 1;
        @(posedge clk);
        rd_en = 0;
        
        // 测试5: 同时读写
        $display("Test 5: Simultaneous read and write");
        repeat(10) begin
            @(posedge clk);
            wr_en = $urandom_range(0,1);
            rd_en = $urandom_range(0,1);
        end
        
        // 结束仿真
        #100 $finish;
    end
    
    // 波形转储
    initial begin
        $dumpfile("fifo_wave.vcd");
        $dumpvars(0, fifo_tb);
    end
    
    // 断言绑定 - 已经在fifo_bind.sv中完成
    
endmodule
```
