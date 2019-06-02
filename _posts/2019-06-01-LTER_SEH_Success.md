---
layout: single
title: CTP/OSCE Prep -- A Noob's Approach to Alphanumeric Shellcode (LTER SEH Overwrite)
date: 2019-6-01
classes: wide
header:
  teaser: /assets/images/CTP/immunity.jpg
tags:
  - buffer overflow
  - Windows
  - x86
  - shellcoding
  - exploit development
  - assembly
  - python
  - OSCE
  - CTP
  - SEH
--- 
![](/assets/images/CTP/1920x1080_Wallpaper.jpg)

## Introduction

This series of posts will focus on the concepts I'm learning/practicing in preparation for [CTP/OSCE](https://www.offensive-security.com/information-security-training/cracking-the-perimeter/). In this series of posts, I plan on exploring:
+ fuzzing,
+ vanilla EIP overwrite,
+ SEH overwrite, and
+ egghunters.

Writing these entries will force me to become intimately familiar with these topics, and hopefully you can get something out of them as well! 

In this particular post, we will be approaching an overflow in the `LTER` parameter trying to utilize all the tricks we've learned thus far. 

If you have not already done so, please read some of the posts in the 'CTP/OSCE Prep' series as this post will be **light** on review! 

## Goals

For this post, our goal is to walk through the right way to do the SEH overwrite exploit to `LTER` on Vulnserver and learn a new technique for encoding shellcode. 

## Doyler (@doylersec) Shoutout

