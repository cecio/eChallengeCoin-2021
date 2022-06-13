# eChallengeCoin 2021 Hack Guide

Welcome back! This is my second edition of the "eChallengeCoin" hack guide.  This is a bage/coin/board created by [Bradán Lane Studio](https://www.tindie.com/stores/bradanlane/) and some time ago I created a guide for the [2020](https://github.com/cecio/eChallengeCoin-2020/blob/main/README.md) edition.

<img src="https://github.com/cecio/eChallengeCoin-2021/blob/main/Pictures/Front.jpg" alt="thecoin" style="zoom:17%;"/>

The board contains an adventure and some games, which are very funny. But since I like reversing, I'd like to approach it finding  "my way" :-).

I suggest to have a look to the previous guide to have some background and have a clearer picture, but you can also just read this if you prefer.

As the last time, consider this tutorial as an additional challenge: I'll give you hints on how to hack this, but then you have to do your homework to get things out from the badge :-)

## Tools

I used other tools this time:

- An **Arduino Uno** used to dump the firmware and the EEPROM
- A couple of component for the Arduino: a `10μF capacitor` and a `4k7 resistor`
- [jtag2updi](https://github.com/ElTangas/jtag2updi)
- [Ghidra](https://github.com/NationalSecurityAgency/ghidra) to have a "static" look to the code

## Step 1: a look into the Hardware

To expose the juicy part, carefully remove the 3D printed part on the back. There are different components there, but the most interesting part for us is the MCU: `ATmega4809`

<img src="https://github.com/cecio/eChallengeCoin-2021/blob/main/Pictures/Back_1.jpg" alt="thecoin_back_1" style="zoom:27%;"/>

We are interested in the MCU because the first thing that we need to do is to try to dump the firmware which, in this kind of MCU, is stored in the flash of the MCU itself. By reading the [DataSheet](http://ww1.microchip.com/downloads/en/DeviceDoc/ATmega4808-4809-Data-Sheet-DS40002173A.pdf) you can have an idea of the features.

As you can see the board exposes the PINs/interfaces on the edge of the coin (let's call them PADs). So, since for the 2020 version I used a SPI connection, I started looking for the same, but this time things looks different: the 2021 version has more "Touch" PADs and fewer unlabelled parts on the border (only one!).

A deeper look into the `ATmega4809` DataSheet revealed me an interesting thing: this MCU has a so called `Single-pin Unified Program Debug Interface (UPDI)` mapped on PIN 41:

<img src="https://github.com/cecio/eChallengeCoin-2021/blob/main/Pictures/4809_pinout.png" alt="Pinout" style="zoom:47%;"/>

and guess what? With a multimeter I mapped this pin exactly to the only unlabelled PAD! 

`UPDI` is a Microchip proprietary interface for external programming and on-chip debugging of a device. Very interesting! So, let me check if I found the way...

## Step 2: dumping EEPROM and FLASH

**WARNING**: *before following these steps, I suggest to remove the battery from the eChallengeCoin. And, as a general rule, if you do something wrong, you may brick/destroy your boards. Be careful, you are responsible of your actions!*

I started to look around in order to find a way to interface with `UPDI` and I stumbled into this project:  [jtag2updi](https://github.com/ElTangas/jtag2updi)

Great! It looks like I can turn any **Arduino Uno** into a `UPDI` interface with few mods.

To do it, just follow these few steps:

- clone the `jtag2updi` repo

- get the content of `source` folder and copy it in your **Arduino IDE** folder (mine is `~/sketchbook`), in `jtag2updi`

- open, compile and upload the sotfware to your **Arduino**

- now place some components on the **Arduino** (disconnect it before):

  <img src="https://github.com/cecio/eChallengeCoin-2021/blob/main/Pictures/arduino_components.jpg" alt="Components" style="zoom:47%;"/>

- it's time to connect the **Arduino** to the **Coin**. You can use alligator clips (see picture below):

  - *black* pad to **Arduino** GND
  - *red* pad to **Arduino** 3.3v
  - *green* pad to Arduino PIN 6 (with resistor)

  <img src="https://github.com/cecio/eChallengeCoin-2021/blob/main/Pictures/Back_2.jpg" alt="Connections" style="zoom:47%;"/>

- connect **Arduino** to your PC

- copy the `avrdude.conf` from `jtag2updi` repository to your local folder

- and finally, dump the flash with (replace `/dev/ttyACM0` with the one mapped by your PC):

  ```
  avrdude -C ./avrdude.conf -c jtag2updi -p m4809 -P /dev/ttyACM0 -Uflash:r:flash.bin:r
  ```

- you can also dump the EEPROM content:

  ```
  avrdude -C ./avrdude.conf -c jtag2updi -p m4809 -P /dev/ttyACM0 -Ueeprom:r:eeprom.bin:r
  ```

## Step 3: the analysis

Unfortunately, unlike the last time with the 2020 version, we don't have an emulator to run the code of this MCU. I tried to use the `simavr`, but I was not able to get it working with this dump.

Since `UPDI` can be used for debugging as well, probably we could use the same interface to attach a debugger directly to the board, but to be honest, I didn't investigated this too much.

But no problem, we can go static!

The EEPROM looks like this:

<img src="https://github.com/cecio/eChallengeCoin-2021/blob/main/Pictures/eeprom.png" alt="EEPROM" style="zoom:87%;"/>

As you can see most of the EEPROM seems to be unused, but you can try to play with the values and see what happens reloading it (**be careful, you can brick your coin!** The last time we used an emulator...this time it's the real hardware). Since the EEPROM is where the board is saving your progress, you can see how things are reflected in the EEPROM itself by dumping it afrer you did something significative.

Now it's time to open the `flash.bin` file with **Ghidra** and see if we can figure out something. After analysing the code, it's pretty clear that (as usual) there are code sections and some sections that looks like data:

<img src="https://github.com/cecio/eChallengeCoin-2021/blob/main/Pictures/ghidra.png" alt="Ghidra" style="zoom:67%;"/>

Leveraging some of the things learned with 2020 version, some reversing and some guessing I understood that the whole adventure text was stored and compressed with the following format, in a specific section of the flash:

`<# of bytes to read><compressed string>`

So, with the following Python code, you can extract all you need to complete the adventure...

```
def dec_buffer(buffer):
    i = 1
    oldb = 0
    dec = ''
    for b in buffer:
        
        tmpchr = chr((( oldb + b >> i ) & 0xFF) & 0x7F)
        if tmpchr.isprintable():
            dec += tmpchr
        else:
            break
        oldb = ( b << 8 )
        i += 1
    
        if i == 8:
            dec += chr(b & 0x7F) 
            i = 1
            oldb = 0
    
    return dec

enc = open('flash.bin','rb')
buffer = enc.read()
enc.close()

start_addr = 0x3af9
dec_string = ''

while (start_addr < 0x934e):
    len = buffer[start_addr]
    dec_string = dec_buffer(buffer[start_addr + 1:start_addr + len + 1])
    print('Offset 0x%x: %s' % (start_addr + 1, dec_string))

    start_addr = start_addr + len + 1
```

And you'll get something like this:

```
Offset 0x3afa: However you reached the end, you win!
Offset 0x3b1c: Of course, there is also the possibility you did not solve the challenges using the clues in the story and you just hacked the eChallengeCoin. That's OK too!
Offset 0x3ba7: Did you find the Easter Eggs? If not, you can read through the story again at any time.
Offset 0x3bf5: Congratulations on completing the story and helping Sara solve the challenges.
...
```

## Wrap up

Again, it was very funny to play with eChallengeCoin. Please support  [Bradán Lane Studio](https://www.tindie.com/stores/bradanlane/) so that he will continue to create funny things like this. 

Thanks to @Her0_IT for the precious help!

Happy hacking!

 





