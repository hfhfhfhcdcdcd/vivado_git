# 一、我想做什么
## 1. 我想做什么
## 2. 

# 二、代码
## 1. 设计代码
```
module uart_data_tx(                           
input sys_clk         ,          
input rst_n           ,          
input [2:0] time_set  ,   //基础计数器的设置       
input [7:0] data      ,          
input send_en         ,          
output reg uart_tx    ,          
output reg tx_done  
    );
/*-----------------------变量的声明-----------------------------*/
reg [31:0] cnt;//基本计数器
reg [3:0] cnt2;//2级定时器
reg [31:0] time_cnt ;
/*-----------------------设置时间间隔-----------------------------*/ 
always@(*)
if(!rst_n)
    time_cnt<=4800;
else 
    case(time_set)  
        0:time_cnt<=10416;                 //4800; 
        1:time_cnt<=5208;                  //9600; 
        2:time_cnt<=434;                   //115200;
        default:time_cnt<=434;             //115200;
    endcase
/*-----------------------基本计数器-----------------------------*/
always@(posedge sys_clk or negedge rst_n)
if(!rst_n)
    cnt<=32'd0;
else if(send_en)
    if(cnt==time_cnt-1)
        cnt<=32'd0;
    else
        cnt<=cnt+1;
else//!send_en
    cnt<=32'd0;         
/*-----------------------2级计数器-----------------------------*/
always@(posedge sys_clk or negedge rst_n)
if(!rst_n)
    cnt2<=4'd0;//默认发start位
else if(send_en)begin
    if((cnt2>=0)&&(cnt2<10))begin
        if(cnt==time_cnt-1)
            cnt2<=cnt2+1;
        else  
            cnt2<=cnt2;
     end
     else if(cnt2==10)
        cnt2<=0;//cnt2的清零
     else  
            cnt2<=cnt2;
end
else //!send_en
    cnt2<=4'd0;
/*-----------------------uart_tx-----------------------------*/
always@(posedge sys_clk or negedge rst_n)
if(!rst_n)begin
    uart_tx<=0;
    tx_done<=0;
    end
else
    case(cnt2)
        0: begin uart_tx<=0; tx_done<=0; end                        
        1:  uart_tx<=data[0] ;                  
        2:  uart_tx<=data[1] ;                  
        3:  uart_tx<=data[2] ;                  
        4:  uart_tx<=data[3] ;                  
        5:  uart_tx<=data[4] ;                  
        6:  uart_tx<=data[5] ;                  
        7:  uart_tx<=data[6] ;                  
        8:  uart_tx<=data[7] ;                  
        9:  begin uart_tx<=1 ; tx_done<=1; end      
        default:begin uart_tx<=1;tx_done<=0; end  
      endcase                                       
endmodule                                         
```

## 2. 仿真代码
```
`timescale 1ns / 1ps
module tb;
reg sys_clk            ;
reg rst_n              ;
reg [2:0] time_set    ;
reg [7:0]  data        ;
reg send_en            ;
wire uart_tx           ;
wire tx_done           ;
/*---------------------------例化-------------------------*/
uart_data_tx u1(
       .sys_clk    (sys_clk)           ,
       .rst_n      (rst_n  )           ,
       .time_set   (time_set)          ,
       .data       (data   )           ,
       .send_en    (send_en)           ,
       .uart_tx    (uart_tx)           ,
       .tx_done    (tx_done)
    );
/*----------------------------时钟初始化-----------------------*/
initial 
sys_clk=0;
always #10 sys_clk=~sys_clk;
/*----------------------------初始化其余参数-------------------*/
initial begin
rst_n   =0;  
time_set=2;  
data    =8'd0;  
send_en =0;
#201;
rst_n   =1;   
#100;
/*------- 第 1 次 发送数据 -----*/
data=8'b0101_0101;
send_en=1;
@(posedge tx_done)
#8689
send_en=0;
/*------- 第 2 次 发送数据  -----*/
#20000;
data=8'b0000_1111;
send_en=1;
@(posedge tx_done)
#8689
send_en=0;
/*------- 第 3 次 发送数据  -----*/
#20000;
data=8'b1111_1111;
send_en=1;
@(posedge tx_done)
#8689
send_en=0;
$stop;
end
endmodule

