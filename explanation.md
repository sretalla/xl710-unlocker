I had a bit more time this weekend, and I managed to unlock my
X710-DA4. It now accepts any SFP+. I tried four different models of
10GBASE-LR transceivers in the card, and they all seemed to work just
fine. Admittedly, I only tested that they could hit 1.1GiB/s TCP flows
for a few minutes before picking one transceiver type and sticking
with it. In particular, I had some A7EL-LND3-ADMA transceivers lying
around that I found for 10EUR a piece and I am now using these
reliably the whole day. In summary, FUD aside, the X710-DA4 works
great with most SFP+s and the decision to lock this card down seems
more like it has something to do with monopolistic practices than
anything else. If I run into any problems with the cheaper SFP+s I
will follow-up with a reply to the list.

The XL710 has four records in NVM that describe whether or not the
card should reject an unknown SFP+. When you flip these four bits, it
will accept any SFP+. I have tested Linux only, but the driver works
unmodified with an unlocked card. Todd Fujinaka mentioned that Intel
sells an unlocked optics card, so I imagine the drivers were tested
against those cards which is why it works under linux. Therefore, I'd
wager the windows driver likewise works with an unlocked card, though
I have not tested it.

Be warned: running nvmupdate64e -rd will restore the lock bits! Normal
update appears to preserve their value.

To unlock your card, you will need linux 4.5, nvmupdate64e v1.26.17.09
, skill with Cl, and some time. I have not prepared a general-purpose
unlocking tool, because the NVM format might differ between cards and
thus this procedure should only be undertaken by someone who knows
what they are doing.

DO NOT FOLLOW THIS GUIDE UNLESS YOU DON'T FEAR BRICKING YOUR CARD!

Step one is to find the "PHY Capability LAN 0" structure in the NVM.
The NVM image that nvmupdate64e v1.26.17.09 writes seems to have a
corrupt "PHY Capability LAN 0 Pointer", so you will need to find the
record by hand. Use the attached my-tool.c to read the NVM. You will
need to set devid to the PCI express device ID for your card. lspci -n on
my machine showed:

01:00.0 0200: 8086:1572 (rev 01)
01:00.1 0200: 8086:1572 (rev 01)
01:00.2 0200: 8086:1572 (rev 01)
01:00.3 0200: 8086:1572 (rev 01)
... and thus I set devid to 0x1572.

You also need to set ifr.ifr_name to one of the corresponding ethernet
names. In my machine the card was eth2, eth3, eth4, eth5. I imagine
other models might have less devices and the numbers assigned depend
on what other cards udev has seen in your system. It does not matter
which card you pick, as long as it matches the devid you picked
earlier.

Once mytool works, run ./mytool 0x6870 0xc as root and see if it has a
similar record to mine.
My "PHY Capability LAN 0" record looked like this:
00006870 + 00 => 000b
00006870 + 01 => 0022 = external SFP+
00006870 + 02 => 0083 = int: SFI, 1000BASE-KX, SGMII (???)
00006870 + 03 => 1871 = ext: 0x70 = 10GBASE-{SFP+,LR,SR} 0x1801=crap?
00006870 + 04 => 0000
00006870 + 05 => 0000
00006870 + 06 => 3303 = SFP+ copper (passive, active), 10GBase-{SR,LR}, SFP
00006870 + 07 => 000b
 00006870 + 08 => 2b0c
*** This is the important register ***
(0xb=bits 3+1+0 = enable qualification (3), pause TX+RX capable (1+0),
 0xc = 3+2 = 10GbE + 1GbE)
00006870 + 09 => 0a00
00006870 + 0a => 0a1e = default LESM values
00006870 + 0b => 0003

I had 3 more records just like this at offsets 0x687c, 0x6888, and
0x6894. The key field is at offset 0x08 (PHY Capabilities Misc0). You
need to clear bit 11. In my case that meant writing 0x230c.

If your record is not at this location, run mytool 0 0x8000 >
somefile, and start looking for 4 occurrences of "000b" that are
separated by 0xc addresses and repeat four times. This is how I found
my records, since the NVM image from Intel does not have pointers to
the correct location.

Once you've found the first record's offset, update mypoke.c with the
correct offset. Then calculate the correct value for the Misc0
register by subtracting bit 11 and modify the line
"*(uint16_t*)(eeprom+1) = 0x230c;". Then, compile and run mypoke.
Finally, confirm with mytool that your new Misc0 register was
correctly written. If anything goes wrong, run nvmupdate64e -rd BEFORE
you reboot! This will blow away any changes you made by mistake.

If everything looks good, run nvmupdate64e -u just to make sure the
device is in a valid state. Then, power cycle your machine, insert the
SFP+ of your choice, and enjoy your unlocked card.

If this works for you, please let me know. If this proves useful to
enough people, perhaps we can put together a more user-friendly
unlocking procedure. As I said, without seeing a few more cards, I
would not want to automate this.
