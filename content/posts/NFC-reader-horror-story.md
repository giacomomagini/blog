---
title: "ACR122U NFC reader and e-documents: the horror story"
date: 2022-10-18T17:00:00Z
draft: false
tags: [nfc, e-documents, java]
---

## what I'm actually talking about

It's no secret that I work for a company that verifies identity online. What's identity? That's a big question, but most KYC processes involve having a genuine document.

Documents have evolved from paper to plastic, from being readable for humans to machines. The ultimate form of machine-readable documents are digital IDs and Driving Licences; everything is still kind of experimental.
The almost ultimate form of machine-readable documents is electronic documents (e-documents). They aren't experimental but very much part of our life, even though we sometimes ignore their existence.

This blog post will talk a bit about e-documents and the bad relationship with ACR122U NFC readers.

Are you working with e-documents and ACR122U NFC reader? You'll be interested.

Are you wondering why you're reading this? You'll be interested too.

Everyone loves horror stories! :)

## E-documents

E-documents are standard documents that embed an NFC chip containing the document's information in a digital format (name, face image, fingerprint, etc.).
This information is stored in the chip following a standard—the standard accounts for protecting them from tempering using well-known cryptographic techniques.

[ICAO](https://www.icao.int) is the institution maintaining the standards for e-documents.
Some more info on how they work can be found [here](https://www.icao.int/Security/FAL/PKD/BVRT/Pages/Introduction.aspx).

## JMRTD: a heroic java library

Dealing with cryptography, APDU commands and e-documents standards would be a major headache for any software engineer. A headache, long months of development, if not years.

JMRTD provide a great API to read data from e-documents, and it's up to date with the latest evolutions of the standard. You can read different documents using different standard versions with few lines of code.

[Here](https://github.com/tkaczenko/cardreader/blob/master/reader/src/main/java/com/github/tkaczenko/cardreader/service/DataConnection.java#L47) an example of Ukrainian passport reader using JMRTD.

## Testing PACE on my Italian ID

As part of the latest standards, there's one regarding the authentication phase between the chip and the system reading the chip. This new authentication mechanism is called PACE (Password Authenticated Connection Establishment). I won't go into details, but it's safer than the old BAC (Basic Access Control), and that's why new e-documents are adopting it. More info [here](https://www.icao.int/Security/FAL/PKD/BVRT/Pages/Document-readers.aspx).

JMRTD's API has implemented the whole PACE protocol, and it was just a matter of giving it a try with my new Italian ID.

### First attempt (nice try!)

The first step was writing a simple Java application using JRMTD's API, similar to the [code](https://github.com/tkaczenko/cardreader/blob/master/reader/src/main/java/com/github/tkaczenko/cardreader/service/DataConnection.java#L47) I've pointed out above.

The second step was plugging the NFC reader into my laptop and placing my ID card on top of it.

In half an hour, I was able to execute something working.

Nope, sorry, failing!

If you are a developer, you know that things rarely work in half an hour, especially if there is involved: hardware, third-party libraries and a language you haven't touched for years. So, no panic!

### What the hell does SCARD_E_NOT_TRANSACTED mean?

The failure on my first attempt looked like this.

Stack trace:

```
Caused by: org.jmrtd.CardServiceProtocolException: PICC side exception in mapping nonce step (step: 2)
    at org.jmrtd.protocol.PACEProtocol.doPACEStep2GM(PACEProtocol.java:499) ~[jmrtd-0.7.34.jar:na]
    at org.jmrtd.protocol.PACEProtocol.doPACEStep2(PACEProtocol.java:443) ~[jmrtd-0.7.34.jar:na]
    at org.jmrtd.protocol.PACEProtocol.doPACE(PACEProtocol.java:290) ~[jmrtd-0.7.34.jar:na]
    at org.jmrtd.protocol.PACEProtocol.doPACE(PACEProtocol.java:205) ~[jmrtd-0.7.34.jar:na]
    at org.jmrtd.PassportService.doPACE(PassportService.java:438) ~[jmrtd-0.7.34.jar:na]
    ... 5 common frames omitted
Caused by: net.sf.scuba.smartcards.CardServiceException: Exception during transmit
    at net.sf.scuba.smartcards.TerminalCardService.transmit(TerminalCardService.java:122) ~[scuba-sc-j2se-0.0.20.jar:na]
    at org.jmrtd.protocol.SecureMessagingAPDUSender.transmit(SecureMessagingAPDUSender.java:88) ~[jmrtd-0.7.34.jar:na]
    at org.jmrtd.protocol.PACEAPDUSender.sendGeneralAuthenticate(PACEAPDUSender.java:172) ~[jmrtd-0.7.34.jar:na]
    at org.jmrtd.protocol.PACEProtocol.doPACEStep2GM(PACEProtocol.java:475) ~[jmrtd-0.7.34.jar:na]
    ... 20 common frames omitted
Caused by: javax.smartcardio.CardException: sun.security.smartcardio.PCSCException: SCARD_E_NOT_TRANSACTED
    at java.smartcardio/sun.security.smartcardio.ChannelImpl.doTransmit(ChannelImpl.java:226) ~[java.smartcardio:na]
    at java.smartcardio/sun.security.smartcardio.ChannelImpl.transmit(ChannelImpl.java:89) ~[java.smartcardio:na]
    at net.sf.scuba.smartcards.TerminalCardService.transmit(TerminalCardService.java:116) ~[scuba-sc-j2se-0.0.20.jar:na]
    ... 23 common frames omitted
Caused by: sun.security.smartcardio.PCSCException: SCARD_E_NOT_TRANSACTED
    at java.smartcardio/sun.security.smartcardio.PCSC.SCardTransmit(Native Method) ~[java.smartcardio:na]
    at java.smartcardio/sun.security.smartcardio.ChannelImpl.doTransmit(ChannelImpl.java:192) ~[java.smartcardio:na]
    ... 25 common frames omitted
```

I'm not an expert on APDU transmission, so I googled `SCARD_E_NOT_TRANSACTED`. Nothing seemed to be able to clarify the root cause of this error. The people who encountered it solved it with the most heterogeneous solutions, leading me to think it was a generic transmission error. Like when your father called you to report a problem with his laptop, saying, "it doesn't work!".

I downloaded JRMTD's code to put a breakpoint where the code raises the error, but I didn't get too far. I hit against some code part of the JDK. As foreseeable, the exception `sun.security.smartcardio.PCSCException` was thrown by `sun.security.smartcardio`. That I couldn't debug it.

No more options for the code. What about the hardware?

### Oooooh right, the drivers

It must be the hardware! Well, not the hardware directly but the drivers.

Ok, let's re-install them.

Aaaaaand, same issue!

I google more stuff and read some hypotheses that on the last version of macOS, things might not work as they should for card readers. No evidence, of course, but I have a Linux laptop, so why not try there?

### Oooooh right, the operative system

I opened my glorious and outdated Linux laptop and started from the basics.
I installed the Java IDE, cloned the project and plugged my card reader.

This time the first attempt was even worse. The JVM couldn't even find the NFC reader.
After all, my laptop it's right. I didn't even install the drivers. It would have been too much luck for the Linux world.

I downloaded the drivers, and after a few attempts, I compiled them and successfully executed the command `make install`. Alongside the drivers, I installed pcsc-tools, a tool to manage card readers on Linux.

Ready to roll! Let's execute the java app again on the promising OS...

Aaaaaand, same issue! `SCARD_E_NOT_TRANSACTED`

### Debug all the things

Google: "how to debug ADPU commands pcsc Linux".

Luckily enough, `pcscd` can be launched with a parameter `--debug`. Great!
I did that, and I saw more logs printed on the terminal.

Exciting! We're getting there!

I execute my java app again, and on my terminal pops out this error:

![pcsc log screen](/images/apdu-logs-error-command-too-long.png)

`commands.c:1738:CmdXfrBlockAPDU_short() Command too long (273 bytes) for max: 261 bytes`

Wow, such a clear explanation has been hidden on the deepest logs in the stack the whole time :O

That's the peak of the horror.

Also, why such a long command? During PACE protocol, the code tries to exchange a Public Key with the chip to establish a secure connection. The longer command makes sense.

### Last mile

Ok, maybe that limit it's arbitrary, and I can get around it.
I opened the drivers' source code on my IDE; I found the line commands.c:1738 and changed the limit to 361 bytes. Smart no?

Compile and install again.

Aaaaaand it doesn't work! Error: `LIBUSB_ERROR_TIMEOUT`

The reader doesn't answer when sending an APDU command longer than 261 bytes; there must be some constraint at a hardware level.

## The end

I was disappointed to find out that my beloved ACR122U NFC reader doesn't support APDU commands longer than 261 bytes, and therefore it's not a suitable option for testing ICAO e-documents. I learned it the hard way :'(

It was an interesting journey, and I hope it can help someone trying to do the same.

PS: I bought another NFC reader on eBay. I might have another horror story to tell :D
