---
layout: post
title: 'Fun with Assembly 14: The heap'
date: 2015-08-31
status: publish
type: post
published: true
author: Hywel Carver
---
This is the fourteenth in a series. You might want to read the [previous post]({% post_url 2015-08-30-fun-with-assembly-13 %}) before reading this.

This post is based on the Algiers level on [microcorruption.com](http://microcorruption.com). Like last time, we’re trying to find an input to open a lock without knowing the correct password, using knowledge of assembly language.

# First steps

There are a couple of interesting things happening here:

 - The username and password aren't being copied to the stack, so we probably won't be able to overflow the stack to write overflow the return address.
 - This code uses dynamically allocated memory on the heap instead of statically allocated memory on the stack.

As you'll remember, the stack grows downwards. In each function call, the stack gets extended by a fixed amount, then at the end of the function, the stack is shortened again.

The heap grows upwards. It's especially useful for allocating memory dynamically (i.e. where the size depends on input, so you can't know the size that'll be required at the time of writing the code).

That's done with the two functions `malloc(unsigned int bytes)` (to allocate more space, which returns the address of the allocated memory) and `free(byte*)` which frees the allocated memory at the address given.

### Malloc

This is a long function. But it's important, so let's dive in. It's pretty complicated, so the easiest thing to do first is to convert it into something that resembles C or C++. 

We know the function is going to return a pointer to memory, and takes a number of bytes to allocate.

    byte* malloc(unsigned int r15){
    4466:  c293 0424      tst.b &0x2404
    446a:  0f24           jz    #0x448a <malloc+0x26>
    446c:  1e42 0024      mov   &0x2400, r14
    4470:  8e4e 0000      mov   r14, 0x0(r14)
    4474:  8e4e 0200      mov   r14, 0x2(r14)
    4478:  1d42 0224      mov   &0x2402, r13
    447c:  3d50 faff      add   #0xfffa, r13
    4480:  0d5d           add   r13, r13
    4482:  8e4d 0400      mov   r13, 0x4(r14)
    4486:  c243 0424      mov.b #0x0, &0x2404
    448a:  1b42 0024      mov   &0x2400, r11
    448e:  0e4b           mov   r11, r14
    4490:  1d4e 0400      mov   0x4(r14), r13
    4494:  1db3           bit   #0x1, r13
    4496:  2820           jnz   #0x44e8 <malloc+0x84>
    4498:  0c4d           mov   r13, r12
    449a:  12c3           clrc
    449c:  0c10           rrc   r12
    449e:  0c9f           cmp   r15, r12
    44a0:  2338           jl    #0x44e8 <malloc+0x84>
    44a2:  0b4f           mov   r15, r11
    44a4:  3b50 0600      add   #0x6, r11
    44a8:  0c9b           cmp   r11, r12
    44aa:  042c           jc    #0x44b4 <malloc+0x50>
    44ac:  1dd3           bis   #0x1, r13
    44ae:  8e4d 0400      mov   r13, 0x4(r14)
    44b2:  163c           jmp   #0x44e0 <malloc+0x7c>
    44b4:  0d4f           mov   r15, r13
    44b6:  0d5d           add   r13, r13
    44b8:  1dd3           bis   #0x1, r13
    44ba:  8e4d 0400      mov   r13, 0x4(r14)
    44be:  0d4e           mov   r14, r13
    44c0:  3d50 0600      add   #0x6, r13
    44c4:  0d5f           add   r15, r13
    44c6:  8d4e 0000      mov   r14, 0x0(r13)
    44ca:  9d4e 0200 0200 mov   0x2(r14), 0x2(r13)
    44d0:  0c8f           sub   r15, r12
    44d2:  3c50 faff      add   #0xfffa, r12
    44d6:  0c5c           add   r12, r12
    44d8:  8d4c 0400      mov   r12, 0x4(r13)
    44dc:  8e4d 0200      mov   r13, 0x2(r14)
    44e0:  0f4e           mov   r14, r15
    44e2:  3f50 0600      add   #0x6, r15
    44e6:  0e3c           jmp   #0x4504 <malloc+0xa0>
    44e8:  0d4e           mov   r14, r13
    44ea:  1e4e 0200      mov   0x2(r14), r14
    44ee:  0e9d           cmp   r13, r14
    44f0:  0228           jnc   #0x44f6 <malloc+0x92>
    44f2:  0e9b           cmp   r11, r14
    44f4:  cd23           jne   #0x4490 <malloc+0x2c>
    44f6:  3f40 4a44      mov   #0x444a "Heap exausted; aborting.", r15
    44fa:  b012 1a47      call  #0x471a <puts>
    44fe:  3040 4044      br    #0x4440 <__stop_progExec__>
    4502:  0f43           clr   r15
    }

Also:

 - Jumping to an address is probably an `if`.
 - If the jumped-to address holds an instruction that follows another jump, that's probably an `if ... else` (because both paths will never be executed together).
 - A jump near the end of the function back to a point earlier in the function is probably a loop.
 - Conditions for going back to the beginning of the loop, or for jumping to the address after the loop translate to `break`s.
 - Jumps to the very end of the function are `return`s.

<!-- comment to force next section to render as code -->

    byte* malloc(unsigned int r15){
      if (*((byte*) 0x2404) != 0){
    446c:  1e42 0024      mov   &0x2400, r14
    4470:  8e4e 0000      mov   r14, 0x0(r14)
    4474:  8e4e 0200      mov   r14, 0x2(r14)
    4478:  1d42 0224      mov   &0x2402, r13
    447c:  3d50 faff      add   #0xfffa, r13
    4480:  0d5d           add   r13, r13
    4482:  8e4d 0400      mov   r13, 0x4(r14)
    4486:  c243 0424      mov.b #0x0, &0x2404
      }
      r14 = r11 = *0x2400;
      while(true){
        r13 = *(r14 + 4);
        r12 = r13 / 2;
        if ((r13 % 2 == 0) && r15 >= r12){
    44a2:  0b4f           mov   r15, r11
    44a4:  3b50 0600      add   #0x6, r11
      if(r11 >= r12){
    44ac:  1dd3           bis   #0x1, r13
    44ae:  8e4d 0400      mov   r13, 0x4(r14)
      }
      else
      {
    44b4:  0d4f           mov   r15, r13
    44b6:  0d5d           add   r13, r13
    44b8:  1dd3           bis   #0x1, r13
    44ba:  8e4d 0400      mov   r13, 0x4(r14)
    44be:  0d4e           mov   r14, r13
    44c0:  3d50 0600      add   #0x6, r13
    44c4:  0d5f           add   r15, r13
    44c6:  8d4e 0000      mov   r14, 0x0(r13)
    44ca:  9d4e 0200 0200 mov   0x2(r14), 0x2(r13)
    44d0:  0c8f           sub   r15, r12
    44d2:  3c50 faff      add   #0xfffa, r12
    44d6:  0c5c           add   r12, r12
    44d8:  8d4c 0400      mov   r12, 0x4(r13)
    44dc:  8e4d 0200      mov   r13, 0x2(r14)
      }
        return r14 + 6
      }
    44e8:  0d4e           mov   r14, r13
    44ea:  1e4e 0200      mov   0x2(r14), r14
        if(r13 >= r14){
          break;
        }
        if(r11 == r14){
          break;
        }
      }
    44f6:  3f40 4a44      mov   #0x444a "Heap exausted; aborting.", r15
    44fa:  b012 1a47      call  #0x471a <puts>
    44fe:  3040 4044      br    #0x4440 <__stop_progExec__>
      return 0;
    }

Now let's convert the rest of the assembly, and rename some obvious variables. Judging by the last line before looping (`r14 = *(r14 + 2);`), the 3rd and 4th bytes of the 6 at the beginning of each chunk point to the next chunk, the 1st and 2nd point to the previous chunk. We can also see that the 5th and 6th bytes get compared with the numbers of bytes required. It seems likely that they are the size of the chunk available.

If you run the code, you can also see that `malloc` uses the first 4 bytes from `0x2400` onwards to store header information for the heap, then starts allocating from `0x2408` onwards.

With all of that in mind, we can pretty accurately reverse-engineer the function.

{% highlight C %}
    byte* malloc(unsigned int bytes_requested){
      int* heap_header  = (int*) 0x2400;
      int* heap_content = *((int**) heap_header[0]);
    
      // Some kind of initialization of the heap that's only going to be
      // done once.
      if(heap_header[2] != 0){
        // Store the address of the header in its own first 2 bytes.
        heap_content[0] = heap_content;
        heap_content[1] = heap_content;
        heap_content[2] = (heap_header[1] - 6) * 2;
        heap_header[2] = 0x0;
      }
      // Start at the first chunk.
      int* current_chunk = heap_content;
      // And then go round all the chunks
      while(true){
        int current_chunk_status_and_size = current_chunk[2];
        int current_size = current_chunk_status_and_size >> 1;
        // If the current chunk is at least as big as required, 
        // and not currently in use.
        if(current_size > bytes_requested || 
            (current_chunk_status_and_size & 0x1) != 0){
          // Need the requested bytes + 6 more for the metadata.
          int bytes_required = bytes_requested + 6;
          // If the required bytes is more than the current chunk size,
          if(bytes_required >= current_size){
            // just mark the current chunk as in use.
            current_chunk[2] = current_chunk_status_and_size | 1;
          }
          else{
            // Split the chunk. The current chunk is as big as is required.
            // Mark it as in use, too.
            int status_for_this_chunk = bytes_requested + bytes_requested;
            current_chunk[2] = status_for_this_chunk | 0x1;
            // Next chunk starts immediately after the metadata and data for this chunk.
            int* next_chunk = current_chunk + 6 + bytes_requested;
            // Next chunk's previous is the current chunk, and its
            // next is the current chunk's next.
            next_chunk[0] = current_chunk;
            next_chunk[1] = current_chunk[1];
            // Size the extra chunk, and calculate its status
            current_size = current_size - bytes_requested - 6;
            int new_chunk_status = current_size * 2;
            next_chunk[2] = new_chunk_status;
            // And the current chunk's next is the new next_chunk.
            current_chunk[1] = new_chunk;
          }
          // Return the address of the start of the data in this chunk.
          return current_chunk + 6;
        }
        // Move onto next chunk to try again.
        previous_chunk = current_chunk;
        current_chunk = current_chunk[1];
        // If the chunks are in a weird order, we've run out of chunks to try.
        if(previous_chunk >= current_chunk){
          break;
        }
        if(r11 == current_chunk){
          break;
        }
      }
      puts("Heap exausted; aborting.");
      exit(1);
      return 0;
    }
{% endhighlight %}

By reverse engineering the `malloc` code, we've learned roughly how it works. Each chunk created by `malloc` begins with a 6-byte header section, followed by a payload. The first pair of bytes point to the previous chunk, the second pair point to the next chunk and the last pair are the size of the chunk multiplied by 2, with the least signficiant bit set to 1 if the chunk is in use.

### Free

`free` is the function called to dynamically deallocate memory allocated by `malloc`. Let's try the same thing.

    4508 <free>
    4508:  0b12           push  r11
    450a:  3f50 faff      add   #0xfffa, r15
    450e:  1d4f 0400      mov   0x4(r15), r13
    4512:  3df0 feff      and   #0xfffe, r13
    4516:  8f4d 0400      mov   r13, 0x4(r15)
    451a:  2e4f           mov   @r15, r14
    451c:  1c4e 0400      mov   0x4(r14), r12
    4520:  1cb3           bit   #0x1, r12
    4522:  0d20           jnz   #0x453e <free+0x36>
    4524:  3c50 0600      add   #0x6, r12
    4528:  0c5d           add   r13, r12
    452a:  8e4c 0400      mov   r12, 0x4(r14)
    452e:  9e4f 0200 0200 mov   0x2(r15), 0x2(r14)
    4534:  1d4f 0200      mov   0x2(r15), r13
    4538:  8d4e 0000      mov   r14, 0x0(r13)
    453c:  2f4f           mov   @r15, r15
    453e:  1e4f 0200      mov   0x2(r15), r14
    4542:  1d4e 0400      mov   0x4(r14), r13
    4546:  1db3           bit   #0x1, r13
    4548:  0b20           jnz   #0x4560 <free+0x58>
    454a:  1d5f 0400      add   0x4(r15), r13
    454e:  3d50 0600      add   #0x6, r13
    4552:  8f4d 0400      mov   r13, 0x4(r15)
    4556:  9f4e 0200 0200 mov   0x2(r14), 0x2(r15)
    455c:  8e4f 0000      mov   r15, 0x0(r14)
    4560:  3b41           pop   r11
    4562:  3041           ret

Taking the same approach of reverse engineering this function, we get this (I've changed the byte operations to integer operations where it made more sense):

{% highlight C %}
    void free(byte* address){
      int* metadata_location = address - 6;
      // Mark the chunk as being out of use (i.e. least-significant bit not set)
      int size = metadata_location[2] & 0xfffe;
      metadata_location[2] = size;
    
      int* previous_chunk = *((unsigned int*)metadata_location);
      int previous_chunk_size_and_status = previous_chunk[2];
    
      // If the previous chunk is not in use, combine it with this one.
      if((previous_chunk_size_and_status & 0x1) == 0){
        // Add this chunk's size to the previous chunk
        previous_chunk_size_and_status += 6 + size;
        previous_chunk[2] = previous_chunk_size_and_status;
    
        // Change the previous chunk's "next" pointer to point to this chunk's "next"
        previous_chunk[1] = metadata_location[1];
    
        // Change the next chunk's "previous" pointer to point to the previous chunk.
        *(metadata_location[1]) = previous_chunk;
    
        // This chunk's metadata_location is now the previous chunk's.
        metadata_location = metadata_location[0];
      }
    
      int* next_chunk = metadata_location[1];
      int next_chunk_size_and_status = next_chunk[2];
    
      // If the next chunk is not in use, combine it with this one.
      if((next_chunk_size_and_status & 0x1) == 0){
        // Add the next chunk's size to this chunk's
        metadata_location[2] += next_chunk_size_and_status + 6;
        // Make this chunk's "next" pointer point to the next chunk's "next"
        metadata_location[1] = next_chunk[1];
        // Make the next chunk's "previous" pointer point to this chunk.
        *((int*)metadata_location[1]) = metadata_location;
      }
    }
{% endhighlight %}

So, `free` does the opposite of `malloc`. When a chunk is freed, `free` types to combine with neighbouring chunks that are not currently in use, and fixes pointers in the metadata.

### The exploit

`malloc` and `free` must somehow be useful to open the door. Let's look at how they're used in the login function.

    463a <login>
    463a:  0b12           push  r11
    463c:  0a12           push  r10
    463e:  3f40 1000      mov   #0x10, r15
    4642:  b012 6444      call  #0x4464 <malloc>
    4646:  0a4f           mov   r15, r10
    4648:  3f40 1000      mov   #0x10, r15
    464c:  b012 6444      call  #0x4464 <malloc>
    4650:  0b4f           mov   r15, r11

Allocate 16 bytes for each, and store the address in `r10` and `r11`.

    4652:  3f40 9a45      mov   #0x459a, r15
    4656:  b012 1a47      call  #0x471a <puts>
    465a:  3f40 c845      mov   #0x45c8, r15
    465e:  b012 1a47      call  #0x471a <puts>
    4662:  3e40 3000      mov   #0x30, r14
    4666:  0f4a           mov   r10, r15
    4668:  b012 0a47      call  #0x470a <getsn>

Print out some stuff, then get 48 bytes into the first buffer. Then do the same for the second.

    466c:  3f40 c845      mov   #0x45c8, r15
    4670:  b012 1a47      call  #0x471a <puts>
    4674:  3f40 d445      mov   #0x45d4, r15
    4678:  b012 1a47      call  #0x471a <puts>
    467c:  3e40 3000      mov   #0x30, r14
    4680:  0f4b           mov   r11, r15
    4682:  b012 0a47      call  #0x470a <getsn>

Later on the password is freed, then the username. However, because we're able to write more bytes (48) than were allocated (16), we can overwrite some of the metadata on the heap.

Currently the start of the heapspace looks like this:

    2408: <6 bytes username chunk metadata>
    240e: <16 bytes for the username>
    2410: <2 bytes of password chunk metadata, pointing to username chunk>
    2412: <2 bytes of password chunk metadata, pointing to next chunk>
    2414: <2 bytes of password chunk metadata: status>
    2416: <16 bytes for the password>

So how can we manipulate the password chunk's metadata? Well, when this chunk is `free`d, its previous chunk is in use, so all that will happen is that its status flag will be set to 0. However, if we manipulate the metadata to point towards somewhere else, we can change the memory there.

The simplest way would be to make the password chunk's "previous" block appear to be in the area where the `login` function's return address is stored. That way, `free` will change the bytes that the function returns to.

The return address is kept at `0x439a`. If we're going to treat those as the two size & status bytes of the metadata, the block would start at `0x4396`. Let's make that the "previous" chunk to the password chunk. We'll point the "next" chunk to be `0x4400` so that it doesn't interfere with anything else. Now, how do we want the status byte to be changed?

Currently the return address is `0x4440`, but we want it to be `0x4564`. `free` will change the previous block's status by adding the password block's size + 6. So that means we'll need a size of `0x11e` (because `0x11e` + `0x6` + `0x4440` = `0x4564`). 

Putting that all together, we need a username of 16 bytes of `0x41` (the letter A) , then `9643`, `0044` and `1e01`, to overwrite the return address with the right content to unlock the door.

`41414141414141414141414141414141964300441e01`

# The door springs open

The complexity is really ramping up now. This exploit was a multiple step process, involving understanding how the memory was dynamically allocated, then exploiting a buffer overflow in a way that would interfere with the deallocation mechanism to rewrite a return address. Sneaky.