# Understanding Control-C

> **Bash:** 5.1, commit [9439ce094c9a](https://git.savannah.gnu.org/cgit/bash.git/commit/?id=9439ce094c9aa7557a9d53ac7b412a23aa66e36b) ("Bash-5.1 patch 16: fix interpretation of multiple instances of ! in \[\[ conditional commands")\
> **Linux:** 5.18-rc5, commit [9c095bd0d4c4](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=9c095bd0d4c451d31d0fd1131cc09d3b60de815d) ("Merge branch 'hns3-next'")\
> **QEMU:** 7.0.50, commit [2d20a57453f6](https://repo.or.cz/qemu/armbru.git/commit/2d20a57453f6a206938cbbf77bed0b378c806c1f) ("Merge tag 'pull-fixes-for-7.1-200422-1' of ht<span>tps://github.com/stsquad/qemu into staging")

---

```terminal
ypl@home:~$ sudo make isntall
make: *** No rule to make target 'isntall'.  Stop.
ypl@home:~$ sudo make isntal^C
ypl@home:~$ 
ypl@home:~$ aargh
-bash: aargh: command not found
```

In this post, I'll (try to...) show you what happens in Linux when you press `<Ctrl-C>` in an interactive `bash` shell.

## Customizing your `^C`

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

Yay! You just customized your `^C` (a.k.a. "special control characters").

## WIP

Tired, more stuff coming soon...
