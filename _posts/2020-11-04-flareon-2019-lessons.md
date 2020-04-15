---
title: What I've learned solving Flare-On 2019
description: Lessons learned and tips for CTFs
date: 2020-04-11 17:45:00 +/-0000
categories: [Write-Ups]
tags: [flareon, ctf]
post_image: /assets/img/2020-11-04-flareon-2019-lessons/flare-on-title.png
image:
  path: /assets/img/2020-11-04-flareon-2019-lessons/preview.png
  height: 628
  width: 1200
seo:
  date_modified: 2020-04-14 13:15:07 +0300
---
# Intro

In this blogpost, I'm going to share a few insights after solving all of the [Flare-On 2019](http://flare-on.com/) challenges. Having access to the awesome [Write-up of write-ups](https://medium.com/@remco_verhoef/flareon6-write-up-of-write-ups-6ead20914ef0) that contains multiple write-ups for every challenge, each taking a different approach, allowed me to really examine the choices I have made solving each challenge and make conclusions for future similar tasks.

This assembly of tips will be useful for anyone who's just starting to play CTFs, allowing them to learn from the experience of others. Having said that, I also think that even though to the experienced CTF player those things may look obvious, it's still useful to get a reminder from time to time. Let's start with some examples from Flare-On and end with some general tips.

> ⚠️ Spoilers for Flare-On 2019 ahead ⚠️ 

# Go through the backdoor

Challenge authors usually have one or two possible solutions in mind when they write a challenge. Most of the time, there are a few additional solutions that the authors didn't think about. Sometimes those solutions can be much easier than the intended one! We can have a look at challenge 7 - `wopr` as an example of that.

