# Node.js Color Detection using the Raspberry Pi Camera

## Hardware

I am using a [Raspberry Pi 2 B](https://www.raspberrypi.org/products/raspberry-pi-2-model-b/) running Raspbian Jessie and a [Raspberry Pi Camera](https://www.raspberrypi.org/products/camera-module/). I am writing the code on a Mac connected to the Pi over ssh.

## Software

### Node.js

I recommend getting started using Node.js on the Raspberry Pi with this [AdaFruit tutorial](https://learn.adafruit.com/node-embedded-development/installing-node-dot-js) except don't install it like they do. Apt-get packages for Node.js are way out of date. I have found that [n for Node.js version management](https://github.com/tj/n) works well to install current versions of Node.js on the Raspberry Pi.

### [Atom.io](http://atom.io/)

I am using GitHub's Atom.io code editor on Mac to write the JavaScript. I recommend the [remote-atom](https://atom.io/packages/remote-atom) plugin to easily sync the code files to the Raspberry Pi. Use ratom to ssh in like so:

    ssh -R 52698:localhost:52698 pi@raspberrypi

## Performance

If you are curious about how much this visual processing in Node.js asks of your hardware resources, I checked. The *viewserver.js* process (which imports *Senses.js* which imports *frogeye.js*) puts my Raspberry Pi 2 at about 4.0% Cpu with negligible RAM usage. This is at a time lapse camera setting of 0 (take pictures as fast as possible, minimum 30ms) and sending sense data to the client every 20ms (50 times per second.) Here's the top output if you don't believe me.

<img src="img/topfrogeye.png" alt"Top output of viewserver.js">

The view client runs on a different machine so its usage is not reflected in the above data.

### Reverse-Engineering the Y'UV Image Format

The Raspberry Pi camera can output uncompressed files in [Y'UV format](https://en.wikipedia.org/wiki/YUV) (or YUV or YCbCr) that are easier to analyze than JPEGs, once you know what all the ones and zeros mean. I had some fun using [Hex Fiend](http://ridiculousfish.com/hexfiend/) to edit the raw files, then converted them to JPEG with [ImageMagick](http://www.imagemagick.org/script/index.php) with this command:

    convert -size "64x48" -depth 8 yuv:test.raw conv.jpg

Then used Apple Preview to see how I mangled the picture.

#### Summary of Y'UV

I am using a 64 pixel wide by 48 pixel high image, so let's assume that for my examples. This generates a file 4608 bytes in size (1.5 * 64 * 48) where the first 2/3 is a simple black-and-white bitmap and the last 1/3 is color information. How it stores the image dimensions is a mystery. Magic maybe?

I am digging YUV format for image processing. It is not only easy to use, but the format was designed to store and transmit only the information that human eyes care most about. Brightness levels are the majority the file, while color information is simplified and lower resolution.

##### Y Component

The first part of the binary data of a `raspiyuv` capture is 3072 (64 x 48) 8-bit values (0x00-0xFF, or 0-255 in decimal) that represent the Y, or luma, component of the image where 0x00 is the darkest and 0xFF is the brightest value. This is what I am using for most of the image analysis.

##### UV Component

The color information is 1/2 the resolution of the luma component. So in my example, it is a 32 pixel x 24 pixel bitmap. Also, I am used to an RGB color space, but UV represents color using only 2 components instead of three somehow.

<img alt="UV color plane at Y=0.5" src="https://upload.wikimedia.org/wikipedia/commons/f/f9/YUV_UV_plane.svg" title="By Tonyle - Own work, CC BY-SA 3.0, https://commons.wikimedia.org/w/index.php?curid=6977944" width="300" height="300">

##### Convert to HSL

When I started working with recognizing colors, it became quickly apparent that I needed a a mathematical representation of color in a way that makes sense to a human. YUV is an efficient way to communicate color to a machine, but not very useful when you want to identify things that are "red". Hue/Saturation/Luminance is a color scheme that makes more sense to people. I was unable to find any clear method to convert between the two even though it looks like there should be a simple relationship. Luckily [this article](http://www.quasimondo.com/archives/000696.php) pointed out what should have been obvious by just looking at the UV color space image above: If you draw a line from the origin to a (u, v) value, you get an arrow that points to a hue, and the length of that arrow is the saturation. I converted UV value of 0-255 to a hue value of 0.0-1.0 like so:

    function uvToHue(u, v) {
        var normalU = (2 * u / 255) - 1.0,
            normalV = (2 * v / 255) - 1.0;

        return (Math.atan2(normalV, normalU) + Math.PI) / (Math.PI * 2);
    }

Whenever I use `atan2` my result always seems to be off by 90 degrees from what I think it should be, and this result is no different. (Hue value 0.0 is green instead of red.) I don't think it matters for my application, as long as the number relates to a specific color so I'm not going to worry too much about it at the moment.
