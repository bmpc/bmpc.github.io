---
layout: post
title:  "Building an Amstrad CPC expansion ROM in a breadboard"
descrition: "Build an Amstrad CPC expansion ROM prototype using a breadboard and a couple of logic gates"
date:   2021-04-15 20:00:00 +0100
image: /post_assets/3/circuit_preview.jpeg
tags: Z80 amstrad cpc6128 ROM EEPROM retrocomputer
redirect_from: 
  - /2021/04/15/bulding-an-amstrad-cpc-expansion-rom.html
comments: true
---


I've recently started to play around with an Amstrad CPC 6128. I was able to interact with the CPC using an Arduino Mega connected to the expansion port and act as a peripheral device using an IO port. 
This project/experiment required some programs running on the CPC, and one of the main things left to do was to include these programs in a ROM so they could be accessible using RSX extensions.

I have discovered another excellent [article by KernelCrash](https://www.kernelcrash.com/blog/amstrad-cpc-rom-emulation-using-an-stm32f4/2018/06/03/) that uses an STM32F4 to emulate a ROM. This is exactly what I'm looking for.
<!--more-->
In order to understand how expansion ROMs work in the CPC 6128 in the first place, I set up to build an expansion ROM.

Back in the 80's, it was common to have DIY expansion boards. These projects were often included in magazines, just like the ones below:
 - [https://www.cpcwiki.eu/index.php/ACU_Romboard_(DIY)](https://www.cpcwiki.eu/index.php/ACU_Romboard_(DIY))
 - [https://www.cpcwiki.eu/imgs/3/32/Amstrad_rom_expander.pdf](https://www.cpcwiki.eu/imgs/3/32/Amstrad_rom_expander.pdf)

I followed these projects to build a prototype using a breadboard and a couple of logic gates.

## How it works

Whenever the CPC wants to communicate with the Upper ROM, it will first write a byte with the selected Upper ROM bank number into port **$DF00**. There can be up to 252 expansion ROMS mapped over the top 16K of memory, starting at #C000.
After this, the **ROMEN** signal will go LOW, and the CPC will start addressing the Upper ROM. 

The CPC kernel is responsible for selecting a ROM by sending this IO request to port $DF00. When the CPC is turned on, ROM 0 is selected as the foreground program. If there is no expansion ROM mapped to ROM address 0, the onboard ROM is used, and BASIC is entered. Otherwise, if an expansion ROM is fitted at ROM address 0, it takes precedence over the onboard ROM. The Gate Array controls if the Upper or Lower ROM is enabled (or if RAM is mapped to that region).

We must guarantee that there is a unique Upper ROM enabled at any given time. Therefore, whenever an expansion ROM is active, the internal CPC Upper ROM must be disabled. This is done by bringing the **ROMDIS** line LOW, a signal conveniently provided by the CPC for this purpose. After this, any memory requests to the ROM's address space will be handled solely by the selected expansion ROM.

The CPC does not store the selected ROM Bank number anywhere. Peripheral devices are responsible for **latching** the selected ROM bank whenever the bank number is written to port $DF00. 

The Kernel supports two varieties of expansion ROMs: foreground and background. A foreground ROM contains one or more programs, only one of which may be running at one time. The onboard BASIC is the default foreground program. Other examples are CP/M and applications such as word processors and spreadsheets. Background ROMs lie dormant until initialized by the foreground program. During initialization, the background software may allocate itself some memory and initialize any data structures and hardware.

The Amstrad CPC provides **Resident System Extensions** (**RSX**) that are similar in use to a background ROM, but must be loaded into RAM before they can be used. This is a useful expansion mechanism that allows the CPC to recognize external commands which are mapped by the system and can be called directly from BASIC. This is exactly what I need in order to provide the utility programs directly from the expansion port.


## The circuit

In this project, there will only be a single expansion ROM bank. Therefore, I can simply use a **D-latch** to store HIGH when the appropriate Upper ROM is selected and LOW otherwise.

We must start by checking when the **IOREQ** line goes low. This means an I/O request is being triggered. Because of the way CPC uses I/O space, it is only necessary to detect address line **A13** going LOW to know when the address is **$DF00**. This greatly simplifies the decoding process as we just need to check one line instead of 16. We are also expecting a write to the IO port (**WR**). Therefore we must assert if this line is also LOW.

The IOREQ, WR, and A13 lines are all connected to a single NOR gate. The NOR gate output will be connected directly to the Enabled line of the D-Latch. When a byte is written to $DF00 it will enable the D-latch, effectively latching the selected ROM bank.
I've chosen **ROM bank 15** as it was the easiest to decode. Bank 15 will be enabled when the value **0xF** (**00001111**) is written to $DF00. This means data lines **D0, D1, D2, and D3** are all HIGH. Since I had 4-input AND gates available, this was simple. I connected the four lowest data lines to a single AND gate and the output to the D-latch data line.

Whenever the operating system is using the Lower ROM (firmware), all other ROMS (including expansion ROMS) must be disabled. This is achieved by connecting both the Q output of the latch and the **A15** line to another AND gate. The output of this AND will determine if our ROM is enabled by setting the **CE** pin LOW and also disable the internal CPC ROM by bringing the **ROMDIS** line LOW. 

The **ROMEN** signal is connected directly to the **OE** (Output Enable) pin of our expansion ROM. Both CE and OE need to be LOW for the ROM to output information into the databus.

I quickly assembled the circuit in a breadboard using AND and NOR gates. I used the 74HC logic family chips since I had some lying around. I have also created the D-latch using discrete AND/NOR gates.

For the initial test, I flashed an expansion test ROM with an RSX ([GitHub Gist](https://gist.github.com/bmpc/9a66f5b7567079f6ee243594332bd158)) into a 32K EEPROM (**AT28C25615U**) using my [**Arduino EEPROM programmer**](https://github.com/bmpc/arduino_eeprom_programmer). 

When I first plugged in the circuit to the CPC expansion port, the CPC would freeze or restarted. The EEPROM was not even plugged into the breadboard yet. I could also see that the circuit was very unstable. Something was definitely not working correctly.

After a couple of hours debugging the problem, I realized that I had used a triple input NOR gate instead of a dual input like I was expecting. 🤦
After replacing the chip with the correct one, the CPC booted just fine. To my surprise, after plugging in the EEPROM, I saw the welcome message that I had programmed into the ROM on the boot screen. Good news! 

After this, I tried a couple more ROMS like "Discology" or even a game like "Pac-man", and everything was working correctly.

You can find more Amstrad CPC ROMS here: [http://www.cpcwiki.eu/index.php/ROM_List](http://www.cpcwiki.eu/index.php/ROM_List)

Here is the ROM expander cirtuit:

<div style="text-align:center">
  <img src="/post_assets/3/circuit.png" alt="Circuit" width="80%" />
</div>

Components:
 - AT28C25615U EEPROM x 1
 - 74HC02 (2-input NOR) x 1
 - 74HC27 (3-input NOR) x 1
 - 74HC21N (4-input AND) x 2
 - 1kΩ resistor x 1
 - Diode x 1
 - Breadboard x 1

And here is the circuit in a breadboard connected to the CPC:

<div style="text-align:center;margin-bottom:40px">
  <img src="/post_assets/3/circuit_preview.jpeg" alt="breadboard circuit" width="80%" />
</div>

The small breadboard you see on top of the main breadboard is a D-latch circuit.

This is the ROM being loaded when the CPC boots:

<div style="text-align:center;margin-bottom:40px">
  <img src="/post_assets/3/pacman_rom.jpeg" alt="breadboard circuit" width="80%" />
</div>


## Final thoughts

I now understand the basics of how expansion ROMs work in an Amstrad CPC 6128 and how ROM bank switching occurs.

My next goal is to try and emulate an expansion ROM using the Arduino. I suspect this is not going to be easy as KernelCrash used a microcontroller with a significant faster clock running at 168MHz+.

Nevertheless, I have another way of interacting with the CPC and loading ROMS with operating systems, utilities and even games.

