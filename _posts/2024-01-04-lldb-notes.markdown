---
title:  "LLDB notes"
date:   2024-01-4 12:00:00 +0100
categories: [Notes, LLDB]
tags: [LLDB]
---

Working with LLDB, I often come across new things like commands, additional tools, and more.
I'd like to use this post as a sort of notebook where I can jot down important things and notes about LLDB and LLVM in general that I've stumbled upon and wouldn't want to lose.
I hope that in the future, this post will grow into something valuable.

### Commands

Enable logging:
```console
(lldb) log enable -f /LOG/FILE/PATH -v lldb all
```

Run python code from the debugger:
```console
(lldb) script import time; print(f"Time: {time.time()}");
```

Measure the command evaluation time:
```console
(lldb) script import time; s = time.time(); lldb.debugger.HandleCommand("""<COMMAND>"""); time.time() - s
```

But this long command can be aliased:
```bash
echo "command regex measure 's/(.+)/script import time; s = time.time(); lldb.debugger.HandleCommand(\"\"\"%1\"\"\"); time.time() - s;/'" >> ~/.lldbinit
```

And then just
```console
(lldb) measure expr -- demo()
```

Multiline expressions (`### <<` is a comment beginning, no need to write it):
```console
(lldb) expr -- ### << press enter to start the multiline expression from a new line
int x = 1;
x + 2;
### << after this empty line, the expression will be evaluated
```

### Tools

Tool for parsing, dumping, and performing other operations on PDB files:
```bash
llvm-pdbutil -h
```
