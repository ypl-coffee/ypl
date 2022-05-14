[Test Link](ctrl_c.md)

`fs/binfmt_elf.c` 里，`load_elf_binary()` 用一个大 loop 找类型是 `PT_INTERP` 的 program header，如果找到了，就说明需要 interpreter。

截取 `readelf -l` 的输出：

```
  INTERP         0x0000000000000fe4 0x0000000000400fe4 0x0000000000400fe4
                 0x000000000000001c 0x000000000000001c  R      0x1
      [Requesting program interpreter: /lib64/ld-linux-x86-64.so.2]
```

回到 `load_elf_binary()`，如果需要 interpreter：

```c
if (interpreter) {
        elf_entry = load_elf_interp(interp_elf_ex,
                                    interpreter,
                                    load_bias, interp_elf_phdata,
                                    &arch_state);
```

然后，

```c
                elf_entry += interp_elf_ex->e_entry;
```

else，如果不需要 interpreter：

```c
        } else {
                elf_entry = e_entry;
```

最后，
```c
        regs = current_pt_regs();
```

```c
        START_THREAD(elf_ex, regs, elf_entry, bprm->p);
```

对于 x86_64 来说，相当于调用了 `arch/x86/kernel/process_64.c:start_thread()`：

```c
void
start_thread(struct pt_regs *regs, unsigned long new_ip, unsigned long new_sp)
{
        start_thread_common(regs, new_ip, new_sp,
                            __USER_CS, __USER_DS, 0);
}
EXPORT_SYMBOL_GPL(start_thread);
```

```c
        regs->ip                = new_ip;
```

这就相当于直接修改了 current_pt_regs，等 exec() 系统调用结束以后就自动跳到 new_ip 了，应该是这样…
