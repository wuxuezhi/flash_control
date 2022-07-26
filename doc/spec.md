# interact with SPI flash
与spi的最小交互单位为一个COMMAND（以下称之为命令）。每个命令由instruction phase(以下称之为指令阶段)、address phase(地址阶段)、dummy phase（dummy阶段）、data phase（数据阶段）组成。
## standard/dual/quad SPI
standard/dual/quad SPI仅仅定义了接口的传输线的数量。不同的flash厂商对SPI有自己的规定。但大部分都遵守以下约定。
* 指令码05H为"read status register", standard SPI传输。
* 向SPI flash传输任何命令前（除了READ STATUS REGISTER命令），最好确认flash不处于忙状态（即Write In Process为低）。
* 指令码03H为"read data", standard SPI传输。
* 指令码06H为"write enable"，standard SPI传输。
* "Dual Output Fast Read"，指令阶段、地址阶段、dummy阶段为standard SPI,数据阶段为dual SPI
* "Quad Output Fast Read",指令阶段、地址阶段、dummy阶段为standard SPI,数据阶段为quad SPI
* "Dual Input/Output(I/O) Fast Read",指令阶段、为standard SPI,地址阶段、dummy 阶段、数据阶段为duad SPI
* "Qual Input/Output(I/O) Fast Read",指令阶段、为standard SPI,地址阶段、dummy 阶段、数据阶段为quad SPI

## Command reg
|           | instruction valid | instruction code|address valid|address width|addr wire width|dummy width|dummy wire width|data valid|data width|data input|data wire width|
|:----------|:-----------------:|:---------------:|:-----------:|:-----------:|:-------------:|:---------:|:----------:|:----------:|:-----------------:|:---:|:-------------:|
| Bit Field |     31            | 30:23           |22           |21:20        |19:18          |17:14      |   13:12      |11   |10:3          |2    |1:0|
* xxx valid 代表该阶段有效。如address valid 为0，代表这个命令没有地址阶段。 
* address width: 00 代表地址长度为1B, 01 为2 Byte， 10 为3B，11为4B。
* xxx wire width:00 代表该阶段的传输为standard SPI, 01 为dual SPI， 10 为quad SPI， 11 reserved.
* dummy width代表dummy 传输 "dummy width"个时钟周期，4‘b0000代表无dummy phase,4'b1111代表15个时钟周期。
* 数据阶段字节数 = data width + 1。意味着最低为一个字节，最高为256B.(注：当前的设计内部数据寄存器配置为64B，因此该字段虽然有8位，但请仅仅使用低6位)。

## Flash Controller memory map
![](flash_controller_memory.png)
flash controller 从外面接收AXI传过来的交易，将其转换为SPI的COMMAND。
* 低24MB 地址为普通memory，高4KB地址为flash controller 内部寄存器。

## Flash Controller Reg Address
* R 代表该寄存器是可读的
* W 代表该寄存器是可写的
* reset val 为寄存器的复位值

|reg idx|address| reg name | property |reset val| Note|
|:------|:-----:|:--------:|:--------:|:-------:|:---:|
|0      |0x1800000|Command 0 | RW     |0x81f0083c|1,2 |
|1      |0x1800004|Command 1 | RW     |0x82800804| 3  |
|2      |0x1800008|Command 2 | RW     |0x00000000|    |
|3      |0x180000c|Command 3 | RW     |0x00000000|    |
|4      |0x1800010|Command 4 | RW     |0x00000000|    |
|5      |0x1800014|Command 5 | RW     |0x00000000|    |
|6      |0x1800018|Command 6 | RW     |0x00000000|    |
|7      |0x180001c|Command 7 | RW     |0x00000000|    |
|8      |0x1800020|addr buffer|RW     |0x00000000|  4 |
|9      |0x1800024|data buffer|RW     |0X00000000| 5  |
|       |0x1800100|Command 0 trigger|W  |          | 6    |





1. AXI master 向 Flash memory空间的读交易将触发0号寄存器中的命令。该读命令的地址为AXI master发送的读交易的读地址通道中的地址。 并且命令返回的数据在AXI read transaction的读数据通道返回。由于0号寄存器的复位值为"read data"命令，因此上电后可以直接从flash读数据。用户可以更改Command 0内容。
2. 0x81f0083c为”READ DATA”命令，standard spi传输、8B的读数据,3B 地址。
3. 0x82800804为“READ STATUS REGISTER" 命令，standard spi传输、1B的读数据。
4. 除了自动触发COMMAND（如向memory空间的读会触发Command 0), 其它命令的地址阶段将会使用addr buf寄存器中的值。
5. data buffer位宽为64B,地址跨度从0x1800024~0x1800060.从flash返回的数据会存到这里（Command 0 和 Command 1 返回的数据不会存入data buffer)。向flash(而不是flash controller)发送的数据来自这里。 