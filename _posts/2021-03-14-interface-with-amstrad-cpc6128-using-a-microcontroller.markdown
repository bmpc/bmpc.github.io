---
layout: post
title:  "Interfacing with the Amstrad CPC 6128 using a microcontroller"
descrition: "Connecting the Amstrad CPC 6128 expansion port to a microcontroller (Arduino Mega) and copying information from a micro SD card."
date:   2021-03-14 8:00:00 +0100
image: /post_assets/2/circuit1.jpeg
tags: Z80 amstrad cpc6128 arduino retrocomputer
comments: true
---

I'm a proud owner of an Amstrad CPC 6128 with a green monochrome monitor. I got this computer around 2004, long after its golden era – the 80's. I feel that I've never given it the love it deserved. Back then, I played a couple of games and abandoned it almost immediately… until now.

I've recently become interested in retrocomputing (mostly because of this guy [https://internalregister.github.io](https://internalregister.github.io/), so I decided to take the CPC 6128 out of the attic and give it another try.

Everything was working fine except the disk drive. I disassembled and cleaned the whole computer. After this, I fixed the drive by adding a new rubber band (a very common problem with these drives as the original rubber band will break with time).

Everything ready, let's start! The first thing I did was to play some games stored on the (very old) 3.5" disks I had and to learn Basic 1.1 (Locomotive Basic 1.1).

There's an extensive archive of games and programs free to download: [https://www.planetemu.net/roms/amstrad-cpc-games-dsk](https://www.planetemu.net/roms/amstrad-cpc-games-dsk). However, I did not have any way of copying these files into the CPC. 

With this need, and the urge to learn more about these 8-bit computers, I started researching how to interact with the CPC and perhaps even upload games and programs using an SD card or directly from another computer.

I almost immediately discovered the expansion port on the back of the CPC. This port provides direct access to the Z80 CPU and some other CPC functions. I reckon that using a microcontroller (like the Arduino) would be a good way to start exploring the signals from this port.

Searching the web for previous projects using the expansion port resulted in two significant results:

