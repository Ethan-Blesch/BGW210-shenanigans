# Reversing the BGW210-700 and unlocking the bootloader
When I started looking for a winter break project, I was told that wifi router firmware was extremely easy to find your first "real" bug in- which turned out to be excellent advice. Not knowing anything about picking a target, I looked at the model number on the back of my router, and found a [firmware dump](https://archive.org/details/BGW210-700-Firmware-Collection). Checking it out with `binwalk`, there was a bunch of licensing and certificate data, along with a squashfs filesystem.

## The SDB daemon
From what I found, the SDB daemon, contained in the `/bin/sdb`, `/bin/sdb_helper`, and `/lib/libsdb.so` binaries, is the central piece of software on the system. Although it's not really relevant to the rest of this writeup, I'm going over it in case other people end up working with it (and because I wasted a lot of time reversing it). It handles logging, manages a database, handles commands sent from the web portal, and does what looks like a good portion of the initialization stuff after boot. It creates a linux socket at /var/tmp/sdb.server and listens for requests using its own custom protocol. Either emulating or writing a stub for it is basically mandatory if you plan to emulate something from the firmware dump, since even if a binary doesn't rely on its database or commands, it will likely still use it for logging and refuse to run if it can't connect to the socket. 

## The webpages
The webpages are served via cgi-bin using `lighttpd`, with a custom binary called `/bin/webasp` generating the HTML from some weird templated Lua. Confusingly, there's actually code for three separate web interfaces in the `/www` directory, although `/www/att` is the only one served by default. 
Inside of `/www/wificert`, one of the 2 web portals not enabled by default, however, there's something <i>very</i> interesting. 
```lua
if ENV.REQUEST_METHOD == "POST" then
  if FORM.Cancel ~= nil then
    if FORM.uploadfile ~= nil and FORM.uploadfile ~= "" then
        os.execute("rm -f "..FORM.uploadfile)
```
Even though I couldn't use it for my initial access, this is your run-of-the-mill command injection, and it seemed like it could be an easy way to set up persistence, although sadly it never ended up coming into play. (I think I can still count this as "found a command injection bug" though?)

## UART and the BIOS
The BGW210 has some pins on the board which allow for direct access over a serial cable. So, after ordering a guinea pig BGW210 and a UART to USB adapter, I was able to solder the right pins and only burnt myself once. [this writeup](https://gist.github.com/quonic/fab50d7e0979ab21675150a15957cb66) explains the disassembly pretty thoroughly, although the exploit it describes is no longer functioning, and you can't downgrade anymore either.<br><img width="604" height="532" alt="image" src="https://github.com/user-attachments/assets/cdbf8381-3391-4dde-ba9a-01269e6ee67b" /><br>^ my abysmal soldering job :/ <br><br>
My final setup was a cheap UART to USB adapter from temu connected to PuTTY (baud rate is 115200, and flow control needs to be set to `None`), and the USB drive mentioned in the writeup is unnecessary. After plugging in the power to the router, plugging in the UART adapter to my PC, and opening a serial connection, I was able to see the console output of the router booting.
![](https://github.com/user-attachments/assets/aa2fa3f4-0fe2-4c4c-af75-92668b86d594)<b>

## Finding the password
<b> Spoiler alert- this section's gonna be short. </b>
After extracting the UBoot image from the .bin file, I opened up Ghidra, and while I was waiting for it to start, I tried `strings uboot.bin | grep password`. Maybe that'll give us some function names or error strings or...
```bash
$ strings uboot.bin | grep password
Enter password ( attempting autoboot in %2ds )
password12345
```
![no fucking way.png](https://i.imgur.com/xZWVFNu.png) <br>
Surely there's no way that's it, right? <br> <b>Right? <br>
wrong.</b><br>
## The Grave Mistake
Unfortunately, my brave, brave lab-rat BGW210 from ebay has met with a tragic fate and is now in a vegetative state. After typing in `password12345` at the login prompt during boot, I was dropped into the BIOS, which had, among other useful things like an option to boot from usb, commands for mounting, reading, and writing various filesystems & flash chips. Importantly, U-boot has you read files & other data directly to an address in physical memory, and then directly print out said physical memory. I found an interesting-looking file to read on one of the mountable filesystems, pasted in what I thought was some unused memory I could put it in for the address, and hit enter. And then it stopped responding to my inputs over PuTTY. <br><br> 
![oh fuck.jpg](https://i.giphy.com/18gEqArCQNlo5zowZn.webp) <br><br>
It turns out, due to Virtualbox's janky shared clipboard, what Windows thought was on my clipboard wasn't the scratch memory I'd copied inside virtualbox- it was the memory-mapped address of the flash chip containing the bootloader- specifically, the address of the [interrupt vector table](https://en.wikipedia.org/wiki/Interrupt_vector_table). Fuuuuuuuuck. 

## Conclusions 
- Using the boot from USB functionality in the bootloader, the BGW210 can boot or install custom firmware like OpenWRT, which is pretty cool.
- If someone more careful than me used the available tools inside U-Boot, they could boot or install a patched or outdated image. This can be used to get a shell on the device and extract certificates, do further reversing, and generally get up to shenanigans and/or mischeif.    
- Use EXTREME caution when dealing with U-Boot- you *will* pay dearly
<br>
<img width="480" height="529" alt="image" src="https://github.com/user-attachments/assets/3763c371-1cee-4bf0-8d22-342672c3118d" />