This challenge was a *[PyInstaller](https://www.pyinstaller.org/)* Windows executable.

> PyInstaller freezes (packages) Python applications into stand-alone executables, under Windows, GNU/Linux, Mac OS X, FreeBSD, Solaris and AIX.

It can basically take any python script and turn it into a standalone executable. After unpacking and decompiling the python code, the challenge boiled down to this (code modified for simplicity):

```python
h = list(wrong())

launch_code = input().encode()

x = list(launch_code.ljust(16, b'\0'))

b[0] = x[2] ^ x[3] ^ x[4] ^ x[8] ^ x[11] ^ x[14]
...
b[15] = x[1] ^ x[3] ^ x[5] ^ x[9] ^ x[10] ^ x[11] ^ x[13] ^ x[15]

if b == h:
    flag = fire(eye, launch_code).decode()
    print(f"CONGRATULATIONS! YOU FOUND THE FLAG:\n\n{flag}\n")
```

1. `h` is calculated using the function `wrong()`
2. The user inserts the `launch_code`
3. `b` gets initialized using some the bytes from the `launch_code`
4. If `b == h` the user gets the flag

This means that our flag is somehow derived from `h` which is calculated in `wrong()`.

```python
def wrong():
    trust = windll.kernel32.GetModuleHandleW(None)
    ...
    spare = bytearray(string_at(trust + inhabitant, tropical))
    ...
    return hashlib.md5(spare).digest() 
```

All the variable names in `wrong()` are obfuscated, but using the functions that are called in it we can easily see that it gets a handle to the process's file in memory, does a few modifications to the data from memory, and finally returns the md5 of that data. Since the original `wopr` is a *PyInstaller* executable, the process's file it will operate on will be `wopr.exe`, and we'll not be able to modify the python code in order to debug or dump values during its execution. However, if we'll try to execute the decompiled code using a normal Python interpreter, the process's file that will be in use will be `python.exe` and will not get us to the desired outcome. What I did here was to analyze `wrong()` and understand what it's doing. It basically removed all *reloc* information from the `.code` section of `wopr.exe` and then calculated its md5. Then I decided to dump the `.code` section of the original `wopr.exe` from memory and make the decompiled Python code work with my dumped section instead of the in-memory one. This is how I managed to get the desired `h` value, and later the flag. My guess is that this is the intended method.

When I finished the challenge, I went ahead and read some write-ups (a benefit you get when not doing things on time) and I encountered a much easier [solution](https://r0hansh.github.io/posts/flare-on6#7---wopr). If you know how *PyInstaller* executables work, you know it will create a directory in `%temp%` for its resources and it will create a file named `__init__.py` that will be executed within the context of the *PyInstaller* binary, in our case, in the context of `wopr.exe`!

```terminal
PS C:\Users\user\AppData\Local\Temp\_MEI28322\this> gci

    Directory: C:\Users\sivan\AppData\Local\Temp\_MEI28322\this
Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-a----         4/9/2020   3:47 PM            256 key
-a----         4/9/2020   3:47 PM              0 __init__.py
```

From here you could have simply put a breakpoint in `wopr.exe` before it executes what's inside `__init__.py` and modify the code there so it'll dump you the right `h` value, which will be easier than going through the entire process I went through. I will without a doubt utilize this the next time I'll encounter a *PyInstaller* challenge.

# Don't be afraid to learn something new

Even when you feel you have a complete arsenal of tools at your disposal, don't hesitate to use something new! Take challenge 8 - `snake` as an example. This challenge is a `.nes` file which I've never encountered before.

```terminal
$ file snake.nes
snake.nes: NES ROM image (iNES): 1x16k PRG, 1x8k CHR [V-mirror]
```

Turns out it is a *Nintendo Entertainment System* ROM*.* I didn't have any tool in my arsenal that can help me face this challenge, so I turned to Google of course and ended up in a NESDEV forum that recommended `FCEUX` as the goto debugger for *NES* ROMs. It turned out to be a really powerful debugger, and it helped tremendously in solving this task.

![FCEUX Debugger executing snake](/assets/img/2020-11-04-flareon-2019-lessons/snake-debugger.png)
*FCEUX Debugger executing snake*

# Read the write-ups

Solving a challenge can teach you a lot and be extremely satisfying, but only solving does not fulfill the entire learning potential. Do make sure you read through write-ups from different people. You'll probably find some details you might have missed, you'll find out the most effective methods of approaching it, and you may encounter some new tools/techniques that will be able to help you in similar challenges in the future. Don't miss out on the last bit of learning you can get from a challenge!

A good example of that can be the 11th challenge - `vv_max.exe`. This challenge contained an AVX VM, that performed base64 decoding on the input and compared it to a constant value. What I personally did was to write a disassembler for that VM, analyze what it's doing, understand the base64 logic and then simply calculate the correct input. Here are some of the other methods I've seen on different write-ups:

- Patching the original binary so that it'll expose the final result (no need for writing a disassembler)
- Using a theorem prover ([Z3](https://github.com/Z3Prover/z3)) to get the correct input after writing a disassembler
- Manually reversing the calculation to get to the correct input
- Brute-forcing
- Compiling the VM into an executable and using symbolic execution ([angr](https://angr.io/))

So as you can see, there is more than one way to peel a potato, and the more of them you know, the easier for you it'll be to solve challenges in the future.

# Testing your hypothesis

When you get more and more experience in solving challenges, you'll find yourself making educated guesses of what's the solution, before you actually know it for a fact. In that case (and if circumstances allow you) I would encourage you to test your hypothesis! Especially if testing it is actually shorter than really understanding whether it's the right solution or not.

I encountered that scenario solving the challenge mentioned above - `vv_max.exe`. Pretty quickly after writing the disassembler, I did a bit of Googling about usages of AVX operations and stumbled upon a paper describing how to perform base64 encoding in an efficient way using those operations. I thought to myself "huh, this could be what I'm seeing here, but I need to know more to be sure". So I continued to analyze the VM code. I was so into analyzing and understanding the VM code that I forgot my first assumption and continued to ignore the signs showing me that it was indeed base64 decoding. Only after a few more hours, did I really understand (with no doubt) that I was looking at base64 encoding. At that point, I remembered my initial assumption and became a bit disappointed that I had not tested it right then and there. It would have saved me a few precious hours, but we're in this for the long run.

# E**t tu Brute(-force)?**

As much as I personally hate using brute-force for solving challenges, it wouldn't be smart to completely avoid that.

An easy example for that can be challenge 10 - `MugatuWare`. This challenge was fantastic; from using common malware techniques like patching the IAT, to using esoteric Windows IPC mechanism like *[Mailslots](https://docs.microsoft.com/en-us/windows/win32/ipc/mailslots),* and finally implementing a bug seen in a real-world ransomware sample. 

Long story short, what you had to do in order to solve this challenge was to decrypt a *.gif* image that the provided ransomware sample has encrypted. After analyzing the ransomware binary to find that the encryption algorithm it uses to encrypt files was a modified version of [`XTEA`](https://en.wikipedia.org/wiki/XTEA), you should have noticed that the author made a crucial mistake, and only used a 4-byte key for the encryption (as opposed to the 16-byte key in the original implementation). This calls for brute-force. Not only that, but the challenge authors were kind enough to provide you with a hint - the first byte of the key. No doubt the intended and the best solution here was to use brute-force.

![Brute-force hint](/assets/img/2020-11-04-flareon-2019-lessons/the_key_to_success_0000.gif)

*Brute-force hint*

But brute-force is not only useful when it's the intended solution. We can take a look at challenge 3 -`FlareBear` as an example. In the challenge, you had an android app in which you can create a pet bear. This bear has 3 attributes - `mass`, `happy` and `clean`. If the following condition applies, the bear will get "ecstatic" and give you the flag.

```java
mass == 72 && happy == 30 && clean == 0
```

The values are affected by your interactions with the bear, you can feed it, play with it and clean it. The straightforward way of solving this is to understand the logic of the program; how much does every interaction change every attribute, and calculate the exact actions you need to perform to bring the bear to an ecstatic mode. 

Many times, if you don't want to invest your time in understanding the program (when the situation allows) you can simply brute-force it. In this example, that means copying all the application's code and programmatically interacting with the bear until it hands you the flag.

![Ecstatic Flare bear](/assets/img/2020-11-04-flareon-2019-lessons/flarebear.png)

*Ecstatic Flare bear*

# Stick to the script

You **must** master at least one scripting language to be a successful CTF player. Almost every CTF task will require you to do one of the following:

- Manipulate Data
- Automate a procedure
- Reimplement some logic
- Solve an equation of some sort

In order to do those things effectively, you need to take advantage of a scripting language. This doesn't necessarily have to be an actual scripting language per se, but rather **any** language that you feel comfortable with and that allows you to perform the stuff mentioned above without much effort.

```terminal
[0x00000000]> fo
    ╭──╮    ╭───────────────────────────────╮
    │ _│    │                               |
    │ O O  <  I script in C, because I can. │
    │  │╭   │                               │
    ││ ││   ╰───────────────────────────────╯
    │└─┘│
    ╰───╯
```

Now when it comes to which language to choose, it mostly boils down to personal preference. My go-to (and many others' as well) is Python. Every language will have its pros and cons of course. Some of the pros of Python for instance is that it is rather easy to learn, has a nice syntax and many modules developed specifically for solving CTF tasks. One common con is that if you work with data and convert between `int` and `unsigned int` types a lot, you'll have to pay close attention to what you're doing.

This is the most basic skill. It was called for even in the first stage of the first Flare-On challenge, where the desired input was an array of bytes XORed with a fixed value. This is an easy one-liner in Python:

```terminal
>>> ''.join([chr(ord(c)^ord('A')) for c in [ '\u0003',' ','&','$','-','\u001e','\u0002',' ','/','/','.','/']])
​
'Bagel_Cannon'
```

The more comfortable you'll feel with your scripting abilities, the more efficient CTF player you'll be, being able to translate your thoughts into actions.

# There is at least one solution

Getting stuck can be unpleasant. Nevertheless, always remember that as opposed to real-life problems, **a CTF challenge always has at least one solution!** So take a step back, try a different approach and don't get discouraged.

# Put in the time

If you're trying to get serious about CTFs, your mindset should be *"This is not a CTF"*. 

What if it was a real-world problem? You'll try every possible way you can think of to solve this problem. Trying to solve a CTF challenge shouldn't be different. Try to get rid of thoughts like *"Can I solve this?"* and try to think more about *"How can I solve this?"*.

# Be prepared

This shouldn't come as a surprise to you, but being prepared makes a world of a difference. The tools you use matter if you want to do things efficiently. With time, you'll collect your own little arsenal of tools you like to use. It's useful to have a few options for each purpose, so you can utilize the strengths of each tool. For example, having *IDA Pro/Ghidra/Cutter* to analyze the code and using *r2* for quick and dirty automation you need on the fly.

In addition to tools, you can also develop your own helper scripts that will make your work quicker and more comfortable. A great example of this can be seen on Gynvael Coldwind's GitHub (Gynvael is the retired captain of one of the best CTF teams in the world - Dragon Sector) where you can find the [skeleton script](https://github.com/gynvael/random-stuff/blob/master/pwnbase/pwnbase.py) for exploits he uses during CTFs. Gynvael also hosts a weekly [stream on YouTube](https://www.youtube.com/user/GynvaelEN) where he likes to solve CTF challenges quite often.

# Write you own challenges

The same way that a good malware analyst needs to know how to write malware, and an incident responder needs to know how attackers operate in networks, you should experiment with writing your own challenge. Whether it is to volunteer and contribute to a CTF organized in your community, to test out your friends, or event just for your pleasure. It is extremely helpful to know the "other-side" of CTFs. It can help you "get into the author's mind" when you're solving challenges and you can understand how you should solve a challenge by thinking about how it was constructed.

![](/assets/img/2020-11-04-flareon-2019-lessons/Untitled.png)

Finally, don't forget to enjoy yourself when playing CTFs! 🤓