1. [Universal Serial Interface for Amstrad CPC (a.k.a USIfAC)](http://retroworkbench.blogspot.com/p/universal-serial-interface-for-amstrad.html)
2. [Arduino IO card for the CPC6128](https://hackaday.io/project/169565-arduino-io-card-for-amstrad-cpc-6128)

The second project (which was also using an Arduino as the microcontroller) was based (to some extend) on the first project (which was built using a PIC 16F1579). The idea was to create an IO extension that would communicate with the CPC using a specific I/O port.

There are two specific CPU instructions on the Z80 for Input and Output: **IN** and **OUT**. These instructions allow programs to send/receive a byte to/from a particular address port. These I/O ports can be mapped to external peripherals, and therefore can be used to transfer data directly from an Arduino to the CPC memory or save files to the disk drive.

## Decoding logic

Provided with this information, I started connecting all the relevant lines from the expansion port to the Arduino.

Here is the expansion port pin layout. 

<div style="text-align:center">
  <img src="/post_assets/2/CPCExpansionPort.png" alt="CPC Expansion Port" width="60%" />
</div>

    1  Sound     2  GND
    3  A15       4  A14
    5  A13       6  A12
    7  A11       8  A10
    9  A9        10 A8
    11 A7        12 A6
    13 A5        14 A4
    15 A3        16 A2
    17 A1        18 A0
    19 D7        20 D6
    21 D5        22 D4
    23 D3        24 D2
    25 D1        26 D0
    27 VCC       28 *MREQ
    29 *M1       30 *RFSH
    31 *IORQ     32 *RD
    33 *WR       34 *HALT
    35 *INT      36 *NMI
    37 *BUSRQ    38 *BUSAK
    39 READY     40 *BRST
    41 *RSET     42 *ROMEN
    43 ROMDIS    44 *RAMRD
    45 RAMDIS    46 CURSOR
    47 LPEN      48 *EXP
    49 GND       50 CLK4

[http://www.cpcwiki.eu/index.php/Connector:Expansion_port](http://www.cpcwiki.eu/index.php/Connector:Expansion_port)

First of all, I needed to choose a specific port to communicate with the Arduino. I went with the same port used in the previous projects (**&FBD0**) as it is typically used for Serial communication: [http://cpctech.cpc-live.com/docs/iopord.html](http://cpctech.cpc-live.com/docs/iopord.html).

Some decoding logic is required to know when the CPC is trying to communicate with the &FDB0 port. This logic was implemented using just a couple of NOR gates and one AND gate with the relevant address and control lines. 

The (**D0…D7**) data lines are all necessary, of course. These lines will contain the byte being transferred.

When there's an I/O request, the Z80 brings the **IOREQ** line low. The IN and OUT operations are identified by the **RD** and **WR** lines, respectively. When the CPU reads a given port with IN, the **RD** line is LOW; otherwise, it is high. 

Another important signal is the **M1** which stands for Machine cycle one. Each instruction cycle is composed of tree machine cycles: M1, M2 and M3. M1 is the "op code fetch" machine cycle. This signal is active low. We must make sure M1 is high when communicating with the Z80.

The final signal (and definitely the most interesting) is the **WAIT**. When this signal is low, it tells the CPU that the addressed memory or **I/O devices** are not ready for a data transfer. The CPU will continue to enter the WAIT state whenever this signal is active, effectively pausing the CPU.

I ended up not using all the address lines for decoding. If bit 10 and bit 5 are reset, this means we are using an expansion peripheral and, more specifically, the serial port according to the CPC I/O port allocation. The other bits are ignored.

While assembling the circuit, I discovered that some of the CPC 6128 lines required pull-up resistors. The interrupt line was being triggered without any reason because these pins were floating, namely the address lines A0, A5, A10, and the IOREQ line. I suspect this is related to the chip family I used: 74HC. The other projects used CMOS chips and didn't need any pull-up resistors.

Below is the schematic for the circuit I used:

<div style="text-align:center">
  <img src="/post_assets/2/schematic.svg" alt="Circuit" width="80%" />
</div>

Components:
 - 220Ω resistor x 3
 - 10kΩ resistor x 1
 - NPN transistor x 1
 - 74HC21N x 1
 - 74HC27N x 1
 - Arduino Mega 2560 x 1
 - Breadboard x 1
 - Micro SD card reader x 1


## Synchronizing the Z80 with the microcontroller

The timing when communicating between Z80 IN/OUT instructions and the Arduino is critical. The Z80 is clocked at 4 MHz while the Arduino Mega (which I'm using for this project) is clocked at 16 MHz. However, this speed difference is not sufficient for the Arduino to reply to the Z80 in time or read the data bus before the Z80 moved on to do other things and released it. Hence, we must use the WAIT signal to pause the CPU while the Arduino does its job of a) putting a byte into the data bus or b) reading a byte from the data bus.

Whenever the decoding logic signals that a byte is being transferred (IN/OUT), an interrupt is triggered in the Arduino. We can then set the WAIT line LOW. Again, timing is the key. Setting the WAIT line LOW using software only after the interrupt is triggered is not an option because the Z80 WAIT state is sampled before we can reply.

Therefore, the interrupt signal itself is used to bring the WAIT line low. After this, we must find a way to release the WAIT line (set it HIGH) after the Arduino finishes processing the byte. This can be done using a transistor and a control line from the Arduino as a switch. The control line is connected to the Emitter, the interrupt line connected to the Base, and the WAIT line to the Collector.

This control line will always be active. This means the WAIT signal is also triggered if this control line is LOW and the interrupt is also triggered. When the Arduino is ready, it will bring the control line HIGH for a brief moment, giving enough time for the Z80 to process the byte (in case of an IN instruction). 

This moment is also crucial. If it's too long, the Arduino might not be ready to process the next interrupt. On the other hand, if it is too short, the Z80 might not have enough time to sample the data bus. The other projects used a "sample and hold" circuit with the correct resistor and capacitor to set the control line LOW again.

Studying the Z80 timing diagram for Input/Output cycles, we can see that the In is sampled from the data bus for a brief moment, and right after this, the IOREQ goes HIGH. 

<div style="text-align:center">
  <img src="/post_assets/2/z80_io_timing.png" alt="Z80 IO timing diagram" width="80%" />
</div>

I used this knowledge to release the Arduino line at just the right time. If the IOREQ line is HIGH, this means the interrupt line is no longer active. Right after pulling the control line HIGH, the interrupt line is polled continuously. When this signal changes, we can bring the control line LOW again to be ready for the next request/interrupt. Here is where we take advantage of the faster clock on the Arduino. This poll needs to be done in AVR assembly to ensure the Arduino starts polling the line before the Z80 sets the IOREQ HIGH. Here is the code that releases the WAIT line:

    void releaseWaitAfterWrite() {
      __asm__ __volatile__(
        "SBI  PORTC, 0              \n" // Set bit 0 in PORTC - WAIT line HIGH 
        "SBIC PINE, 4               \n" // Skip next instruction if Bit 4 (Interrupt) is Cleared
        "RJMP .-4                   \n" // Relative Jump -4 bytes - 
        "LDI  r25, 0x00             \n" // Load r25 with 0x00 - B00000000
        "OUT  DDRA, r25             \n" // store r25 in DDRA - Set DDRA as output again (default)
        "CBI  PORTC, 0              \n" // Clear bit 0 in PORTC - WAIT line LOW 
        "SEI                        \n" // Set Global Interrupt
        
        :::"r25"
      );
    }
    
After much trial and error, I managed to send and receive bytes to/from the Z80.

This can easily be tested just by calling the following commands from Basic:

    OUT &FBD0, value   // send byte
    value = INP(&FBD0) // receive byte

After this, I created a small Z80 assembly program to continuously receive bytes from IN commands without any issue. This was proud that the WAIT lined was doing its job.

<div style="text-align:center;margin-bottom:40px">
  <img src="/post_assets/2/circuit1.jpeg" alt="breadboard circuit" width="80%" />
</div>




## Listing and copying files

Since I was already capable of transferring bytes to and from the Arduino, I started thinking about transferring information (e.g., programs or games) from the Arduino to the CPC. 

After some research, I knew that it would be challenging (to say the least) to load games directly into memory. This is because most games use multiple files that are loaded at different stages. Some use multiple binary files, and the majority use a small Basic loader program. 

I really admire @ikonsgr's work as he has really managed to do this for most games by overriding the CPC system routine calls to load files from disk (interesting discussion here: 
[https://www.cpcwiki.eu/forum/programming/mc-startboot-program-without-reseting-firmware-jumpblock](https://www.cpcwiki.eu/forum/programming/mc-startboot-program-without-reseting-firmware-jumpblock/). 

Hence, my goal was more straightforward: transfer information to and from the CPC disk drive.

The only thing I needed was a couple of CPC programs to: 
1. show all the files present on an SD card controlled by the Arduino, and
2. to copy a file from the SD card into the CPC disc drive.

I created a simple protocol to communicate with the Arduino using IN/OUT instructions. Using this protocol, I programmed two small Z80 assembly programs (which was a new experience to me):
 - *dir* – lists all the files present on the root folder of the SD card
 - *copy* – which, provided with the filename as a parameter, copies the SD card file into the CPC disk drive.

Of course, I had to find a way to copy the binaries into the CPC drive to test. This was kind of a chicken-egg problem. I created a small Arduino program to copy these initial binaries, which contained the binary code directly within the Arduino sketch.

With some patience and a lot of debugging, I managed to do this successfully. I tried some games, and they worked flawlessly. I did get some occasional errors, but these were related to old 3.5'' disks with bad sectors.

Most games nowadays are compacted into ".DSK" files, a disk image format. I extracted the files and copied all of them on the SD card. Then, I copied all the files into the CPC disk using the copy program.

<div style="text-align:center">
  <img src="/post_assets/2/cpc_dir.jpeg" alt="dir cmd" width="80%" />
</div>

<div style="text-align:center;margin-bottom:40px">
  <img src="/post_assets/2/cpc_copy.jpeg" alt="copy cmd" width="80%" />
</div>




## What's next?

I still have plans to improve this project by:
 - Creating RSX extensions for the copy and dir programs that can be loaded automatically from the Arduino by emulating a CPC ROM. This way, there is no need to have the dir and copy programs in 3.5" disks. 
 - Creating a PCB based on an Arduino shield to make this project final and remove the breadboard (now covered in dust) from the back of my CPC.  
 - Support ".DSK" image files directly in the SD card

## Final thoughts

I want to highlight that this project's main goal was for me to learn how these machines worked back in the day. 

Using the expansion port to transfer information to/from the CPC is really not the best way to copy games or programs (unless you can really load them directly into memory - like USIfAC). For this, we can simply use a GOTEK drive [http://www.cpcwiki.eu/index.php/Gotek](http://www.cpcwiki.eu/index.php/Gotek) and replace the internal CPC disk drive completely. 

I managed to learn the basics of the Z80 CPU. I also gained some knowledge on how to create basic electronic circuits with some decoding logic.

The thing that fascinates me the most about these computers is understanding the interaction between software and hardware.

## GitHub project

You can find more details about this project, along with the source code here: [https://github.com/bmpc/amstrad_cpc6128_interface](https://github.com/bmpc/amstrad_cpc6128_interface)
