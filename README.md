# Capturing ECN on MacOS

Since I'm stuck in homeoffice I was wondering how often connections would fail/succeed during the day when using ECN.
My router uses the CAKE AQM and performs ECN markings in both directions.

This script was developed/tested on MacOS Big Sur 11.2.3.

It basically tracks connections, ECN markings and TCP ECN state with the help of dtrace (see below) for incomming packets.
The rest is handled by a python script that just captures the output and accumulates all information for each connection and dumps it, as soon as the connection terminates, to stdout.

`capture_ecn` embedds both scripts and also forces ECN use for all connections. 
You need to run it with superuser permissions.
The script can also take some arguments, but the defaults should be fine.

## Dtrace

You need to enable dtrace capabilities by disabling the System Integrity Protection (SIP) for it.

1. Boot into recovery mode (CMD+R or (on M1) long press poewr button on boot until you can select Boot options)
2. Get a terminal
3. Disable SIP with: `csrutil disable`
4. Enable SIP but exclude dtrace: `csrutil enable --without dtrace`
5. Reboot


## Notes on Mac M1 / ARM / Apple Silicon

I've started writing this on an x86_64, since then, I got a new Macbook (M1) but it seems that dtrace is not yet up to speed.
For this tool, kernel trace points are missing and juding by the fact that my Mac crashes when using other, system-provided, dtrace scripts, things probably need some time.


## Discussion

### Testing ECN blackholing (forward path)

You can test dropping ECN SYNs in the forward direction using pf (for v4):

```
block out inet proto tcp from any to any flags SEW/SEW
```


### Testing ECN blackholing (return path)

I noticed this when I blocked (using pf and ipv4) incoming ECN enabled SYN/ACKs:

``` 
block in inet proto tcp from any to any flags SAE/SAE
``` 

Having this enabled causes the following packets (see `drop_ecn_incoming.pcap` (note: the SYN/ACK/ECN is in the capture as the firewall is executed after the pcap capture)):

```
Sender (MacOS)   ----- SYN/ECN/CWR  ---> Receiver (Linux)
Sender (MacOS)   <---- SYN/ACK/ECN  ---- Receiver (Linux)   (dropped)
...
Sender (MacOS)   --------- SYN  -------> Receiver (Linux)   (retransmising without ECN/CWR)
Sender (MacOS)   <---- SYN/ACK/ECN  ---- Receiver (Linux)   (dropped)
Sender (MacOS)   <---- SYN/ACK/ECN  ---- Receiver (Linux)   (dropped)
```

While MacOS retransmits its SYN without ECN, Linux seems to ignore this and simply retransmits its SYN/ACK keeping the ECN flag enabled (is this a feature or a bug?).
So if the ECN flag is the cause for the packet drop (as in this example), the connection will never establish.



### Tracking state on close vs on initialization

While the script tracks the ecn state on initalization and on connection close.
I don't know if this ever makes a difference. 
The field in the output are derived from the state transition to established (or connection failed).
For this reason, both the close and the initial full `ecn_flags` are provided in the output.
Having some fields derived was just to get a quick visual view of what happened.
See the script for a full list of bits that can be set for `ecn_flags` which is copied from `netinet/tcp_var.h`.


