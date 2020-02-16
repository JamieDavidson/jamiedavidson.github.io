---
layout: post
title: "Cracking PUBG mobile's UserConfig encryption"
date: 2020-02-16 02:06:51 +0000
category: side-projects
tags: [encryption, just-for-fun]
---



## Cracking PUBG mobile's UserConfig encryption

### Introduction

**This is a technical writeup of the process that I followed to accomplish this objective back in early 2018. Because of the time between my writeup and writing the application, some of these details may now be inaccurate. This is an example of how NOT to do encryption, by the way. Please don't use this as a guide to 'secure' anything, ever.**

I picked up [Player Unknown's Battlegrounds](https://store.steampowered.com/app/578080/PLAYERUNKNOWNS_BATTLEGROUNDS/) (PUBG) in March 2017, when it was available through Steam's early access beta program. On the 9th of February 2018, a [mobile version was released to Android and iOS]([https://en.wikipedia.org/wiki/PlayerUnknown%27s_Battlegrounds#Mobile_versions](https://en.wikipedia.org/wiki/PlayerUnknown's_Battlegrounds#Mobile_versions)). The early days of PUBG were rife with bugs and exploits, allowing players to do things like [remove grass and drastically reduce the quality of textures](https://www.unknowncheats.me/forum/playerunknown-s-battlegrounds/239044-op-config-commands.html) without third party software, granting a player a huge advantage that at the time had no detection vectors. These were slowly patched over time, requiring a player to reverse engineer the game in order to achieve the same result, stopping the average player from achieving it.

When the mobile version of PUBG was released it was apparent that it was an old build of PUBG, and had gone through a series of graphical changes and small optimisations to accommodate mobile devices. This meant that a lot of exploits that were previously patched were available for use in the mobile version of the game on release, as the developers didn't think about this ahead of time.

The stop gap solution for these exploits was to encrypt some configuration (.ini) files used for these exploits. I can only assume that the hope was that the average player would be unable to bypass it while the developers worked on patching them properly.

### What is a .ini file?

The [`.ini` file format](https://en.wikipedia.org/wiki/INI_file) is common for configuration settings in many older languages such as C++, which is the language used for the Unreal Engine on which PUBG is built. The file format is not language specific, however newer languages and technologies avoid leaning on the format, and for Windows they were gradually replaced in favour of XML configuration files over a decade ago.

These configuration files are [well documented by Unreal](https://docs.unrealengine.com/en-US/Programming/Basics/ConfigurationFiles/index.html) and therefore require minimal amounts of reading to experiment with different options, even if they're not included in the release of the game. They allow easy saving and loading of things like graphical settings and input mappings.

PUBG Mobile uses the configuration file `UserCustom.ini` file to store these settings. The file was not encrypted in the original release of the game, and thankfully I kept a copy of the file around when the patch went live.

### Analysis of the encrypted file and patterns

#### Comparing encrypted vs decrypted

Here's what a section of the unencrypted file looks like:

```ini
[UserCustom DeviceProfile]
+CVars=r.PUBGLDR=1
```

This has 4 parts, the header `[UserCustom DeviceProfile]` dictating what portion of the settings this section belongs to.

`CVar` stands for `Console Variable` which is a common term used in game engines such as [Unreal Engine](https://docs.unrealengine.com/en-US/Programming/Development/Tools/ConsoleManager/index.html) to describe something which is intended to be changed in order to affect the output of the application.

`r.PUBGLDR` is the name of the console variable, and `=1` just sets its value to 1. We can change this to any number we like, but do not guarantee that the engine will change its behaviour if we set it outside of any predetermined values.

Here's the same console variable after the encryption process

```ini
[UserCustom DeviceProfile]
+CVars=0B57292C3B3E353D2B4448
```

All we can tell at this stage is that `r.PUBGLDR=1` is equal to `0B57292C3B3E353D2B4448` post encryption. One telling factor of these is that comparing the length of these 2 strings, the unencrypted string's length is 11 characters, and the encrypted string's length is twice that at 22. This isn't enough information to be certain of anything just yet, but it's an important thing to note for later.

Looking at a few more encrypted and decrypted console variables, the next four in the file:

```ini
+CVars=r.PUBGDeviceFPSLow=40
+CVars=r.PUBGDeviceFPSMid=40
+CVars=r.PUBGDeviceFPSHigh=40
+CVars=r.PUBGDeviceFPSHDR=40
```

And their encrypted forms:

```ini
+CVars=0B57292C3B3E3D1C0F101A1C3F292A35160E444D49
+CVars=0B57292C3B3E3D1C0F101A1C3F292A34101D444D49
+CVars=0B57292C3B3E3D1C0F101A1C3F292A31101E11444D49
+CVars=0B57292C3B3E3D1C0F101A1C3F292A313D2B444D49
```

Now we can start to observe some patterns:

These encrypted strings all start with `0B 57 29 2C 3B 3E`. Similarly, the unencrypted versions all start with r.PUBG. 6 bytes, 6 characters. This is a strong indication that each of these characters has been encrypted one at a time.

Typically when working with encrypted strings, introducing a single new character will change the output completely. You can observe this by playing around with an encryption tool online, such as [https://aesencryption.net](https://aesencryption.net/) 

We can confirm this theory by looking at the character `=` which according to these examples is `44`. We can look at other encrypted values in the file, such as:

`+CVars=0B57292C3B3E280C1815100D00351C0F1C15444B`, which corresponds to `+CVars=r.PUBGQualityLevel=2` and indeed the second last character in the encrypted string is `44`, our `=`.

Now we can test out another theory... is this actually encrypted at all? Have we just been tricked by hexadecimal? Are these just the ASCII keycodes in HEX form?

#### Back up, what's ASCII?

[ASCII](https://en.wikipedia.org/wiki/ASCII) stands for **American Standard Code for Information Interchange**, and is an agreed upon standard for character encoding. It's a standard which was developed, tweaked and approved by an entire committee in order to ensure that different brands of computers could talk to each other. All of that might sound tremendously boring (because it is), but it's something that we often take for granted as it's so intertwined with every aspect of technology. It's hard to imagine today, but there was a time when communication between different brands of devices was a huge technical challenge. [Here's a handy link to the history of ASCII](http://ascii-world.wikidot.com/history) if you're interested in reading more.

There was a phase where people on the internet created [works of art dubbed "ASCII art"](https://www.asciiart.eu/) (and also draw a lot of dicks, as you do). This was due to their creation consisting entirely of text characters. Nowadays there's software to do it for you which ruined most of the fun :(

The website [asciitable](https://asciitable.com) (registered in 1999 and a child site of [lookuptables](https://lookuptables.com)) hosts an ASCII table, showing the mapping of numbers to ASCII characters.

![ASCII TABLE](http://www.asciitable.com/index/asciifull.gif)

Our hexadecimal number for our equals sign is 44, which is 68 in decimal. Unfortunately, 68 maps to `D`, not what we were looking for. So that's one theory out of the window.

#### Some manual mapping

Given our earlier conclusion that one decrypted character maps to one byte (2 characters in HEX), we can start to manually map out some characters to see if there are any observable patterns. This is as simple as comparing the encrypted vs unencrypted strings. Just comparing a single short string can provide a large number of characters.

```
0B 57 29 2C 3B 3E 3D 1C 0F 10 1A 1C 3F 29 2A 35 16 0E 44 4D 49
r  .  P  U  B  G  D  e  v  i  c  e  F  P  S  L  o  w  =  4  0
```

Allows us to map out the following characters:

| Character | Hexadecimal value | Decimal value |
| --------- | ----------------- | ------------- |
| c         | 1A                | 26            |
| e         | 1C                | 28            |
| i         | 10                | 16            |
| o         | 16                | 22            |
| r         | 0B                | 11            |
| v         | 0F                | 15            |
| w         | 0E                | 14            |
|           |                   |               |
| .         | 57                | 97            |
| =         | 44                | 68            |
|           |                   |               |
| 0         | 49                | 73            |
| 4         | 4D                | 77            |
|           |                   |               |
| B         | 3B                | 59            |
| D         | 3D                | 61            |
| F         | 3F                | 63            |
| G         | 3E                | 62            |
| L         | 35                | 53            |
| P         | 29                | 41            |
| S         | 2A                | 42            |
| U         | 2C                | 44            |

We can start to see some patterns emerging again and test out some more theories now. For example, `c` and `e` are 2 characters apart, and so we may predict that `d` will be 1B, or 27 in decimal.

This theory doesn't hold up if we look at `F` and `G`, which we may have expected to be `62` and `63` respectively as F is before G. Instead, the inverse is true.

Another observation is that characters 0 and 4 are exactly 4 apart, so 1-3 may be 74-76. Lets pick another couple to fill in some more characters and see where it leads us.

```
0B 57 34 16 1B 10 15 1C 57 38 15 0E 18 00 0A 2B 1C 0A 16 15 0F 1C 3D 1C 09 0D 11 44 49
r  .  M  o  b  i  l  e  .  A  l  w  a  y  s  R  e  s  o  l  v  e  D  e  p  t  h  =  0

0B 57 2A 11 18 1D 16 0E 28 0C 18 15 10 0D 00 44 4A
r  .  S  h  a  d  o  w  Q  u  a  l  i  t  y  =  3
```



Unfortunately this confirms that there is a pattern - but it proves that it's not as simple as one to one keycode to hexadecimal mapping. It does reveal another pattern, however:

| Character | Hexadecimal value | Decimal value |
| --------- | ----------------- | ------------- |
| r         | 0B                | 11            |
| s         | 0A                | 10            |
| t         | 0D                | 13            |
| u         | 0C                | 12            |
| v         | 0F                | 15            |
| w         | 0E                | 14            |
| x         | ??                | ??            |
| y         | 00                | 0             |
| z         | ??                | ??            |

What this reveals is that the result of _some operation_ on the character y is 0. The character y is equivalent to the decimal number 121, and as characters get closer to the keycode 121 the result approaches 0.

This is a potential indicator that the character y, 121, or 01111001 is the key for some encryption operation. So lets review some common **bitwise operators** to see if we can figure out what may have been done.

### Bitwise operations and cracking the code

These bitwise operations are commonly used for encryption, and can be used in combination with one another and multiple times in a single encryption method.

To easily evaluate bitwise operations, I use a website called [Bitwise CMD](http://bitwisecmd.com/).

#### Bit shifting

The first bitwise operator we'll look at is called bit shifting.

Bit shifting is an operation which manipulates a series of bits to produce a different result. If we take the number 12, which is expressed in binary as `00001100` we can apply a bit shifting operator left or right a certain number of places.

| Operation                   | Result (binary) | Result (decimal) |
| --------------------------- | --------------- | ---------------- |
| Shift left twice (12 << 2)  | 00110000        | **48**           |
| Shift left once (12 << 1)   | 00011000        | **24**           |
| None                        | 00001100        | **12**           |
| Shift right once (12 >> 1)  | 00000110        | **6**            |
| Shift right twice (12 >> 2) | 00000011        | **3**            |

This bit shifting operation has the same effect as multiplication and division, where `12 << 1` is the same as `12 * 2`, and in the case of shifting left, the left-most bit becomes the right-most bit and vice versa. So an operation `10000001 << 1` results in `00000011`.

#### Bitwise AND operator

AND operation compares each bit from both sides of the operation and applies the following logic to them:

| Left bit | Right bit | Resulting bit |
| -------- | --------- | ------------- |
| 0        | 0         | 0             |
| 0        | 1         | 0             |
| 1        | 0         | 0             |
| 1        | 1         | 1             |

In short, the result is 1 if both left and right bits are 1, otherwise the result is 0.

This is the same as multiplying the pairs of bits, where `1 * 1 = 1`, `1 * 0 = 0` and `0 * 0 = 0`.

As an example, lets take the numbers 123 and 234.

```asciiarmor
  123 01111011
& 234 11101010
= 106 01101010
```

#### Bitwise OR operator

OR operation compares each bit and applies the following:

| Left bit | Right bit | Resulting bit |
| -------- | --------- | ------------- |
| 0        | 0         | 0             |
| 0        | 1         | 1             |
| 1        | 0         | 1             |
| 1        | 1         | 1             |

In short, the result is 1 if either left or right bit is 1, and 0 if both are 0.

Lets use 123 and 234 as our example again.

```asciiarmor
  123 01111011 
| 234 11101010
= 251 11111011
```

#### Bitwise XOR operator

The last operator we'll look at is the XOR (eXclusive-OR) operator. Here's the logic table for XOR:

| Left bit | Right bit | Resulting bit |
| -------- | --------- | ------------- |
| 0        | 0         | 0             |
| 0        | 1         | 1             |
| 1        | 0         | 1             |
| 1        | 1         | 0             |

To summarise the XOR operator, the result is 1 if either bit is 1, but not both of them.

Applying this to 123 and 234 gives us the following result:

```asciiarmor
  123 01111011
^ 234 11101010
= 145 10010001
```

One thing that's interesting about the XOR operator is that anything XOR itself will always yield a result of 0...

```asciiarmor
  121 01111001
^ 121 01111001
= 0   00000000
```

Remembering earlier that `121` (the character y) is encrypted as 0, we can theorise that `121` is the key for a XOR operator which produces the encrypted character.

Lets pick a random character to test this, taking the character `o`, which has a decimal value of `111` and when encrypted has a value of `22`

```asciiarmor
  111 01101111 0x6f
^ 121 01111001 0x79
= 22  00010110 0x16
```

And thus we have a terrible, provable and easily reversible encryption algorithm. To get back to the original value, all we need to do is XOR the resulting `22` with the key `121` and we arrive back at `111`

```asciiarmor
  22  00010110
^ 121 01111001
= 111 01101111
```



### Writing some code to do this for me

The full code (including a build in the releases) is [available in a repository on my Github page](https://github.com/JamieDavidson/PUBGMobileConfigDecrypter)

Writing a small class in my language of choice (C#) for this algorithm is pretty straight forward:

```c#
using System;

namespace PUBGMobileConfigDecrypter
{
    internal sealed class Encryption
    {
        private const int Key = 121;
        public static char DecryptCharacter(string encryptedCharacter)
        {
            var stringAsInt = Convert.ToInt32(encryptedCharacter, 16);

            var decrypted = Key ^ stringAsInt;
            return (char)decrypted;
        }

        public static string EncryptCharacter(char plaintextCharacter)
        {
            var encrypted = Key ^ plaintextCharacter;
            return encrypted.ToString("X2");
        }
    }
}
```

`^` is the operator used for the bitwise XOR operation, where `&` is AND and `|` is OR, similar to how `&&` is logical AND, and `||` is logical OR. This is a common set of characters for a lot of languages.

Rather than lengthen this post with full code, you can view the full repository at the link above.



### Conclusion

Without using any reverse engineering techniques (which would risk infringing copyright) - this process was entirely manual investigation with predetermined input and output.

Without an example of a decrypted file, this process would have been much more challenging. If encryption had been applied from day 1, there would have been a lot more guesswork involved. An alternative would have been to use a configuration file from the steam release of PUBG, and then try to match the settings. However, odds are that the difference between the config files for the desktop release and mobile release would likely have hampered my efforts.