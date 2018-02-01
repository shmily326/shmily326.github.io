---
layout: post
title: LZ77算法的简单Python实现
categories: [Python]
description: 利用 LZ77 算法并结合 bitarray 库，对文本文件进行压缩和解压缩。
keywords: LZ77, Python, bitarray
---

# 前言
什么是LZ77算法呢？首先推荐大家学习以下两篇博文：[LZ77压缩算法编码原理详解(结合图片和简单代码)](http://www.cnblogs.com/junyuhuang/p/4138376.html)
, [【数据压缩】LZ77算法原理及实现](https://www.cnblogs.com/en-heng/p/4992916.html) 。
由于国内介绍该算法的文章比较少，因此阅读一下英文文档也大有裨益：[LZ77 Compression Algorithm](https://msdn.microsoft.com/en-us/library/ee916854.aspx) , [WiKi](https://en.wikipedia.org/wiki/LZ77_and_LZ78) 。

本文代码参考自 GitHub 上的 [A Python Compressor](https://github.com/manassra/LZ77-Compressor/) ，
由于原 repo 的采用的是 Python2.X 以及 0.8.1 版本的 bitarray ，其中某些方法已经做出改动，
故本文基于 Python3.X 以及 0.8.2 版本的 [bitarray](ilanschnell/bitarray) ，实现了对文本文件的压缩和解压缩，同时也对原 repo 中觉得不合理或是无法理解的地方做出了修改。

# 正文
## 设计思路 & 功能设计
### 压缩
>用 (p,l,c) 表示 Lookahead Buffer 中字符串的最长匹配结果，其中
> - p 表示**最长匹配**时，字典中字符开始时的位置（相对于 Cursor 位置），也即偏移量
> - l 为最长匹配字符串的长度
> - c 指 Lookahead Buffer 最长匹配结束时的下一字符
>    
>压缩的过程，就是重复输出 (p, l, c)，并将 Cursor 移动至 l+1，伪代码如下：
>```
>Repeat:
>    Output (p, l, c)
>    Cursor --> l+1
>Until to the end of string
>```
>压缩示例如图所示：
>![压缩过程图](http://p031d3a6k.bkt.clouddn.com/201801312044_139.png)

同样的，对于例子 “abcabbcaabacaacaab”，应该有输出：`(0, 0, 'a'), (0, 0, 'b'),
(0, 0, 'c'), (3, 2, 'b'), (4, 2, 'a'), (5, 1, 'a'), (5, 3, 'c'), (3, 2, 'b').`

当然，如果我们真的把原字符串中没有得到匹配的首个 `a` 用三元组 `(0, 0, 'a')` 直接代替存储的话，
显而易见地是若文本内容重复性不高时，压缩得到的文件将比未压缩的文件还要大，这是不能接受的，因此
为了尽可能地提高压缩率，我们需要定义一些规则并使用 bitarray 将三元组的内容换种形式存储。规则如下：
- 若在搜索窗口（字典）中未匹配到 Lookahead Buffer 中的子串时，就以 `0 bit + 8 bits` 表示下一个字符 c ，
其中首位的 `0` 是未匹配成功的标志。
- 若匹配到了子串，则以 `1 bit + 10 bits + 6 bits + 8 bits` 表示，其中首位的`1`是匹配
成功的标志，接着的 10 bits 用于记录偏移量 p ，接着的 6 bits 表示匹配到的子串的长度 l ，
最后 8 bits 表示下一个字符 c 。

如此一来，可见若文本内容重复性越高，压缩后的文件相比于原始文件节约的存储空间则越多，因此压缩率将会更高。

### 最长匹配
可以发现，在上述压缩过程中最重要的是找到 Lookahead Buffer 中相应于搜索窗口里的最长匹配子串。
这里需要注意的是偏移量 p 小于 匹配字符串长度 l 的情况，也即出现了循环复制，例如：

`acbacbacba` 对应输出为 `(0, 0, 'a'), (0, 0, 'c'), (0, 0, 'b'), (3, 4, 'c'), (6, 2, '')`

根据前三个三元组我们可以得到 `abc` ，接着 `(3, 4, 'c')` 表示向左偏移 3 位，复制 4 位，
在最后结果加上字符 `c` 。目前我们只有长度为 3 的字符串，却要向后要复制 4 位，这直观上看来
似乎是不符合逻辑的，但确实是可行的。当你复制 3 位以后将得到 `acbacb` ，而第四位正是复制得到的
第二个 `a` ，所以最终得到 `acbacbac` ，这便是一个循环复制的过程。而对于 `p==0 & l==0`
以及 `p >= l` 的情况都很容易理解，这里不再赘述。

根据 Lookahead Buffer 中选取的子串长度（递增序），在搜索窗口中构造用于与子串相比较的字符串，
然后再将得到的字符串与选取的子串相比较，若相同则更新 (p, l, c)，接着增加子串长度并重复以上步骤，
这便是选取最长匹配的方法。

### 解压
在解压缩时，对于三元组 (p, l, c) 有如下三种情况（设输出字符串为 `out`）：
1. `p==0 & l==0` ，即没有匹配项，此时直接在输出接上字符 `c` 即可
2. `p >= l` ，则此时应在输出接上 `out[-p : -p+l]`
3. `p < l` ，则此时应在输出接上 `l` 次 `out[-p]`（`out` 每加一次都在变长）

实际上我们发现情况 2 和 3 是等价的，同时三元组也是以 01's 的形式存储的，因此我们只需判断
首位的标志变量位是 0 还是 1 ，就可以对其后的若干位做出相应的判断和处理，也即解压过程。

## 代码实现

``` python
# coding: utf-8

from bitarray import bitarray
import os

class LZ77:

    max_window_size = 1023 # 0b1111111111
    max_buffer_size = 63 # 0b111111

    def __init__(self, window_size=6, buffer_size=4):
        self.window_size = min(window_size, self.max_window_size) # 搜索窗口大小
        self.buffer_size = min(buffer_size, self.max_buffer_size) # Lookahead Buffer 大小

    def max_match(self, cursor, data):
        '''
        cursor：指向搜索窗口末尾元素， data：待压缩字符串
        return：返回三元组
        '''
        p = -1  # 偏移量
        l = -1  # 复制长度
        c = ''  # 下一字符
        end = min(cursor+self.buffer_size, len(data))
        for i in range(cursor+1, end+1):
            substring = data[cursor+1:i+1]
            start = max(0, cursor-self.window_size+1)
            for j in range(start, cursor+1):
                # 构造待匹配的字符串
                repetition_times = len(substring)//(cursor-j+1)
                repetition_count = len(substring)%(cursor-j+1)
                str_to_check = data[j:cursor+1]*repetition_times + \
                               data[j:j+repetition_count]
                # 为了提高压缩率，可以修改 len(substring) > 1 其中的 1
                if str_to_check == substring and len(substring)>1:
                    p = cursor - j + 1
                    l = len(substring)
                    # 防止直接访问 a[i+1]越界
                    c = data[i+1] if len(data)>=i+2 else ''
        # 若未匹配成功，直接返回下一字符
        if p == -1 and l == -1:
            return (0, 0, data[cursor+1])
        return (p, l, c)

    def compress(self, input_file_path = '', output_file_path = ''):
        '''
        input_file：待压缩字符文本
        output_file：压缩后的文件
        0 bit + 8 bits 下一字符
        1 bit + 10 bits 偏移量 + 6 bits 长度 + 8 bits 下一字符
        '''

        try:
            with open(input_file_path, 'r') as f:
                data = f.read()
        except IOError:
            print("Could Not Open Input File !")
            raise

        out = bitarray(endian = "big")  # 大端模式
        cursor = -1
        while cursor < len(data) - 1:
            (p, l, c) = self.max_match(cursor, data)
            if p:
                out.append(True)
                '''
                    不可修改：bytes([162]) = b'\xa2'  bytes(3) = b'\x00\x00\x00'
                             bytes([1,2, ord('1'),ord('2')]) = b'\x01\x0212'
                    可修改：  b = bytearray(2)
                '''
                # 将 10 bits 偏移量 + 6 bits 长度 共 16 bits 以 2个字节存储
                out.frombytes(bytes([p >> 2]))
                out.frombytes(bytes([((p & 0x03) << 6) | l]))
            if not p:
                out.append(False)
            if c:   # 确保c非空串，以免 ord(c)出错
                out.frombytes(bytes([ord(c)]))
            cursor += (l + 1) # 这里多加 1，即 c 的长度
        out.fill()  # 末尾补充0，凑足整字节以方便读取

        if output_file_path:
            try:
                with open(output_file_path, 'wb') as f:
                    out.tofile(f)
                    print("Compressed And Saved Successfully !")
                return None
            except IOError:
                print("Could Not Open Output File !")
                raise
        return out

    def decompress(self, input_file_path = '', output_file_path = ''):
        '''
        input_file: 二进制文件
        output_file：原字符文本
        return：解压后的字符串
        '''
        data = bitarray(endian='big')
        out = ''

        try:
            with open(input_file_path, 'rb') as f:
                data.fromfile(f)
        except IOError:
            print("Could Not Open Input File !")
            raise

        while len(data) >= 9:
            flag = data.pop(0)
            # 判断首位标志变量
            if flag:
                # ord(b'\x71') = 113 = b'\x71'[0]
                byte1 = ord(data[0:8].tobytes())
                byte2 = ord(data[8:16].tobytes())
                next_ch = data[16:24].tobytes().decode()
                p = (byte1 << 2) | (byte2 >> 6)
                l = byte2 & 0x3f
                del data[0:24]
                # 循环复制
                for i in range(l):
                    out += out[-p]
                out += next_ch
            else:
                out += data[0:8].tobytes().decode()
                del data[0:8]

        if output_file_path:
            try:
                with open(output_file_path, 'w') as f:
                    f.write(out)
                    print("Decompressed And Saved Successfully !")
                return None
            except IOError:
                print("Could Not Open Output File !")
        return out

if __name__ == "__main__":
    c = LZ77(window_size = 100, buffer_size = 30)
    filename = "test"
    c.compress(filename + ".txt", filename + "_compressed.bin")
    c.decompress(filename + "_compressed.bin", filename + "_decompressed.txt")

    with open(filename + ".txt", "r") as f:
        data_before = f.read()
    with open(filename + "_decompressed.txt", "r") as result:
        data_after = result.read()

    print(data_before == data_after)
    size_before = os.path.getsize(filename + ".txt")
    size_after = os.path.getsize(filename + "_compressed.bin")
    print("Compression Ratio: {0:.2f}%".format(size_after/size_before*100))

```

## 运行结果 & 分析
下面比较不同的搜索窗口大小（window_size）以及不同的 Lookahead Buffer 大小（buffer_size）
对测试文本压缩率的影响（ ps ：压缩后能成功复原原文本；pps ：压缩率是文件压缩后的大小与压缩前的大小之比）：

测试文本段 1 ：
>mahi magi magi mahi mahi hello mahi madi mahi mahi mahi facebook hello world mahi magi magi mahi mahi hello mahi madi mahi mahi mahi facebook hello world mahi magi magi mahi mahi hello mahi madi mahi mahi mahi facebook hello world

| window_size\buffer_size |     5      |      20      |      40      |      63      |
| :---------------------: |:----------:|:------------:|:------------:|:------------:|
|          20             |   80.17%   |     65.52%   |    65.52%    |    65.52%    |
|          100            |   60.34%   |     30.17%   |    25.86%    |    24.57%    |
|          500            |   60.34%   |     30.17%   |    25.86%    |    24.57%    |
|          1023           |   60.34%   |     30.17%   |    25.86%    |    24.57%    |

测试文本段 2 ：
>Tomorrow will be better. Good morning, ladies and gentlemen, I' m very happy to be making a speech here. Today my topic is " Tomorrow will be better. " China born and China bred, I love China very much. I' m proud that I have got the beautiful yellow skin, black eyes and black hair. I' m also proud that I speak the most beautiful language in the world Chinese. When I heard that China, our motherland, would have a chance to hold the 2008 Olympic Games, I felt very happy and excited, and I hoped I could do something useful to build China into a beautiful and energetic country. In the 21st century, the environment is becoming more and more important, so we have " Green Olympic" as a slogan. Of course a slogan is just a goal. The most important thing is that we should do some things to make it true. Such as sorting the trash, saving the energy and so on. We are still students now, so we can' t do anything really big, but if everybody does something good for our environment, we could make our motherland more beautiful. In fact, the things we can do are easy. Like not littering used batteries everywhere, sorting the trash that we want to throw away, and also protecting the animals and plants around us. Luckily, I had a chance to take part in an activity, that is, we planted 5 trees everybody in our campus. My classmates and I worked very hard. This activity is not only about planting trees, but also contributing to our environment. Look at the trees, I believe, tomorrow there will be more trees and flowers standing in our campus and giving Chinese people a beautiful environment. We are all Chinese people, living on the earth. We have only one motherland China, just like we have only one earth. If we don't beautify our environment, who will? Good environment depends on good human consciousness. We should say that protecting our surroundings is not easy, that' s why I' m standing here to summon people do it. If we do this from the bottom of our hearts tomorrow, China will be more brilliant, it will have blue skies, white clouds and cleaner rivers. Who doesn' t want China to become more and more beautiful? If we do our best, tomorrow will be better! Thank you for your listening! Thank you all!

| window_size\buffer_size |     5      |      20      |      40      |      63      |
| :---------------------: |:----------:|:------------:|:------------:|:------------:|
|          20             |   106.97%  |     106.25%  |    106.25%   |    106.25%   |
|          100            |   95.33%   |     92.94%   |    92.94%    |    92.94%    |
|          500            |   81.03%   |     74.83%   |    74.79%    |    74.79%    |
|          1023           |   76.49%   |     69.62%   |    69.57%    |    69.57%    |

根据上面的测试结果，我们可以发现：
 - window_size 、buffer_size 越大，压缩效果越明显。但对于一段确定的文本，window_size / buffer_size 大到一定程度之后，压缩率将不再变化。
 - 对于内容连续重复性不高的文本来说，较小的 window_size 、buffer_size 会导致压缩后的文件反而比原文件大。
 - 更改压缩过程中定义的 bitarray 存储三元组信息的规则将得到不同的压缩效果。

# 总结
本文使用 Python 语言，并利用 bitarray 库实现了简单的 LZ77 压缩算法的功能。我们知道现在流行的一种压缩算法 ZIP 就是基于 LZ77 的思想演变过来的，但 ZIP 对 LZ77 编码之后的结果又继续进行压缩，直到难以压缩为止。LZ77 算法利用的是 The Same-as-Earlier Trick ,而另一种无损压缩的技巧是 The Shorter-Symbol Trick，其典型的代表就是今天在通信和数据存储系统中得到了广泛应用的霍夫曼编码，正是结合了多种技术和思想才构成了使我们受益的各种成熟、高效的压缩算法。
