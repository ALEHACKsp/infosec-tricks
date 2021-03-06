# Remote Server Time

This page lists a bunch of ways you can get the current date/time of a remote server which you can reach over the network. Some are more accurate than others.

## Why?

That question can be interpreted in one of two ways.

**Why would I want to know the time on a remote server?**

There are a lot of reasons why. You might be analysing a malware C&C box and want some idea of what timezone or region it's in. You might have found a vulnerability where a key cryptographic value is derived solely from the current timestamp. You might just be curious.

**Why are you documenting all of these approaches?**

This started out as [a Twitter thread](https://twitter.com/hedgeberg/status/992121613909942272) and I realised that many of the tricks I know about aren't well-known or documented. So I thought I'd document them.

## Requirements

This page expects that you have already enumerated services on the remote system, using a tool such as nmap. Based on the available services you can then attempt the options here. Remember that services are not always on standard ports.

A few useful enumeration commands are as follows:

**Full TCP SYN scan with service detection:**

```
nmap -sS -Pn -A -T4 --p1-65535 [host]
```

**UDP top ports scan with service detection:**

```
nmap -sU -Pn -A -T4 --top-ports 1000 [host]
```

This scans the top 1000 most common UDP ports. The reason for using the `--top-ports` approach here is that UDP scanning is incredibly slow, due to the stateless nature of the protocol. You can switch out `--top-ports 1000` for the usual `-p1-65535`, but be aware that it will probably take many hours to complete the scan. You may wish to set the `--min-rate` option too if nmap's adaptive retransmissions give you trouble.

## Approaches

### HTTP Date Header

Many HTTP servers return a `Date` header in response to requests. For example:

```
GET / HTTP/1.1
Server: example.com

HTTP/1.1 200 OK
Content-Type: text/html
Content-Length: 123
Connection: close
Date: Fri, 04 May 2018 22:28:58 GMT
```

This is probably the simplest way. You can use the network inspector in your browser (F12 in most of them) to see the response headers if you don't want to send the request manually.

### ICMP Timestamp

This is almost never enabled but you can always hope.

The following Linux command sends an ICMP timestamp request and returns the result, if any:

```
ping -T tsonly [host]
```

If you get a reply, the number is an integer that expresses the time since UTC midnight in milliseconds. 

Some servers respond with garbage values so do not trust this too much.

### TLS ServerRandom Field

This is probably the second most reliable approach after the HTTP Date Header. Any service that supports SSL/TLS should support this trick, including those which use STARTTLS. For example, you can use it on the following protocols:

* HTTPS
* FTPS
* POP3 (either POP3S explicitly or with STLS)
* IMAPS
* MySQL, MSSQL, PostgreSQL, etc. if connection encryption is enabled
* LDAPS
* IRC with secure ports (usually TCP 6697)
* NNTPS

When the TLS connection is made, the client sends a ClientHello packet, to which the server responds with a ServerHello packet. The ServerHello packet contains a field called ServerRandom, which is an opaque 32 byte value generated by the server. In many cases, however, the first 4 bytes of this field are in fact the 32-bit UNIX timestamp, i.e. the number of seconds since January 1st, 1970, 00:00:00 UTC. The rationale behind this is that a monotonically increasing value removes the potential for collisions between ServerRandom values. Wireshark will display this timestamp value for you when it dissects a TLS ServerHello packet.

Note that more recent implementations of TLS servers offset this value by a random amount, such that the value continues to increment monotonically but does not bear any resemblance to the real time - it can be as much as 80 years out. This was implemented as an explicit defence against timestamp fingerprinting.

For protocols that are wrapped in TLS by default (e.g. HTTPS, FTPS), or protocols that support non-TLS and TLS connections on the same port (e.g. MySQL, PostgreSQL), you don't need to use a client specific to that protocol. Instead you can use the following command to open an SSL/TLS connection and ignore the underlying protocol:

```
openssl s_client -connect [host]:[port]
```

You can then monitor this connection in Wireshark and see the timestamp.

For protocols that use STARTTLS, you'll need to trigger the starting condition. Usually this means using a client specific to the protocol and enabling STARTTLS support.

### SMTP

When connecting to an SMTP server the initial `220` message sent by the server often has the date and time in it:

```
220 localhost.localdomain ESMTP Sendmail 8.13.6/8.13.6; Sat, 5 Apr 2018 00:32:24 +0100
```

If you see SMTP open this is a good and simple option you can try via `telnet` or `nc`.

### SMB Negotiate

When you try to authenticate to an SMB server, the Negotiate Protocol Response packet contains a timestamp. The following Wireshark filter will show you these packets:

```
smb2.cmd == 0 && smb2.flags.response == 1
```

The packet can also sometimes tell you the boot time of the server, although this field is optional.

If the system is really old and only supports SMB1 you'll likely need to enable that explicitly on your side (it's usually disabled for security) and modify the above Wireshark filter to use `smb` rather than `smb2`.

### NTP

On the off-chance that the box is running an NTP server (usually UDP port 123) you can try to get a timestamp from it that way. The `ntpq` command is useful here:

```
ntpq -pn [host]
```

It's worth noting that there are some NTP server configurations that send NTP traffic to broadcast and multicast addresses on the local subnet. If you're on the same network as the server you should check Wireshark to see if there's any NTP traffic flying around.

### Finger

Once we stop laughing at the funny name and the fact that someone actually left it enabled (in 2018, really!?) we can sometimes use it to see or guess the current server time.

The following command will return the list of users that are logged on:

```
finger @[host]
```

From here we can query a specific user:

```
finger [user]@[host]
```

This usually returns the time that the user logged in, and how long they have been idle. This gives you a rough idea of the time to within a few hours.