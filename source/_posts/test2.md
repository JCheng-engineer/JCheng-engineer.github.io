# 魔键触发方式修改

首先用于系统shell失去响应时进行问题诊断。目前的CPU架构和串口终端工具对ctrl+break触发sysrq适配不好，所以CGEL开发了新触发方式，sysrq的新触发方式为 ctrl+6 再加命令功能键。命令功能键与开源linux命令功能键完全一致。虽说是新的触发的方式，但在底层实现上均是依靠break error 寄存器的读写来实现对sysrq的处理，依旧存在一些问题。

如在nxp 的串口驱动实现，代码如下：

```c
static void serial8250_read_char(struct uart_8250_port *up, unsigned char lsr)

{

​	struct uart_port *port = &up->port;

​	unsigned char ch;

​	char flag = TTY_NORMAL;

​	if (likely(lsr & UART_LSR_DR))

​		ch = serial_in(up, UART_RX);

​	else

​		/*

​		 * Intel 82571 has a Serial Over Lan device that will

​		 * set UART_LSR_BI without setting UART_LSR_DR when

​		 * it receives a break. To avoid reading from the

​		 * receive buffer without UART_LSR_DR bit set, we

​		 * just force the read character to be 0

​		 */

​		ch = 0;

​	port->icount.rx++;

​	/*

  \* Add new keypress Ctrl+6 for Sysrq,

  \* 0x1e is the ASCII of Ctrl+6.

  */

​	if (ch == 0x1e) {

​		if (up->port.cons && up->port.cons->index == up->port.line)

​				lsr |= UART_LSR_BI | UART_MSR_DCD;

​	}

 

​	lsr |= up->lsr_saved_flags;

​	up->lsr_saved_flags = 0;

 

​	if (unlikely(lsr & UART_LSR_BRK_ERROR_BITS)) {

​		if (lsr & UART_LSR_BI) {

​			lsr &= ~(UART_LSR_FE | UART_LSR_PE);

​			port->icount.brk++;

​			/*

​			 * We do the SysRQ and SAK checking

​			 * here because otherwise the break

​			 * may get masked by ignore_status_mask

​			 * or read_status_mask.

​			 */

​			if (uart_handle_break(port))

​				return;

​		} else if (lsr & UART_LSR_PE)

​			port->icount.parity++;

​		else if (lsr & UART_LSR_FE)

​			port->icount.frame++;

​		if (lsr & UART_LSR_OE)

​			port->icount.overrun++;

 

​		/*

​		 * Mask off conditions which should be ignored.

​		 */

​		lsr &= port->read_status_mask;

 

​		if (lsr & UART_LSR_BI) {

​			pr_debug("%s: handling break\n", __func__);

​			flag = TTY_BREAK;

​		} else if (lsr & UART_LSR_PE)

​			flag = TTY_PARITY;

​		else if (lsr & UART_LSR_FE)

​			flag = TTY_FRAME;

​	}

​	if (uart_handle_sysrq_char(port, ch))

​		return;

​	uart_insert_char(port, lsr, UART_LSR_OE, ch, flag);

}
```

但是目前发现在PC进入睡眠时会发送一个break信号（具体没有通过示波器去测试到底是一个怎样的信号，但是最后结果会产生break error 寄存器的写入），从而误触发魔键，对于误触发的魔键，说简单点是一段可以忽略的打印，但严重的可能会导致单板进程挂掉或者重启（具体可见魔键的功能）。因此屏蔽掉魔键的break这种特殊触发方式，使其变得更加难已触发。

具体为

 ```c
	if (ch == 0x1e) {

​		if (uart_handle_break(port))

​				return;

​		//if (up->port.cons && up->port.cons->index == up->port.line)

​		//lsr |= UART_LSR_BI | UART_MSR_DCD;

​	}

 

​	lsr |= up->lsr_saved_flags;

​	up->lsr_saved_flags = 0;

 

​	if (unlikely(lsr & UART_LSR_BRK_ERROR_BITS)) {

​		if (lsr & UART_LSR_BI) {

​			lsr &= ~(UART_LSR_FE | UART_LSR_PE);

​			port->icount.brk++;

​			return;

​			/*

​			 * We do the SysRQ and SAK checking

​			 * here because otherwise the break

​			 * may get masked by ignore_status_mask

​			 * or read_status_mask.

​			 */

​			//if (uart_handle_break(port))

​				//return;

​		} else if (lsr & UART_LSR_PE)
 ```







 

 