Just want to take a second and shoutout @doylersec for all of his help with this particular exploit. You should probably read his [blog post](https://www.doyler.net/security-not-included/vulnserver-lter-seh) on this exploit before anything else. It was a very clever solution and he was extremely charitable explaining it to me. 

On a sidenote, he's also partially responsible for me getting a few certifications. The content on his blog led me down several challenging and rewarding paths and I don't think it's an exaggeration to say that I wouldn't be where I am today without his content. 

*The content you publish for others to learn is important and can have a huge impact!*

## Alphanumeric Shellcode

As we discovered in the [previous post](https://h0mbre.github.io/LTER_SEH_Exploit/), this particular command, `LTER`, is filtering for alphanumeric shellcode. To reiterate, that restricts us to the following characters: 
```terminal_session
\x01\x02\x03\x04\x05\x06\x07\x08\x09\x0b\x0c\x0e\x0f\x10\x11\x12\x13\x14\x15\x16\x17\x18\x19\x1a\x1b\x1c\x1d\x1e\x1f\x20\x21\x22\x23\x24\x25\x26\x27\x28\x29\x2a\x2b\x2c\x2d\x2e\x30\x31\x32\x33\x34\x35\x36\x37\x38\x39\x3b\x3c\x3d\x3e\x41\x42\x43\x44\x45\x46\x47\x48\x49\x4a\x4b\x4c\x4d\x4e\x4f\x50\x51\x52\x53\x54\x55\x56\x57\x58\x59\x5a\x5b\x5c\x5d\x5e\x5f\x60\x61\x62\x63\x64\x65\x66\x67\x68\x69\x6a\x6b\x6c\x6d\x6e\x6f\x70\x71\x72\x73\x74\x75\x76\x77\x78\x79\x7a\x7b\x7c\x7d\x7e\x7f
```

One way to overcome this limitation is to 'sub encode' your shellcode. As VelloSec explains in [CARVING SHELLCODE USING RESTRICTIVE CHARACTER SETS](http://vellosec.net/2018/08/carving-shellcode-using-restrictive-character-sets/), the manual process for sub encoding your payloads can be very tedious. I really recommend you read the VelloSec blog post. I probably had to read it through 6 times today. 

### Wrap Around Concept
One thing you need to know is, if you subtract your 4 byte payload from `0`, the value will wrap around. Let's use the Windows calculator to show this. To make things simple let's use a forbidden character of `\xf7` and show how we could get that somewhere on the stack without ever using it via sub encoding. 
1. First, we subtract `f7` from `0`. 

![](/assets/images/CTP/calc3.JPG)

2. We end up with `FFFF FFFF FFFF FF09`. We can ignore the proceeding `f` chars. 
3. Now that we have our value `09`, we need to manipulate it so that it ends up equaling `f7` without us ever using a forbidden character. 
4. Our next job is come up with 3 numbers that added together will equal our `09`. We'll use `04`, `03`, and `02`.
5. If we then use **three** `SUB` instructions, we can reach our original `f7` value. 
6. `0` - `4` = `FFFF FFFF FFFF FFFC‬`
7. `FFFF FFFF FFFF FFFC‬` - `3` = `FFFF FFFF FFFF FFF9‬`
8. `FFFF FFFF FFFF FFF9‬` - `2` = `FFFF FFFF FFFF FFF7`

As you can see, we ended up back at our `F7` without ever using it! That fundamental concept will be what we use throughout this exploit. 

### Automating Encoding 
At a high-level what we're going to accomplish with sub encoding and how we're going to use it in this exploit is: 
1. We're going to use `AND` operations to zero out the `EAX` register,
2. We're going to manipulate the `EAX` register with `SUB` and `ADD` instructions so that it eventually holds the value of our intended 4 byte payload,
3. We're going to push that value onto the stack so that `ESP` is pointing to it. 

As VelloSec put it lightly, manual encoding each 4 byte string can be tedious (especially if at some point you have to encode an entire reverse shell payload). Luckily, @ihack4falafel (Hashim Jawad) has created an amazing encoder called [Slink](https://github.com/ihack4falafel/Slink) for us to use. His encoder uses more `ADD` instructions but abuses the same wrap around concept. 

Let's show an example of how to use the tool with the test payload: `\xfe\xcf\xff\xe3`

![](/assets/images/CTP/test1.gif)

As you can see, the tool took almost no time at all to encode our payload. One thing to note, Slink only encodes 4 bytes at a time so if you submit a payload that's longer than 4 bytes make sure you grab **ALL** the output from Slink. A good thing to look out for is the `Shellcode final size:`.

### Using Encoded Payloads 
So now that we have our encoded payload, how do we get this to actually execute? As you can see, the final instruction in our encoded payload is always `\x50` or `push EAX`. This is going to place the value of `EAX` ontop of the stack and decrement `ESP` by 4 bytes. What does this mean? This means that wherever `ESP` is, when we go through our `SUB` instructions and push `EAX`, `ESP - 4` is going to be where our code is that we want to execute. To demonstrate, let's use some actual code we use in the exploit. 

Let's say we want to short-jump backwards (or negative) the maximum amount. The code to do this is `\xeb\x80`. Obviously we can't use `\xeb` or `\x80` as both of these bytes are not in our allowable range. Let's leverage Slink!

![](/assets/images/CTP/test2.gif)

Now we have our code:
```terminal_session
jump = ""
jump += "\x25\x4A\x4D\x4E\x55" ## and  eax, 0x554e4d4a
jump += "\x25\x35\x32\x31\x2A" ## and  eax, 0x2a313235
jump += "\x05\x76\x40\x50\x50" ## add  eax, 0x50504076
jump += "\x05\x75\x40\x40\x40" ## add  eax, 0x40404075
jump += "\x50"                 ## push eax 
```

Once we complete the `add  eax, 0x40404075` line, `EAX` will hold the value `909080EB`. This is dark magic.
Now when we `push eax`, `909080EB` will go into the 4 bytes "below" (really above visually, but below address wise as the stack grows in address size as it goes down) `ESP` and then `ESP` will be decremented by 4. 

So how can we use this? Well if we move our ESP to an advantageous spot before using our encoded shellcode, we could have execution finish our decoder, place our desired value onto the stack right below our decoder, and then control will pass to our desired value (shellcode). 

### Moving ESP Before Decoding
First, what is meant by 'decoding' in this context? Decoding happens when we push `EAX` onto the stack, this places our real code (the value held by `EAX`) right below `ESP`. 

So if our decoded shellcode is going to end up right below `ESP`, we need to know where that is and we need to move it to a location we want so that the program execution goes to it before going over other, potentially harmful instructions. 

To explain, here are some high-level diagrams. 

If we *DON'T* move `ESP`:

![](/assets/images/CTP/noESP.JPG)




## Resources
+ [OffSec Alphanumeric Shellcode](https://www.offensive-security.com/metasploit-unleashed/alphanumeric-shellcode/)
+ [Corelan Mona Tutorial](https://www.corelan.be/index.php/2011/07/14/mona-py-the-manual/)