```
## 3. 设计代码的注释

```
module uart_data_tx(                           
input sys_clk         ,          
input rst_n           ,          
input [2:0] time_set  ,   //基础计数器的设置       
input [7:0] data      ,          
input send_en         ,          
output reg uart_tx    ,          
output reg tx_done  
    );
一些端口：

==time_set==是一个可以操控1010_1010每个bit之间时间间隔的端口；它的位宽是3位2进制数，可以表达0~8的10进制数。time_set=0、time_set=1......被赋予不同的值代表的含义：time_set=0代表每个bit的间隔时间是4800，time_set=1代表间隔时间是9600......。

【4800、9600、115200都是波特率，波特率是串口通信中用来描述传输速率的单位。波特率是4800，代表1s传输4800个bit，发送一个bit的时间就是1s/4800=1000_000_000ns/4800=208333ns。晶振的一个周期（一上一下）消耗20ns，那么208333就可以记10416个周期。相应的把9600、115200传一个bit消耗的周期算出来，分别是5208、434。】

因此time_set不同的值，可以操控波特率的变化，继而操控是打434个拍子传1bit，还是打5208个拍子传1bit。

==data==是要传输的8bit数据位。在里面写我们想要传的数据。具体要在测试文件里面写入。

==send_en==是使能信号。它拉高代表可以传数据；拉低代表停止传数据。

==uart_tx==：听我解释为什么会多出来这个东西。data是8位的并行数据，串口通信的目的就是要把并行数据串行发出去，并行数据有了，要把并行数据的每一位传给uart_tx，让它在不同的时刻发出去，这就是串行发送。当然这个时间差就是波特率，虽说是分时刻传出去，但我们肉眼是无法捕捉到的，它很快。

==tx_done==是发送完的标志信号，tx_done在数据发完之后拉高，其余时间拉低。拉高代表传输完成，send_en就要在tx_done拉高的同时拉低，停止传输数据。

/*-----------------------变量的声明-----------------------------*/
reg [31:0] cnt;//基本计数器
reg [3:0] cnt2;//2级定时器
reg [31:0] time_cnt ;
/*-----------------------设置时间间隔-----------------------------*/ 
always@(*)
if(!rst_n)
    time_cnt<=4800;
else 
    case(time_set)  
        0:time_cnt<=10416;                 //4800; 
        1:time_cnt<=5208;                  //9600; 
        2:time_cnt<=434;                   //115200;
        default:time_cnt<=434;             //115200;
    endcase
/*-----------------------基本计数器-----------------------------*/
always@(posedge sys_clk or negedge rst_n)
if(!rst_n)
    cnt<=32'd0;
else if(send_en)
    if(cnt==time_cnt-1)
        cnt<=32'd0;
    else
        cnt<=cnt+1;
else//!send_en
    cnt<=32'd0;                                  
/*-----------------------2级计数器-----------------------------*/
always@(posedge sys_clk or negedge rst_n)
if(!rst_n)
    cnt2<=4'd0;//默认发start位
else if(send_en)begin
    if((cnt2>=0)&&(cnt2<10))begin
        if(cnt==time_cnt-1)
            cnt2<=cnt2+1;
        else  
            cnt2<=cnt2;
     end
     else if(cnt2==10)
        cnt2<=0;//cnt2的清零
     else  
            cnt2<=cnt2;
end
else //!send_en
    cnt2<=4'd0;
/*-----------------------uart_tx-----------------------------*/
always@(posedge sys_clk or negedge rst_n)
if(!rst_n)begin
    uart_tx<=0;
    tx_done<=0;
    end
else
    case(cnt2)
        0: begin uart_tx<=0; tx_done<=0; end                        
        1:  uart_tx<=data[0] ;                  
        2:  uart_tx<=data[1] ;                  
        3:  uart_tx<=data[2] ;                  
        4:  uart_tx<=data[3] ;                  
        5:  uart_tx<=data[4] ;                  
        6:  uart_tx<=data[5] ;                  
        7:  uart_tx<=data[6] ;                  
        8:  uart_tx<=data[7] ;                  
        9:  begin uart_tx<=1 ; tx_done<=1; end      
        default:begin uart_tx<=1;tx_done<=0; end  
      endcase                                       
endmodule                                         
```
## 4. 仿真代码的注释
# 三、