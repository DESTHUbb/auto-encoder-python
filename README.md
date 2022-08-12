# auto-encoder-python
 Base64 is the way that 8-bit binary data is encoded into a format that can be represented in 7 bits. This is done by using only the characters A-Z, a-z, 0-9, +, and / in order to represent the data.   

## Theory of operation of the tool.
Here I show you with Python 3 how a basic JPEG decoder works.

## Different parts of JPEG

Let's start with an image made by Ange Albertini. List all the parts of a simple JPEG file. We'll go through each segment, and as you read the article, you'll come back to this illustration more than once.

![image](https://user-images.githubusercontent.com/90658763/184354679-a2e68787-1b84-4a7a-b9cf-ec6d09b11b01.png)

Almost all binary files contain multiple markers (or headers). You can think of them as a kind of bookmarks. They are essential for working with the file and are used by programs like file (on Mac and Linux) so that we can learn the details of the file. Bookmarks indicate exactly where certain information is stored in the file. Most of the time, markers are placed according to the length value of a particular segment.


## Let's write some code to find the start and end markers.

```python
from struct import unpack

marker_mapping = {
    0xffd8: "Start of Image",
    0xffe0: "Application Default Header",
    0xffdb: "Quantization Table",
    0xffc0: "Start of Frame",
    0xffc4: "Define Huffman Table",
    0xffda: "Start of Scan",
    0xffd9: "End of Image"
}

class JPEG:
    def __init__(self, image_file):
        with open(image_file, 'rb') as f:
            self.img_data = f.read()
    
    def decode(self):
        data = self.img_data
        while(True):
            marker, = unpack(">H", data[0:2])
            print(marker_mapping.get(marker))
            if marker == 0xffd8:
                data = data[2:]
            elif marker == 0xffd9:
                return
            elif marker == 0xffda:
                data = data[-2:]
            else:
                lenchunk, = unpack(">H", data[2:4])
                data = data[2+lenchunk:]            
            if len(data)==0:
                break        

if __name__ == "__main__":
    img = JPEG('profile.jpg')
    img.decode()    

# OUTPUT:
# Start of Image
# Application Default Header
# Quantization Table
# Quantization Table
# Start of Frame
# Huffman Table
# Huffman Table
# Huffman Table
# Huffman Table
# Start of Scan
# End of Image
```

### To uncompress the bytes of the image, we use struct. >H tells struct to read the short unsigned data type and work with it in order from largest to smallest (big-endian). JPEG data is stored in the format from largest to smallest. Only EXIF ​​data can be in little-endian format (although this is often not the case). And the size is short 2, so we transfer unpack two bytes of img_data. How did we know what is short? We know that markers that are in JPEG are indicated by four hexadecimal characters: ffd8. One of those characters is equal to four bits (half a byte), so four characters will equal two bytes, as short.

### The Scan Start section is immediately followed by the scanned image data, which does not have a specific length. They continue to the end of the file marker, so for now we "seek" it manually when we find the Start Scan marker.

### Now let's deal with the rest of the image data. To do this, we first study the theory, and then move on to programming.

# JPEG encoding

## First, let's talk about the basic concepts and encoding techniques used in JPEG. And the encoding will be done in reverse order. In my experience, the decryption will be difficult to crack without it.



### The following illustration is not clear to you yet, but I will give you hints as you learn the encoding and decoding process. Here are the steps for JPEG encoding (source):

![image](https://user-images.githubusercontent.com/90658763/184356155-a6e50e58-3c22-4bce-ab79-327e1547eccc.png)

## JPEG color space


According to the JPEG specification (ISO/IEC 10918-6:2013(E) Section 6.1):



Images encoded with a single component are treated as grayscale data, where 0 is black and 255 is white.
Images encoded with three components are considered to be RGB data encoded in YCbCr space. If the image contains the APP14 marker segment described in Section 6.5.3, then the color coding is considered RGB or YCbCr according to the APP14 marker segment information. The relationship between RGB and YCbCr is defined according to ITU-T T.871 | ISO/IEC 10918-5.
, , CMYK-, (0,0,0,0) . APP14, 6.5.3, CMYK YCCK APP14. CMYK YCCK 7.


Most implementations of the JPEG algorithm use luminance and chrominance (YUV encoding) instead of RGB. This is very useful because the human eye is very poor at distinguishing high-frequency changes in brightness in small areas, so you can lower the frequency and the person won't notice the difference. What are you doing? Highly compressed image with almost imperceptible quality degradation.



As in RGB, each pixel is encoded with three color bytes (red, green and blue), so in YUV three bytes are used, but their meaning is different. The Y component defines the brightness of the color (luminance or luma). U and V define the color (chroma): U is responsible for the blue portion and V - for the red portion.



This format was developed at a time when television was not yet so common, and engineers wanted to use the same image encoding format for color and black-and-white television transmission. Read more about it here.



Discrete cosine transform and quantization


JPEG converts the image into 8x8 blocks of pixels (called MCU, Minimum Encoding Unit), shifts the range of pixel values ​​so that the center is 0, then applies a discrete cosine transform to each block and compresses the result using quantization . Let's see what all this means.



The discrete cosine transform (DCT) is a method of transforming discrete data into combinations of cosine waves. Converting an image to a set of cosines seems like a waste of time at first glance, but you'll understand why when you learn the following steps. DCT takes an 8x8 pixel block and tells us how to render that block using an 8x8 matrix of cosine functions. More details here.



The array looks like this:

![image](https://user-images.githubusercontent.com/90658763/184360846-58eb3ca5-3172-48c2-93a9-ecec6b8ed262.png)

## We apply DCT separately to each pixel component. As a result, we get an 8x8 matrix of coefficients, showing the contribution of each (of the 64) cosine functions in the input 8x8 matrix. In the DCT coefficient matrix, the largest values ​​are usually in the upper left corner and the smallest in the lower right corner. The top left is the lowest frequency cosine function and the bottom right is the highest.



This means that in most images there is a large amount of low-frequency information and a small proportion of high-frequency information. If the lower right components of each DCT matrix are assigned a value of 0, then the resulting image will look the same to us, because a person does not distinguish high-frequency changes poorly. This is what we will do in the next step.



I found a great video on this topic. Watch if you don't understand the meaning of PrEP:

https://www.youtube.com/watch?v=Q2aEzeMDHMA&t=2s

### We all know that JPEG is a lossy compression algorithm. But so far we haven't lost anything. We only have 8x8 YUV component blocks converted to 8x8 cosine function blocks without loss of information. The data loss stage is quantification.



Quantization is the process where we take two values ​​from a certain range and convert them to a discrete value. In our case, this is just a clever name for reducing the highest frequency coefficients in the resulting DCT matrix to 0. When saving an image with JPEG, most graphics editors ask you what level of compression you want to set. This is where the loss of high-frequency information occurs. You will no longer be able to recreate the original image from the resulting JPEG image.



Different quantization matrices are used depending on the compression ratio (fun fact: most developers have patents to create a quantization table). We divide the element-by-element DCT matrix of coefficients by the quantization matrix, round the results to integers, and obtain the quantized matrix.



Let's see an example. Let's say there is such a DCT matrix:

![image](https://user-images.githubusercontent.com/90658763/184361357-a0f2e019-7c8b-444b-8e83-7f0bc0fe8c2d.png)

## And here is the usual quantization matrix:

![image](https://user-images.githubusercontent.com/90658763/184361485-6e4057f8-0824-4826-b2a3-9cd06671d522.png)

## The quantized array will look like this:

![image](https://user-images.githubusercontent.com/90658763/184361693-a62b01e0-cadd-40c0-8810-c326f66136b8.png)

## Although the high-frequency information cannot be seen by humans, if you remove too much data from the 8x8 pixel blocks, the image will look too coarse. In such a quantized array, the first value is called the DC value and all others are called AC values. If we take the DC values ​​of all the quantized arrays and generate a new image, we would get a preview with a resolution 8 times smaller than the original image.



I also want to point out that since we use quantization, we need to make sure the colors are within the range of [0.255]. If they fly away, you'll have to manually bring them into this range.



## Zig Zag


After quantization, the JPEG algorithm uses a zigzag scan to convert the array to a one-dimensional form:

![image](https://user-images.githubusercontent.com/90658763/184361967-9591f37e-0d66-461c-b99d-d6d79cdc6af3.png)

## Let's have such a quantized matrix:

![image](https://user-images.githubusercontent.com/90658763/184362099-8332c254-26e1-445c-a0c9-b1dd3d25d8c3.png)

## So the result of the zigzag scan will be as follows:

```python
[15 14 13 12 11 10 9 8 0  ...  0]
```

## This encoding is preferred because after quantization, most of the low-frequency information (the most important) will be located at the beginning of the array, and the zigzag scan stores this data at the beginning of the one-dimensional array. This is useful for the next step, compression.



Run-length encoding and delta encoding


Run-length encoding is used to compress repetitive data. After the zigzag scan, we see that there are mostly zeros at the end of the array. Length encoding allows us to take advantage of this wasted space and use fewer bytes to represent all those zeros. Let's say we have the following data:

```python
10 10 10 10 10 10 10
```


After encoding the string lengths, we get this:


```python
7 10
```

We have compressed 7 bytes into 2 bytes.



Delta encoding is used to represent a byte relative to the previous byte. It will be easier to explain with an example. Let's have the following data:


```python
10 11 12 13 10 9
```


Using delta encoding, they can be represented as follows:


```python
10 1 2 3 0 -1
```

## In JPEG, each DC value in the DCT array is delta encoded relative to the previous DC value. This means that changing the first DCT coefficient of the image will destroy the entire image. But if you change the first value of the last DCT array, it will only affect a very small portion of the image.



This approach is useful because the first CC value in the image usually varies the most, and we use delta encoding to bring the remaining CC values ​​closer to 0, which improves compression with Huffman encoding.

## Huffman coding

It is a lossless compression method. Huffman once wondered, "What is the fewest number of bits I can use to store free text?" As a result, an encoding format was created. Let's say we have text:

```python
a B C D E
```

Normally, each character occupies one byte of space:


```python
to: 01100001
second: 01100010
c: 01100011
d: 01100100
e: 01100101
```


This is the principle of ASCII binary encoding. What if we change the mapping?

```python
# mapping

000: 01100001
001: 01100010
010: 01100011
100: 01100100
011: 01100101
```


Now we need a lot less bits to store the same text:

```python
to: 000
b:001
c:010
re: 100
my: 011
```


That's great, but what if we need to save even more space? For example, like this:


```python
# mapping
0: 01100001
1: 01100010
00:01100011
01: 01100100
10: 01100101
```

```python
to: 0
second: 1
c:00
day: 01
W: 10
```

## Huffman encoding allows such a variable-length match. The input data is taken, the most common characters are combined with a smaller combination of bits and the less frequent characters with larger combinations. And then the resulting mappings are collected into a binary tree. In JPEG, we store DCT information using Huffman coding. Remember I mentioned that delta encoding DC values ​​make Huffman encoding easy? I hope now you understand why. After delta encoding, we need to match fewer "characters" and the size of the tree is reduced.



Tom Scott has a great video explaining how the Huffman algorithm works. Take a look before reading on.

https://youtu.be/JsTptu56GM8

## A JPEG file contains up to four Huffman tables, which are stored in "Define Huffman Table" (starts with 0xffc4). The DCT coefficients are stored in two different Huffman tables: in one, DC values ​​from zigzag tables, in the other, AC values ​​from zigzag tables. This means that when encoding, we need to combine the DC and AC values ​​of the two arrays. The DCT information for the luminance and chromaticity channels is stored separately, so we have two sets of DC information and two sets of AC information, a total of 4 Huffman tables.



If the image is grayscale, then we only have two Huffman tables (one for DC and one for AC), because we don't need color. As you may have discovered, two different images can have very different Huffman tables, so it's important to store them within each JPEG.



Now we know the main content of JPEG images. Let's move on to decoding.

## JPEG decoding


Decoding can be divided into stages:



1- Huffman table extraction and bit decoding.
2- Extraction of DCT coefficients using run-length and delta encoding reversal.
3- Using DCT coefficients to combine cosine waves and reconstruct pixel values ​​for each 8x8 block.
4- Convert YCbCr to RGB for each pixel.
5- Shows the resulting RGB image.


The JPEG standard supports four compression formats:



➡ Base

➡ extended series

➡ Progressive

➡ No loss


We will work with basic compression. It contains a series of 8x8 blocks, one after another. In other formats, the data template is slightly different. To make it clear, I've colored in different segments of the hex content of our image:

![image](https://user-images.githubusercontent.com/90658763/184365001-9edda44a-bf01-41ba-8336-2eeb115979cd.png)

## Extraction of Huffman tables


We already know that JPEG contains four Huffman tables. This is the last encoding, so we'll start decoding with it. Each section with the table contains information:

### Field---------------------Field----------------Description

Marker ID---------------------2 bytes--------------0xff and 0xc4 identify DHT

Length------------------------2 bytes--------------Table length

Huffman table information-----1 byte bits----------0 ... 3: number of tables (0 ... 3, otherwise error) bit 4: type of table, 0 = DC table, 1 = AC tables bits 5 ... 7: not used, must be 0

Characters--------------------16 bytes-------------The number of characters with codes of length 1 ... 16, sum (n) of these bytes is the total number of codes that must be <= 256

Symbol------------------------n bytes--------------The table contains characters in order of increasing code length (n = total number of codes).

## Let's say we have a Huffman table like this (source):

### Symbol----Huffman code----Code length

         a------00----------------2
         
    second-----010----------------3
    
         C-----011----------------3
    
        re-----100----------------3
    
        my-----101----------------3
    
        F------110----------------3
    
     gram-----1110----------------4
    
        h-----11110---------------five
    
        I----111110---------------6
    
        j---1111110---------------7
    
        k---11111110--------------8
    
        l--111111110--------------nine


### It will be stored in a JFIF file like this (in binary format. I'm using ASCII just for clarity):


```python
0 1 5 1 1 1 1 1 1 0 0 0 0 0 0 0 a b c d e f g h i j k l
```


0 means we don't have a Huffman code of length 1. A 1 means we have a Huffman code of length 2, and so on. In the DHT section, immediately after the class and ID, the data is always 16 bytes long. Let's write the code to extract lengths and elements from DHT.

```python
### class JPEG:
    # ...
    
    def decodeHuffman(self, data):
        offset = 0
        header, = unpack("B",data[offset:offset+1])
        offset += 1

        # Extract the 16 bytes containing length data
        lengths = unpack("BBBBBBBBBBBBBBBB", data[offset:offset+16]) 
        offset += 16

        # Extract the elements after the initial 16 bytes
        elements = []
        for i in lengths:
            elements += (unpack("B"*i, data[offset:offset+i]))
            offset += i 

        print("Header: ",header)
        print("lengths: ", lengths)
        print("Elements: ", len(elements))
        data = data[offset:]

    
    def decode(self):
        data = self.img_data
        while(True):
            # ---
            else:
                len_chunk, = unpack(">H", data[2:4])
                len_chunk += 2
                chunk = data[4:len_chunk]

                if marker == 0xffc4:
                    self.decodeHuffman(chunk)
                data = data[len_chunk:]            
            if len(data)==0:
break
```

### After running the code, we get this:

```python
Start of Image
Application Default Header
Quantization Table
Quantization Table
Start of Frame
Huffman Table
Header:  0
lengths:  (0, 2, 2, 3, 1, 1, 1, 0, 0, 0, 0, 0, 0, 0, 0, 0)
Elements:  10
Huffman Table
Header:  16
lengths:  (0, 2, 1, 3, 2, 4, 5, 2, 4, 4, 3, 4, 8, 5, 5, 1)
Elements:  53
Huffman Table
Header:  1
lengths:  (0, 2, 3, 1, 1, 1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0)
Elements:  8
Huffman Table
Header:  17
lengths:  (0, 2, 2, 2, 2, 2, 1, 3, 3, 1, 7, 4, 2, 3, 0, 0)
Elements:  34
Start of Scan
End of Image
```

### Excellent! We have lengths and elements. Now you need to create your own Huffman table class to reconstruct a binary tree from the obtained lengths and elements. I'll copy the code from here:

```python
class HuffmanTable:
    def __init__(self):
        self.root=[]
        self.elements = []
    
    def BitsFromLengths(self, root, element, pos):
        if isinstance(root,list):
            if pos==0:
                if len(root)<2:
                    root.append(element)
                    return True                
                return False
            for i in [0,1]:
                if len(root) == i:
                    root.append([])
                if self.BitsFromLengths(root[i], element, pos-1) == True:
                    return True
        return False
    
    def GetHuffmanBits(self,  lengths, elements):
        self.elements = elements
        ii = 0
        for i in range(len(lengths)):
            for j in range(lengths[i]):
                self.BitsFromLengths(self.root, elements[ii], i)
                ii+=1

    def Find(self,st):
        r = self.root
        while isinstance(r, list):
            r=r[st.GetBit()]
        return  r 

    def GetCode(self, st):
        while(True):
            res = self.Find(st)
            if res == 0:
                return 0
            elif ( res != -1):
                return res
                
class JPEG:
    # -----

    def decodeHuffman(self, data):
        # ----
        hf = HuffmanTable()
        hf.GetHuffmanBits(lengths, elements)
        data = data[offset:]
```

### GetHuffmanBits Takes the length and elements across elements and places them in a root list. It is a binary tree and contains nested lists. You can read on the internet how the Huffman tree works and how to create such a data structure in Python. For our first DHT (from the image at the beginning of the article) we have the following data, lengths and elements:

```python
Hex Data: 00 02 02 03 01 01 01 00 00 00 00 00 00 00 00 00
Lengths:  (0, 2, 2, 3, 1, 1, 1, 0, 0, 0, 0, 0, 0, 0, 0, 0)
Elements: [5, 6, 3, 4, 2, 7, 8, 1, 0, 9]
```
### After the call, the root GetHuffmanBitslist will contain the following data:


```python
[[5, 6], [[3, 4], [[2, 7], [8, [1, [0, [9]]]]]]]

```

### HuffmanTable also contains a GetCode method that walks the tree and uses a Huffman table to return the decoded bits. The method receives a stream of bits as input, just a binary representation of the data. For example, the bit stream for abc would be 011000010110001001100011. First, we convert each character to its ASCII code, which we convert to binary. Let's create a class that helps us convert a string to bits and then count the bits one by one:

```python
class Stream:
    def __init__(self, data):
        self.data= data
        self.pos = 0

    def GetBit(self):
        b = self.data[self.pos >> 3]
        s = 7-(self.pos & 0x7)
        self.pos+=1
        return (b >> s) & 1

    def GetBitN(self, l):
        val = 0
        for i in range(l):
            val = val*2 + self.GetBit()
        return val
```

When initializing, we give the class binary data and then read it using GetBity GetBitN.



Decoding of the quantization table


According to the JPEG standard, the default JPEG image has two quantization tables: for luma and chroma. They start with a 0xffdb marker. The result of our code already contains two of those markers. Let's add the ability to decode quantization tables:


```python
def GetArray(type,l, length):
    s = ""
    for i in range(length):
        s =s+type
    return list(unpack(s,l[:length]))
  
class JPEG:
    # ------
    def __init__(self, image_file):
        self.huffman_tables = {}
        self.quant = {}
        with open(image_file, 'rb') as f:
            self.img_data = f.read()

    def DefineQuantizationTables(self, data):
        hdr, = unpack("B",data[0:1])
        self.quant[hdr] =  GetArray("B", data[1:1+64],64)
        data = data[65:]

    def decodeHuffman(self, data):
        # ------ 
        for i in lengths:
            elements += (GetArray("B", data[off:off+i], i))
            offset += i 
            # ------

    def decode(self):
        # ------
        while(True):
            # ----
            else:
                # -----
                if marker == 0xffc4:
                    self.decodeHuffman(chunk)
                elif marker == 0xffdb:
                    self.DefineQuantizationTables(chunk)
                data = data[len_chunk:]            
            if len(data)==0:
                break        
```

We first define a GetArray method. It's just a convenient technique for decoding a variable number of bytes from binary data. We've also replaced some of the code in the decodeHuffman method to take advantage of the new feature. Then the DefineQuantizationTables method was defined: it reads the title of the section with quantization tables, and then applies the quantization data to the dictionary using the header value as the key. this value can be 0 for luminance and 1 for chromaticity. Each section with quantization tables in JFIF contains 64 bytes of data (for an 8x8 quantization matrix).



If we generate the quantization matrices for our image, we get this:

```python
3    2    2    3    2    2    3    3   
3    3    4    3    3    4    5    8   
5    5    4    4    5    10   7    7   
6    8    12   10   12   12   11   10  
11   11   13   14   18   16   13   14  
17   14   11   11   16   22   16   17  
19   20   21   21   21   12   15   23  
24   22   20   24   18   20   21   20  

3     2    2    3    2    2    3    3
3     2    2    3    2    2    3    3
3     3    4    3    3    4    5    8
5     5    4    4    5    10   7    7
6     8    12   10   12   12   11   10
11    11   13   14   18   16   13   14
17    14   11   11   16   22   16   17
19    20   21   21   21   12   15   23
24    22   20   24   18   20   21   20
```

### Decoding the start of the frame

Here we are not interested in everything. We will extract the width and height of the image, as well as the number of quantization tables for each component. The width and height will be used to start decoding the actual scans of the image from the Start Scan section. Since we will be working primarily with a YCbCr image, we can assume that there will be three components, and their types will be 1, 2, and 3, respectively. Let's write the code to decode this data:

```python
class JPEG:
    def __init__(self, image_file):
        self.huffman_tables = {}
        self.quant = {}
        self.quantMapping = []
        with open(image_file, 'rb') as f:
            self.img_data = f.read()
    # ----
    def BaselineDCT(self, data):
        hdr, self.height, self.width, components = unpack(">BHHB",data[0:6])
        print("size %ix%i" % (self.width,  self.height))

        for i in range(components):
            id, samp, QtbId = unpack("BBB",data[6+i*3:9+i*3])
            self.quantMapping.append(QtbId)         
    
    def decode(self):
        # ----
        while(True):
                # -----
                elif marker == 0xffdb:
                    self.DefineQuantizationTables(chunk)
                elif marker == 0xffc0:
                    self.BaselineDCT(chunk)
                data = data[len_chunk:]            
            if len(data)==0:
                break        
```

We add a list attribute to the quantMapping JPEG class and define a BaselineDCT method that decodes the required data from the SOF section and places the number of quantization tables for each component in the quantMapping list. We'll build on this when we start reading the Starting the Analysis section. This is what our quantMapping photo looks like:


```python
Quant mapping: [0, 1, 1]
```

### Scan decode start

This is the "meat" of a JPEG image, it contains the data for the image itself. We have reached the most important stage. Everything we decoded before can be considered a card that helps us decode the image itself. This section contains the image itself (encoded). We read the section and decrypt using the already decrypted data.

All markers started with 0xff. This value can also be part of the scanned image, but if it is present in this section, it will always be followed by y 0x00. The JPEG encoder inserts it automatically, this is called byte stuffing. Therefore, the decoder must remove them 0x00. Let's start with the SOS decode method that contains such a function and remove the existing 0x00. In our image in the section with a scan there is no 0xffbut it is still a useful addition.

```python
def RemoveFF00(data):
    datapro = []
    i = 0
    while(True):
        b,bnext = unpack("BB",data[i:i+2])        
        if (b == 0xff):
            if (bnext != 0):
                break
            datapro.append(data[i])
            i+=2
        else:
            datapro.append(data[i])
            i+=1
    return datapro,i

class JPEG:
    # ----
    def StartOfScan(self, data, hdrlen):
        data,lenchunk = RemoveFF00(data[hdrlen:])
        return lenchunk+hdrlen
      
    def decode(self):
        data = self.img_data
        while(True):
            marker, = unpack(">H", data[0:2])
            print(marker_mapping.get(marker))
            if marker == 0xffd8:
                data = data[2:]
            elif marker == 0xffd9:
                return
            else:
                len_chunk, = unpack(">H", data[2:4])
                len_chunk += 2
                chunk = data[4:len_chunk]
                if marker == 0xffc4:
                    self.decodeHuffman(chunk)
                elif marker == 0xffdb:
                    self.DefineQuantizationTables(chunk)
                elif marker == 0xffc0:
                    self.BaselineDCT(chunk)
                elif marker == 0xffda:
                    len_chunk = self.StartOfScan(data, len_chunk)
                data = data[len_chunk:]            
            if len(data)==0:
                break
```

Previously, we manually searched for the end of the file when we encountered a 0xffda marker, but now that we have a tool that allows us to systematically view the entire file, we'll move the marker search condition inside the else operator. The RemoveFF00 function simply breaks when it finds something else instead of 0x00 after 0xff. The loop breaks when the function finds 0xffd9, so we can search for the end of the file without fear of surprises. If you run this code, you won't see anything new in the terminal.



Remember, JPEG divides the image into 8x8 matrices. Now we need to convert the scanned image data to a bitstream and process it into 8x8 chunks. Let's add the code to our class:

```python
class JPEG:
    # -----
    def StartOfScan(self, data, hdrlen):
        data,lenchunk = RemoveFF00(data[hdrlen:])
        st = Stream(data)
        oldlumdccoeff, oldCbdccoeff, oldCrdccoeff = 0, 0, 0
        for y in range(self.height//8):
            for x in range(self.width//8):
                matL, oldlumdccoeff = self.BuildMatrix(st,0, self.quant[self.quantMapping[0]], oldlumdccoeff)
                matCr, oldCrdccoeff = self.BuildMatrix(st,1, self.quant[self.quantMapping[1]], oldCrdccoeff)
                matCb, oldCbdccoeff = self.BuildMatrix(st,1, self.quant[self.quantMapping[2]], oldCbdccoeff)
                DrawMatrix(x, y, matL.base, matCb.base, matCr.base )    
        
        return lenchunk +hdrlen
```

Let's start by converting the data into a stream of bits. Remember that delta encoding is applied to the DC element of the quantization matrix (its first element) relative to the previous DC element? So, we initialize oldlumdccoeff, oldCbdccoeff, and oldCrdccoeff with zero values, they will help us keep track of the value of previous DC elements, and it will default to 0 when we encounter the first DC element.



The for loop may seem strange. self.height//8 tells us how many times we can divide the height by 8 8, and it's the same with the width and self.width//8.



BuildMatrix will take a quantization table and add parameters, create an inverse discrete cosine transform matrix and give us the Y, Cr and Cb matrices. And the DrawMatrix function converts them to RGB.



First, let's create an IDCT class, and then start completing the BuildMatrix method.

```python
import math

class IDCT:
    """
    An inverse Discrete Cosine Transformation Class
    """

    def __init__(self):
        self.base = [0] * 64
        self.zigzag = [
            [0, 1, 5, 6, 14, 15, 27, 28],
            [2, 4, 7, 13, 16, 26, 29, 42],
            [3, 8, 12, 17, 25, 30, 41, 43],
            [9, 11, 18, 24, 31, 40, 44, 53],
            [10, 19, 23, 32, 39, 45, 52, 54],
            [20, 22, 33, 38, 46, 51, 55, 60],
            [21, 34, 37, 47, 50, 56, 59, 61],
            [35, 36, 48, 49, 57, 58, 62, 63],
        ]
        self.idct_precision = 8
        self.idct_table = [
            [
                (self.NormCoeff(u) * math.cos(((2.0 * x + 1.0) * u * math.pi) / 16.0))
                for x in range(self.idct_precision)
            ]
            for u in range(self.idct_precision)
        ]

    def NormCoeff(self, n):
        if n == 0:
            return 1.0 / math.sqrt(2.0)
        else:
            return 1.0

    def rearrange_using_zigzag(self):
        for x in range(8):
            for y in range(8):
                self.zigzag[x][y] = self.base[self.zigzag[x][y]]
        return self.zigzag

    def perform_IDCT(self):
        out = [list(range(8)) for i in range(8)]

        for x in range(8):
            for y in range(8):
                local_sum = 0
                for u in range(self.idct_precision):
                    for v in range(self.idct_precision):
                        local_sum += (
                            self.zigzag[v][u]
                            * self.idct_table[u][x]
                            * self.idct_table[v][y]
                        )
                out[y][x] = local_sum // 4
        self.base = out
```

Let's take a look at the IDCT class. When we extract the MCU, it will be stored in a base attribute. We then transform the MCU array by backward zigzag scanning using the rearrange_using_zigzag method. Finally, we will reverse the discrete cosine transform by calling the perform_IDCT method.



As you may recall, the CC table is fixed. We will not consider how DCT works, this is more related to mathematics than programming. We can store this table as a global variable and query for pairs of x,y values. I decided to put the table and its calculation in an IDCT class so that the text would be readable. Each element of the transformed MCU array is multiplied by an idc_variable value and we get the values ​​Y, Cr and Cb.



This will become clearer when we add the BuildMatrix method.



If you modify the zigzag table to look like this:

```python
[[ 0,  1,  5,  6, 14, 15, 27, 28],
[ 2,  4,  7, 13, 16, 26, 29, 42],
[ 3,  8, 12, 17, 25, 30, 41, 43],
[20, 22, 33, 38, 46, 51, 55, 60],
[21, 34, 37, 47, 50, 56, 59, 61],
[35, 36, 48, 49, 57, 58, 62, 63],
[ 9, 11, 18, 24, 31, 40, 44, 53],
[10, 19, 23, 32, 39, 45, 52, 54]]
```

### get this result (note the small artifacts):

```python
[[12, 19, 26, 33, 40, 48, 41, 34,],
[27, 20, 13,  6,  7, 14, 21, 28,],
[ 0,  1,  8, 16,  9,  2,  3, 10,],
[17, 24, 32, 25, 18, 11,  4,  5,],
[35, 42, 49, 56, 57, 50, 43, 36,],
[29, 22, 15, 23, 30, 37, 44, 51,],
[58, 59, 52, 45, 38, 31, 39, 46,],
[53, 60, 61, 54, 47, 55, 62, 63]]
```

## So the result will be like this:

![image](https://user-images.githubusercontent.com/90658763/184373131-9bc4ecf2-eaab-4ba3-98fe-66895b4ac726.png)

## Let's complete our BuildMatrix method:

```python
def DecodeNumber(code, bits):
    l = 2**(code-1)
    if bits>=l:
        return bits
    else:
        return bits-(2*l-1)
      
class JPEG:
    # -----
    def BuildMatrix(self, st, idx, quant, olddccoeff):
        i = IDCT()

        code = self.huffman_tables[0 + idx].GetCode(st)
        bits = st.GetBitN(code)
        dccoeff = DecodeNumber(code, bits) + olddccoeff

        i.base[0] = (dccoeff) * quant[0]
        l = 1
        while l < 64:
            code = self.huffman_tables[16 + idx].GetCode(st)
            if code == 0:
                break

            # The first part of the AC quantization table
            # is the number of leading zeros
            if code > 15:
                l += code >> 4
                code = code & 0x0F

            bits = st.GetBitN(code)

            if l < 64:
                coeff = DecodeNumber(code, bits)
                i.base[l] = coeff * quant[l]
                l += 1

        i.rearrange_using_zigzag()
        i.perform_IDCT()

        return i, dccoeff
```

We start by creating an inverted discrete cosine transform class ( IDCT()). We then read the data into the bit stream and decode it using a Huffman table.



self.huffman_tables[0]and self.huffman_tables[1]see the CC tables for luma and chroma, respectively, and self.huffman_tables[16]and self.huffman_tables[17]see the CA tables for luma and chroma, respectively .



After decoding the bitstream, we use a DecodeNumber function to extract the new delta-encoded DC coefficient and add it to olddccoefficient to get the delta-decoded DC coefficient.



We then repeat the same decoding procedure with the CA values ​​in the quantization matrix. Meaning of code 0 indicates that we have reached the End of Block (EOB) marker and we must stop. Also, the first part of the AC quantization table tells us how many leading zeros we have. Now let's remember how to encode the string lengths. Let's reverse this process and skip all those bits. IDCTs are explicitly assigned zeros in the class.



After decoding the DC and AC values ​​for the MCU, we convert the MCU and reverse the zigzag scan by calling rearrange_using_zigzag. And then we invert the DCT and apply it to the decoded MCU.



The BuildMatrix method will return the inverted DCT matrix and the value of the DC coefficient. Remember that this will be an array for only a minimum 8x8 encoding unit. Let's do this for all the other MCUs in our archive.



## Displaying an image on the screen


Now let's have our code in the StartOfScan method create a Tkinter Canvas and draw each MCU after decoding.

```python
def Clamp(col):
    col = 255 if col>255 else col
    col = 0 if col<0 else col
    return  int(col)

def ColorConversion(Y, Cr, Cb):
    R = Cr*(2-2*.299) + Y
    B = Cb*(2-2*.114) + Y
    G = (Y - .114*B - .299*R)/.587
    return (Clamp(R+128),Clamp(G+128),Clamp(B+128) )
  
def DrawMatrix(x, y, matL, matCb, matCr):
    for yy in range(8):
        for xx in range(8):
            c = "#%02x%02x%02x" % ColorConversion(
                matL[yy][xx], matCb[yy][xx], matCr[yy][xx]
            )
            x1, y1 = (x * 8 + xx) * 2, (y * 8 + yy) * 2
            x2, y2 = (x * 8 + (xx + 1)) * 2, (y * 8 + (yy + 1)) * 2
            w.create_rectangle(x1, y1, x2, y2, fill=c, outline=c)

class JPEG:
    # -----
    def StartOfScan(self, data, hdrlen):
        data,lenchunk = RemoveFF00(data[hdrlen:])
        st = Stream(data)
        oldlumdccoeff, oldCbdccoeff, oldCrdccoeff = 0, 0, 0
        for y in range(self.height//8):
            for x in range(self.width//8):
                matL, oldlumdccoeff = self.BuildMatrix(st,0, self.quant[self.quantMapping[0]], oldlumdccoeff)
                matCr, oldCrdccoeff = self.BuildMatrix(st,1, self.quant[self.quantMapping[1]], oldCrdccoeff)
                matCb, oldCbdccoeff = self.BuildMatrix(st,1, self.quant[self.quantMapping[2]], oldCbdccoeff)
                DrawMatrix(x, y, matL.base, matCb.base, matCr.base )    
        
        return lenchunk+hdrlen
      

if __name__ == "__main__":
    from tkinter import *
    master = Tk()
    w = Canvas(master, width=1600, height=600)
    w.pack()
    img = JPEG('profile.jpg')
    img.decode()    
    mainloop()
```

Let's start with the ColorConversion and Clamp functions. ColorConversion takes the Y, Cr, and Cb values, converts them to RGB components using the formula, and then outputs the aggregate RGB values. Why add 128 to them? Remember, before the JPEG algorithm, DCT is applied to the MCU and subtracts 128 from the color values. If the colors were originally in the range [0.255], JPEG will put them in the range [-128, +128]. We need to reverse this effect when decoding, so we add 128 to RGB. As for Clamp, when unpacking, the resulting values ​​can go outside the range [0.255], so we keep them in this range [0.255].



In the DrawMatrix method we loop through each decoded 8x8 matrix for Y, Cr, and Cb, and convert each element of the matrix to RGB values. After transforming, we draw pixels on the Tkinter canvas using the create_rectangle method. You can find all the code on GitHub. If you run it, my face will appear on the screen.


