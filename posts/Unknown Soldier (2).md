---
title: Unknown Soldier 2
description: A steganography challenge concisting on fixing a PNG image by hand. 
date: '03/22/2024'
icon: https://wiki.nobrackets.fr/img/logo.png
author: Kalitsune
tags:
  - steganography 
  - ctf
published: true
---

<details>
<summary>Original Challenge description 🇫🇷</summary>
<blockquote>
En ouvrant la boîte du soldat inconnu, qui n'est plus si inconnu que ça, vous tombez sur une image.  
Selon la description, il s'agit d'une photo du soldat fraîchement enrôlé.  
Vous vous dites que cela serait un beau geste que de l'envoyer à sa famille en tant que dernier souvenir.  
Malheureusement, elle a été endommagée, essayez de la réparer avant de l'envoyer.
</blockquote>
</details>

<details open>
<summary>Challenge description 🇺🇸</summary>
<blockquote>
When you open the box of the not-so-unknown soldier, you come across an image.
According to the description, it's a photo of the freshly enlisted soldier.
You think it would be a nice gesture to send it to his family as a last memento.
Unfortunately, it's been damaged, so you try to repair it before sending it.
</blockquote>
</details>
# Write up

## Challenge inspection
The challenge isn't necessarily provided with any file, however, in Unknown Soldier (1) we got a file named `picture_of_me.png`. This file appears to be an image, however there's one small issue: the file is not readable.

![The photo](/blog/NBCTF/Error%20File%20corrupted.png)

![Meme computer guy thinking](/blog/NBCTF/Computer%20think.png)

This is strange but not unexpected, we're in a challenge of steganography after all. 
I continue my inspection by using `pngcheck` to try to find the issue.
```sh
❯ pngcheck picture_of_me.png 
picture_of_me.png  invalid IHDR image dimensions (25231972x0)
ERROR: picture_of_me.png
```
It appears that the image has invalid dimensions.

## Investigating the issue with the image header
To check what's wrong with the header, you'll need to open the file using an hex editor.

I'm personally using GHex, a GNOME hex editor.

The first line of our file looks like this:
##### <p><span style="color:#c00000">89 50 4E 47 0D 0A 1A 0A</span> <span style="color:#ffc000">00 00 00 0D</span> <span style="color:#7030a0">49 48 44 52</span> <span style="color:#00b050">01 81 02 64 00 00 00 00 08 06 00 00 00</span> <span style="color:#00b0f0">F7 38 9D 88</span> 00 00 00 04 67</p>

![Meme stressing out that we don't have much time left to solve this challenge](/blog/NBCTF/So%20much%20data%20So%20little%20time.png)

I took the liberty to color the bits of data that are relevant.

Okay so, what are we seeing?

According to https://en.wikipedia.org/wiki/List_of_file_signatures, the signature for the PNG file format is <span style="color:#c00000">89 50 4E 47 0D 0A 1A 0A</span>. Which is  great because that's exactly what we have! This mean that we are indeed looking at a PNG file.

After googling for a bit I also found the [PNG Format Specifications](http://www.libpng.org/pub/png/spec/1.2/PNG-Structure.html) according to which, a PNG chunk of data is organised this way:

<span style="color:#ffc000">[Length (4 byte)]</span> <span style="color:#7030a0">[Chunk Type (4 byte)]</span> <span style="color:#00b050">[Data (any)]</span> <span style="color:#00b0f0">[CRC (4 byte)]</span>

The length value is not really useful for now so let's skip it and check what's the chunk type.
I was able to find that the signature of my chunk ( <span style="color:#7030a0">49 48 44 52</span> ), matched the signature of an <span style="color:#7030a0">IHDR</span> Image header.
Which is really interesting because according to the [PNG Format Specs](http://www.libpng.org/pub/png/spec/1.2/PNG-Chunks.html, the <span style="color:#7030a0">IHDR</span> chunk is the one containing the image <span style="color:#00b050">size information</span>.

Let's take a closer look to the data in this chunk then:

| <span style="color:#00b050">DATA</span>  | <span style="color:#00b050">Width</span> | <span style="color:#00b050">Height</span> | <span style="color:#00b050">Bit depth</span> | <span style="color:#00b050">Color type</span> | <span style="color:#00b050">Compression method</span> | <span style="color:#00b050">Filter method</span> | <span style="color:#00b050">Interlace method</span> |
|-------|-------|--------|-----------|------------|--------------------|---------------|------------------|
| Bytes |   4   |   4    |     1     |      1     |          1         |       1       |        1         |

In this field we find that the dimensions of this file are indeed corrupted. The rest of the data looks fine.

To fix them we'll have to use the <span style="color:#00b0f0">CRC</span> bytes which allows partial repair in case of damage.

![Always read the docs](/blog/NBCTF/Read%20the%20docs.png)
## Getting the headers dimensions back
As mentioned earlier, the <span style="color:#00b0f0">CRC</span> field may just be able to help us repair this image as it is just like a hash and we only have two values to find.

We'll just have to brute force the Height and Width of the image until we find a <span style="color:#00b0f0">CRC</span> that match:
```py
from binascii import crc32

# Put your CRC as bytes here:
correct_crc = int.from_bytes(b'\xf7\x38\x9d\x88',byteorder='big')

for h in range(2000):
    for w in range(2000):
	    #      IHDR HEADER               WIDTH                        HEIGHT               REMAINING DATA (healthy)
        crc=b"\x49\x48\x44\x52"+w.to_bytes(4,byteorder='big')+h.to_bytes(4,byteorder='big')+b"\x08\x06\x00\x00\x00"
	    # If it match the CRC
        if crc32(crc) % (1<<32) == correct_crc:
            print ('FOUND!')
            print ('Width: ',end="")
            print (hex(w))
            print ('Height :',end="")
            print (hex(h))
            exit()
```

And hooray! We found something:
```txt
FOUND!
Width: 0x181
Height :0x264
```

This mean that by jumping back into the hex editor and editing the Width and Height fields, we can finally fix this image! Let's edit our header right away:
##### <p><span style="color:#c00000">89 50 4E 47 0D 0A 1A 0A</span> <span style="color:#ffc000">00 00 00 0D</span> <span style="color:#7030a0">49 48 44 52</span> <span style="color:#00b050">00 00 <u>01 81</u> 00 00 <u>02 64</u> 08 06 00 00 00</span> <span style="color:#00b0f0">F7 38 9D 88</span> 00 **00** 00 04 67</p>

![We fixed it!](/blog/NBCTF/fixed.gif)

The only thing left to do is save.

I ran a quick pngcheck just to be sure:
```sh
❯ pngcheck fixed.png 
OK: fixed.png (385x612, 32-bit RGB+alpha, non-interlaced, 55.7%).
```

And everything seemed good! 

I then opened the file and was greeted by this:
![The image obtained](/blog/NBCTF/Unknown%20solider%20fixed.png)

Awesome! We did it! The flag is:
```txt
NBCTF{m0n_b3ll3_un1f0rm3}
```

As we say in french: Champagne!
![champagne!](/blog/NBCTF/champagne.gif)
