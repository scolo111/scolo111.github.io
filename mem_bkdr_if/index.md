# memory bkdr vif

### 背景
在ic设计验证中，每个memory都会采用一个verilog模型来模拟memory IP。而每个memory的真实存储单元都会由一个多维数组
来模拟。例如一个flash存储单元内的data区就可能叫:
```systemverilog
reg [DataWidth-1:0] main_mem [Depth-1:0]
```
在IC验证时，如果涉及到memory IP的验证或者SOC的系统级验证时，经常需要对多个memory进行初始化，随机，加载数据，读出数据等操作。
此时如果能有个一个统一的接口绑定用于处理这些操作。那么验证代码将会变得更加精简，高效。
### 源码分析
Opentitan 中的文档介绍如下[README](https://docs.opentitan.org/hw/dv/sv/mem_bkdr_if/README/)

实现原理：将main_mem的hierarchy 路径拆分成2部分。MEM_HIER, MEM_SLICE。
* MEM_HIER为每个memory的根路径，在tb中定义。
```systemverilog
  `define FLASH_DATA_MEM_HIER(i) \
      dut.u_flash_eflash.gen_flash_banks[``i``].i_core.i_flash.gen_generic.u_impl_generic.u_mem
```
* MEM_SLICE在mem_bkdr_if内定义的相对。接口的的方法都可以通过MEM_SLICE这个句柄来操作。
```systemverilog
  `define MEM_ARR_PATH_SLICE gen_generic.u_impl_generic.mem
```
* 通过bind将 MEM_HIER 与 MEM_SLICE 拼接。
```systemverilog
bind `FLASH_DATA_MEM_HIER(i) mem_bkdr_if mem_bkdr_if();
```
* 将接口传入uvm_environment
```systemverilog        
   flash_part_e part;
   part = flash_ctrl_pkg::FlashPartData;
   uvm_config_db#(mem_bkdr_vif)::set(null, "*.env", $sformatf("mem_bkdr_vifs[%0s][%0d]",
       part.name(), i), `FLASH_DATA_MEM_HIER(i).mem_bkdr_if);
```
* 通过接口操作memory
```systemverilog        
   cfg.mem_bkdr_vifs[part][i].set_mem();
```
### 注意事项
* Opentitan中的实现方法需要每个memory内的MEM_SLICE同名。对于不同名的memory，目前还没有找到解决方案。
* MEM_SLICE 定义时需要至少2级hierarchy才能被vcs识别为路径。

