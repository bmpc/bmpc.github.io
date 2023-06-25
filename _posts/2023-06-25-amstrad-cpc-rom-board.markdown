---
layout: post
title: "Amstrad CPC 6128 ROM Board"
description: "A ROM Board for the Amstrad CPC 6128 that is capable of loading two expansion ROMs at the same time from an SD card."
date: 2023-06-25 11:00:00 +0100
image: /post_assets/5/rom_board_1.jpg
tags: atmega328p amstrad cpc6128 ROM sdcard retrocomputer
comments: true
---

In this article, I'm going to describe how I've created a ROM Board for the Amstrad CPC 6128. This is a continuation of my previous project: [Building an Amstrad CPC expansion ROM in a breadboard](https://bmpc.github.io/bulding-an-amstrad-cpc-expansion-rom).

Last time, we managed to build an expansion ROM in a breadboard for the Amstrad CPC 6128. Although the prototype worked as expected, I wanted something more "permanent" and usable.
<!--more-->

There are plenty of [ROMS available for the Amstrad CPC](http://www.cpcwiki.eu/index.php/ROM_List). I thought of putting the ROMs on an SD card, and somehow control the ROM that is loaded on the Amstrad CPC.
Since I was familiar with the Arduino platform, I quickly put together a prototype with an Arduino Uno as the microcontroller, a small OLED screen that I had laying around and an SD Card board.

The goal was to have an external custom PCB, based on the atmega328p, connected with the Amstrad CPC 6128 using the [expansion port](https://www.cpcwiki.eu/index.php/Connector:Expansion_port), that would interface with an SD card board and also provided a way to control the currently selected ROMs.

Because the EEPROM I used in my previous project had 32K of memory (**AT28C25615U**), it is possible to have two ROMs loaded at the same time because each ROM is 16K. This is useful as there are some programs and games that require two ROMs available as expansion ROMs.

## How it works

The Arduino is responsible to load the selected ROMs from the SD card into the extension memory whenever the Amstrad CPC boots up. The load process for both ROMs takes around 6 seconds. Hence, we need a way to hold the cpc6128 until the ROMs finish loading.
This can be achieved using the BUSREQ and BUSACK lines on the Z80. When the Amstrad CPC starts, the ROM Board will pull the BUSREQ immediately LOW with a pull low resistor. Once the BUSACK line is LOW, the Z80 is signaling that control of the address bus, data bus and output signals has been released, and our ROM Board can take control and it starts loading the ROMs. When the atmega328p finishes loading the ROMS, it will pull the BUSREQ HIGH and the Z80 resumes processing.

I ended up using a 32K SRAM (**AS6C62256A**) instead of the EEPROM, as the SRAM is more suited for frequent writes. The interface and pinout of both ICs is the same.

The atmega328p keeps the selected ROMs within it's internal EEPROM memory. In order to load the ROMs, the microcontroller will first get the selected ROM names from it's EEPROM and then starts copying the ROMs from the SD card into the SRAM.

When the Z80 resumes, the Amstrad CPC will start the boot process and will find the expansion ROMs (if selected) in positions #14 and #15 for the lower and upper ROM respectively.

From this point forward, the circuit we previously prototyped with a breadboard in the [previous project](https://bmpc.github.io/bulding-an-amstrad-cpc-expansion-rom) assumes control whenever the Amstrad chooses ROM #14 or #15 through the designated IO port.

### Loading the ROMs

In order to copy the ROM data from the SD card into the SRAM memory, three 8-bit shift registers are used as there aren't enough GPIO pins on the atmega328p for the 15 address lines, 8 data lines and multiple other control signals needed for the whole board.

Ensuring that only one device assumes control of the bus at all times is a crucial aspect to consider. Because we are using the same address and data bus of the Amstrad CPC, when the amstrad CPC is accessing the bus, the ROM Board shift registers must be tri-stated. Likewise, whenever the ROM Board is accessing the bus in order to load the ROMs, the Amstrad CPC must release control of the bus. This is accomplished by requesting control of the bus through the BUSREQ line being pulled low.

### Decoding logic

Besides the decoding logic described [previously](https://bmpc.github.io/bulding-an-amstrad-cpc-expansion-rom), additional components were required.

Whenever the Amstrad CPC tries to select a given ROM through the IO port, if the ROM being selected is a ROM provided by the ROM Board (#14 or #15) then a latch is used to store this information. Another latch is used to store which of the ROMs was selected: #14 or #15. A Dual D-Type flipflop (SN74HC74N) was used for both latches.

The Tristate buffer (SN74HC126N) is used to tie the A14 line to the selected upper or lower ROM stored in the previous latch. The A14 line must be tri-stated from the Amstrad CPC bus whenever the ROMS are being loaded.

## User interface

Upon startup, the selected ROMs for the upper and lower positions are loaded to the SRAM. The display shows the loading progress of each ROM. Once the ROMS are loaded, the display shows the loaded ROM names.

In order to select ROMs from the SD card, five buttons are provided so that the user can:
 - **SELECT** a given ROM for the upper or lower ROM position
 - **CANCEL** the selected action and return to the loaded ROMs initial screen.
 - Move **UP** and **DOWN** the ROM list on the SD card
 - **RESET** the Amstrad CPC and the ROM Board

**Browsing ROMs:**

 - From the main screen, click on the UP or DOWN buttons to enter the browsing mode.
 - In browsing mode, navigate using the UP and DOWN buttons to choose a ROM file from the SD card.

**Selecting a ROM:**

 - Once the desired ROM is chosen, press the SELECT button.
 - A new screen will appear, prompting the user to select the ROM bank position: Lower or Upper.
 - Switch between the lower and upper bank by using the UP and DOWN buttons.

**Saving the ROM:**

 - Press the SELECT button to save the ROM file on the selected bank.
 - To complete the process, restart the Amstrad CPC. This can be done by turning off and on the computer or using the RESET button on the Rom Board.

**Canceling ROM Selection:**

 - If you wish to cancel the ROM selection, press the CANCEL button.
 - You will be returned to the main screen.

## The circuit

The circuit can be divided into two parts:
 1. ROM loading, browsing and selection
 2. ROM Expansion interface with the Amstrad CPC

The common part between them is the address and data bus and the SRAM.

<div style="text-align:center">
  <img src="../post_assets/5/circuit_1.png" alt="schematic_1" width="100%" />
</div>

## ROM Board preview

Here are a some photos of the ROM board PCB assembled and some games loaded in the Amstrad CPC 6128 from the ROM Board.

<div style="text-align:center;margin-bottom:20px;">
  <img src="../post_assets/5/rom_board_1.jpg" alt="rom_board_1" width="60%" />
</div>

<div style="display:flex;gap: 20px;justify-content: center;">
  <iframe width="784" height="441" src="https://www.youtube.com/embed/o8quGkeSou8" title="Amstrad CPC ROM Board" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe>
</div>

&nbsp;


## GitHub project

You can find more details about this project, along with the Arduino code and KiCad schematics here: [https://github.com/bmpc/amstrad_cpc6128_romboard](https://github.com/bmpc/amstrad_cpc6128_romboard)

## Final thoughts

Building the prototype for this project on breadboards was not easy. Even though I found and fixed some bugs, the circuit was not stable. Sometimes it worked very well and the ROMs were loaded without any issue, but other times the circuit would crash or restart.
After doing a lot of debugging using my primitive tools such as a multimeter and a cheap logic analyzer, I couldn't find any reason for the unexpected behavior. Due to the multitude of connections and numerous integrated circuits already incorporated into the circuit, my suspicions turned towards the breadboards. If any wire is not making a good connection, this is enough to cause problems. Despite my concerns, I proceeded with building the PCB anyway. Much to my surprise, after soldering all the components, it worked like a charm :satisfied:

The next step is to build a ROM Board controlled entirely from the Amstrad CPC, eliminating the necessity for an external display or separate ROM selection buttons.

Until next time!

## *Disclaimer*
*I'm a software engineer with no background in electronics. Please don't consider what you read here to be best practice. Use at your own risk.*

*Nevertheless, feedback is always welcome.*
