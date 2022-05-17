---
title: "Understanding Control-C"
---

> **arch:** x86_64\
> **Bash:** 5.1, commit [9439ce094c9a](https://git.savannah.gnu.org/cgit/bash.git/commit/?id=9439ce094c9aa7557a9d53ac7b412a23aa66e36b) ("Bash-5.1 patch 16: fix interpretation of multiple instances of ! in \[\[ conditional commands")\
> **Linux:** 5.18-rc5, commit [9c095bd0d4c4](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=9c095bd0d4c451d31d0fd1131cc09d3b60de815d) ("Merge branch 'hns3-next'")\
> **QEMU:** 7.0.50, commit [2d20a57453f6](https://repo.or.cz/qemu/armbru.git/commit/2d20a57453f6a206938cbbf77bed0b378c806c1f) ("Merge tag 'pull-fixes-for-7.1-200422-1' of ht<span>tps://github.com/stsquad/qemu into staging")

```terminal
ypl@home:~$ sudo make isntall
make: *** No rule to make target 'isntall'.  Stop.
ypl@home:~$ sudo make isntal^C
ypl@home:~$ 
ypl@home:~$ aargh
-bash: aargh: command not found
```

This post briefly describes what happens in Linux when you press `<Ctrl-C>`.

## Customizing Your ^C

...but why not have some fun first? Ever wondered where does that `^C` string come from? Tired of it? Apply this to your `bash`:

```diff
diff --git a/lib/readline/signals.c b/lib/readline/signals.c
index f9174ab8a014..93b4b637122a 100644
--- a/lib/readline/signals.c
+++ b/lib/readline/signals.c
@@ -765,7 +765,7 @@ rl_echo_signal_char (int sig)

   if (CTRL_CHAR (c) || c == RUBOUT)
     {
-      cstr[0] = '^';
+      cstr[0] = '%';
       cstr[1] = CTRL_CHAR (c) ? UNCTRL (c) : '?';
       cstr[cslen = 2] = '\0';
     }
```

```terminal
ypl@home:~/bash$ ^C
ypl@home:~/bash$ ./bash
ypl@home:~/bash$ %C
ypl@home:~/bash$ 
```

Yay! You just customized your `^C`!

## 8250 UART Serial Driver

I'm using `qemu-system-x86_64` (`-nographic`) because it's easier. It seems that, whenever I press `<Ctrl-C>`, my guest Linux's [8250 UART](https://en.wikipedia.org/wiki/8250_UART) serial driver receives an interrupt, so let's start there!

> Feeling adventurous? Start from `common_interrupt()`, or even QEMU instead!

```
drivers/tty/serial/8250/8250_core.c:serial8250_interrupt()   /* port->handle_irq() */
                          8250_port.c:serial8250_default_handle_irq()
                                       :serial8250_handle_irq()
                                         :serial8250_rx_chars()
```

Let's take a closer look at `serial8250_rx_chars()`:
   
```c
unsigned char serial8250_rx_chars(struct uart_8250_port *up, unsigned char lsr)
{
	struct uart_port *port = &up->port;
	int max_count = 256;

	do {
		serial8250_read_char(up, lsr);
		if (--max_count == 0)
			break;
		lsr = serial_in(up, UART_LSR);
	} while (lsr & (UART_LSR_DR | UART_LSR_BI));

	tty_flip_buffer_push(&port->state->port);
	return lsr;
}
```

Here,

1. `serial8250_read_char()` pushes this `<Ctrl-C>` to the [TTY flip buffer](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/Documentation/tty/tty_buffer.rst?id=9c095bd0d4c451d31d0fd1131cc09d3b60de815d#n4) by calling `uart_insert_char()`;
2. `tty_flip_buffer_push()` then queues a work to push this TTY flip buffer to the [line discipline (LDISC)](https://en.wikipedia.org/wiki/Line_discipline).

## TTY Core

Next, we traverse a few TTY core functions before getting to the line discipline:

```
drivers/tty/tty_buffers.c:tty_flip_buffer_push()   	/* queue_work() */
...
                         :flush_to_ldisc()
                           :receive_buf() 		/* port->client_ops->receive_buf() */
                   tty_port.c:tty_port_default_receive_buf()
                   tty_buffer.c:tty_ldisc_receive_buf() /* ld->ops->receive_buf2() */
```

We will be dealing with [N_TTY](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/Documentation/tty/n_tty.rst?id=9c095bd0d4c451d31d0fd1131cc09d3b60de815d#n4), the default line discipline. See `tty_ldisc_init()`:

```c
int tty_ldisc_init(struct tty_struct *tty)
{
	struct tty_ldisc *ld = tty_ldisc_get(tty, N_TTY);  /* default to N_TTY */

	if (IS_ERR(ld))
		return PTR_ERR(ld);
	tty->ldisc = ld;
	return 0;
}
```

## N_TTY

```
drivers/tty/n_tty.c:n_tty_receive_buf2()
                     :n_tty_receive_buf_common()
                       :__receive_buf()
                         :n_tty_receive_buf_standard()
                           :n_tty_receive_char_special()
                             :n_tty_receive_signal_char()
                               :isig()
                                 :__isig()
```

In `n_tty_receive_buf_standard()`, N_TTY realizes that `<Ctrl-C>` is a [control character](https://en.wikipedia.org/wiki/Control_character) (ASCII 3, the [End-of-Text character](https://en.wikipedia.org/wiki/End-of-Text_character)) and handles it specially. Look for `ldata->char_map`.

Finally, `__isig()` sends a `SIGINT` to [the process group controlling the tty](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/tty/tty_jobctrl.c?id=9c095bd0d4c451d31d0fd1131cc09d3b60de815d#n413), as specified in [POSIX 1003.1 Section 7.1.1.9, Special Characters](https://www.govinfo.gov/content/pkg/GOVPUB-C13-bf1fc57a5dbcaa993cebad99aca83f64/pdf/GOVPUB-C13-bf1fc57a5dbcaa993cebad99aca83f64.pdf#page=155):

> It generates a SIGINT signal that is sent to all processes in the foreground process group for which the terminal is the controlling terminal.
	
## Signal Handling

Briefly,

```
kernel/signal.c:kill_pgrp()
                 :__kill_pgrp_info()
                   :group_send_sig_info()
                     :do_send_sig_info()
                       :send_signal()
                         :__send_signal()
     include/linux/signal.h:sigaddset()
```

Let's say I pressed `<Ctrl-C>` to stop `yes`. `__send_signal()` adds a pending `SIGINT` for `yes`, to be handled by the kernel later. For example, when `yes` returns from `write(2)`:
   
```
arch/x86/entry/common.c:syscall_exit_to_user_mode()
    kernel/entry/common.c:__syscall_exit_to_user_mode_work()
                           :exit_to_user_mode_prepare()
                             :exit_to_user_mode_loop()
       arch/x86/kernel/signal.c:arch_do_signal_or_restart()
                  kernel/signal.c:get_signal()
                      kernel/exit.c:do_group_exit()
                                     :do_exit()
                    kernel/sched/core.c:do_task_dead()
```
	
`get_signal()` calls `do_group_exit()` to terminate `yes`, since that's the default action for `SIGINT`. See [POSIX 1003.1 Table 3-1, Required Signals](https://www.govinfo.gov/content/pkg/GOVPUB-C13-bf1fc57a5dbcaa993cebad99aca83f64/pdf/GOVPUB-C13-bf1fc57a5dbcaa993cebad99aca83f64.pdf#page=74).

On the other hand, if the process registered a handler for `SIGINT`, `get_signal()` returns to `arch_do_signal_or_restart()`:

```c
void arch_do_signal_or_restart(struct pt_regs *regs, bool has_signal)
{
	struct ksignal ksig;

	if (has_signal && get_signal(&ksig)) {
		/* Whee! Actually deliver the signal.  */
		handle_signal(&ksig, regs);
		return;
	}
...
```

`handle_signal()` will "actually deliver the signal" to user space. This is true for `bash`, for example.

That's it! "Whee!"

## Appendix A: Signal Handling in [GNU Readline](https://tiswww.case.edu/php/chet/readline/rltop.html)

Interestingly, when I press `<Ctrl-C>` in `bash`, `bash` actually receives `SIGINT` **twice**. See my `printk()` output:

```
[   17.337664] [bash, 0xffff96428638ba00] arch_do_signal_or_restart(): Whee! delivering SIGINT.
[   17.348565] [bash, 0xffff96428638ba00] arch_do_signal_or_restart(): Whee! delivering SIGINT.
```

`bash` uses GNU Readline to read input from user. When I press `<Ctrl-C>`, the first `SIGINT` is delivered to GNU Readline's own `SIGINT` handler, which [performs some special processing](https://docs.rtems.org/releases/4.5.1-pre3/toolsdoc/gdb-5.0-docs/readline/readline00030.html), reinstalls `bash`'s `SIGINT` handler, then sends a second `SIGINT` to `bash` itself (the same process).
