+++
title = 'Waifu2x android，这才是我的“本科毕业设计” - 前言+SWIG篇'
image = 'cover.png'
date = 2021-12-22T09:05:00+08:00
lastmod = 2022-01-02T17:11:00+08:00
draft = false
categories = ['Tech']
tags = [
    'Waifu2x',
    'ncnn',
    'SWIG',
    'Python',
    'C',
    'CMake',
    'FFI',
    'binding'
]
+++

# Waifu2x android，这才是我的“本科毕业设计” - 前言+SWIG篇

## Changelog
> 2021/12/27
> 对指针binding部分添加了C++部分的源码，更正了buffer interface带来的性能提升的说法，补充说明了Mat对应的Wrapper。

## 前言 （发牢骚，不想看的话可以跳过去）
### 一切的开始
就是忽然觉得必须要写点什么。

一是因为今年初，成功将nihui的waifu2x-ncnn-vulkan的Python Binding做出来，积攒了一堆还未整理的笔记。（SWIG Binding里面的 ~~坑~~ 要点还挺多的）然后，一直没整理发出来；

二是最近Waifu2x ncnn Android终于公开发布了，还是有些内容想要公开的，比如毕竟很多人不是很清楚怎么将ncnn部署到Android上，而且也没对应的demo（毕竟不能只是白嫖nihui的项目）也因此，想起我的Waifu2x Android企划其实早就在本科时候就有了，只不过过去了有些年头，一时间就忘记了。
早在2016年的时候，Waifu2x刚出来一会就听说了它。那时候使用的还是Waifu2x-caffe。所幸，当时我用的游戏本就是N卡的，可以用CUDA来算，所以体验还不错，同时也被结果惊到了。那是我第一次和Waifu2x接触。也因此，对机器学习产生了兴趣，去学了Coursera上面Andrew Ng的Machine Learning课，并期待之后能自研算法。

### 第一次尝试
回想起来，那个时候刚开始并没有部署到移动端的打算。只是因为有了跟着Gustav做数据分析的经验（实际并没有做啥深度的），想着复现校验SRCNN的内容，并自己训练一个模型出来玩玩。同时，因为我也快要毕业设计选题了，当时就想和导师提能不能让我做这个。结果被打了回去，说这种项目给读研的人做，你就做个什么水印管理系统就得了。（艹）于是，直到现在我还是对此非常怨念的。

~~你不让我做是吧？我自己做~~ 

于是就单干了，最后在2018年三月左右搞出了第一版模型。当时的训练代码，毕业时公开在了Github。项目名称叫[SRCNN-Keras](https://github.com/ArchieMeng/SRCNN_keras)。

在那段时间，紧跟着Apple推出CoreML，Google也推出了自己的移动平台机器学习框架:Tensorflow-lite。毕竟名字带个tensorflow，我以为用它的话，说不定能非常方便地将我自己训练的模型丢到Android上使用。然而我错了。。。

首先，编译tensorflow-lite就够我吃一壶了。那个bazel编译编半天编不出来;然后，Android开发我还是新手，只有帮Fython修修Bug打杂的那点经验。模型转换也不会。所以，遇到了这么多难以解决的困难，我放弃了。

### 转机
如果说到促成Waifu2x ncnn开发契机的话，那大概就以下几个：

- Nihui的waifu2x-ncnn-vulkan
- 因为不满当时Python的视频超解析软件还用着拆帧成图片而不是流式处理方式而做的anime2x-multibackend
- 和Video2x的开发者安利自己的流式处理方法，发现急切需要有个Python调用waifu2x-ncnn-vulkan的库，不然调用效率过差，所以就有了[waifu2x-ncnn-vulkan-python](https://github.com/media2x/waifu2x-ncnn-vulkan-python)

#### [waifu2x-ncnn-vulkan](https://github.com/nihui/waifu2x-ncnn-vulkan): 
让只要有Vulkan API支持的设备都能跑起Waifu2x的算法。ncnn让部署深度学习模型不再局限于老黄家的CUDA。同时，也是Waifu2x ncnn移植的程序。
#### [anime2x-multibackend](https://github.com/ArchieMeng/anime2x-multibackend):
试验Python流式处理视频且支持自定义视频处理器的项目。后面促成的waifu2x-ncnn-vulkan-python的诞生。
#### [waifu2x-ncnn-vulkan-python](https://github.com/media2x/waifu2x-ncnn-vulkan-python)
waifu2x-ncnn-vulkan的Python binding。支持Waifu2x Object等级的初始化，处理，析构控制。比就像是直接调用脚本式的程序效率高多了。（[Video2x](https://github.com/k4yt3x/video2x)和anime2x战术后仰）
同时，也因为这个项目，有了waifu2x-ncnn-vulkan的项目结构了解，更加方便写binding并移植到其他平台或语言了。也是想到这一点，才逐渐有了做Waifu2x ncnn的想法的。

下面开始技术分享环节。

## *-ncnn-vulkan-python系列项目中使用的SWIG binding笔记和要点

### 相关技术与取舍
实际上实现C-Python Binding的方式有很多种，裸的C binding，boost，ctypes, cython，pybind11等等。除了SWIG以外的大多数方法，基本都在[Python Bindings: Calling C or C++ From Python](https://realpython.com/python-bindings-overview/)和[Building a Python C extension module with CMake](https://martinopilia.com/posts/2018/09/15/building-python-extension.html)里讲清楚了。(可惜就是没有SWIG的详细介绍，这也是笔者觉得有必要写本文的原因)
在waifu2x-ncnn-vulkan-python中，由于上游项目是使用CMake做Build system的，所以能方便地支持CMake就是第一考量，然后才是Binding的易用性。所以，最后入法眼的只有SWIG。（个人觉得Pybind11的语法还是稍微复杂了点，不像SWIG直接写）同时，也因为SWIG是通用的Binding语言，所以还给*-ncnn-vulkan-python项目提供其他语言的Binding创造了可能。（不过这是后话了，暂时也没用上这点）

### 最简单的SWIG样例
(假设读者已经安装好了SWIG和编译环境)

当时上手参考的demo项目: [swig-example](https://github.com/danielunderwood/swig-example)

参考文档：http://www.swig.org/Doc4.0/Introduction.html#Introduction_nn4


```C
/* File : example.c */

double My_variable = 3.0;

/* Compute factorial of n */
int fact(int n) {
  if (n <= 1)
    return 1;
  else
    return n*fact(n-1);
}

/* Compute n mod m */
int my_mod(int n, int m) {
  return(n % m);
}
```

```C
/* File : example.i */
%module example
%{
/* 在此放头文件和声明 */
extern double My_variable;
extern int    fact(int);
extern int    my_mod(int n, int m);
%}

/* 放实际做Binding的接口 */
extern double My_variable;
extern int    fact(int);
extern int    my_mod(int n, int m);
```

可以看出，SWIG最基础的就是在*.i的接口文件中定义目标binding的接口就行了。

而对于C++的，需要特殊处理，比如Template。(虽然这部分在waifu2x-ncnn-vulkan-python里面没用到，但是早期试验SWIG binding的时候遇到了)

参考文档：http://www.swig.org/Doc4.0/SWIGPlus.html#SWIGPlus_nn30

样例项目：[DictionaryTree](https://github.com/ArchieMeng/DictionaryTree)

```C++
/* WordSolver.h */
class WordSolver : public DictionaryTree {
public:
    vector<string> solve(vector<vector<char>>);
};
```

```C
// dictionary_tree.i

// to use std::vector
%include "std_vector.i"
namespace std{
    %template(VecChar) vector<char>;
    %template(VecVecChar) vector<vector<char>>;
    %template(VecString) vector<string>;
};
%{
    #include "DictionaryTree.h"
    #include "WordSolver.h"
%}

%include "DictionaryTree.h"
%include "WordSolver.h"
```

同时，通过这个例子，也可以看出，在SWIG中可以直接包含头文件来声明原生C接口，所以非常方便。不过，也不是所有场景只include头文件就能解决的。比如在waifu2x-ncnn-vulkan中，ncnn::Mat太复杂了，于是就不做它的binding了，所以Waifu2x类的接口就选择手写指定的形式，而不是直接include "waifu2x.h"。例子如下：

```C
%{
    #include "waifu2x.h"
    #include "waifu2x_wrapped.h"
%}

class Waifu2x
{
    public:
        Waifu2x(int gpuid, bool tta_mode = false, int num_threads = 1);
        ~Waifu2x();

    public:
        // waifu2x parameters
        int noise;
        int scale;
        int tilesize;
        int prepadding;
};

%include "waifu2x_wrapped.h"
```
其中，原来的Waifu2x类只做了一些必要API的接口binding，而Mat实际使用自定义的Image结构体代替。

### 定义指针binding
在waifu2x-ncnn-vulkan-python中遇到了个棘手的问题，那就是Waifu2x::load()在Windows和Unix系统下的参数(char*和wchar*)是不一样的，然而根据我提的[issue](https://github.com/swig/swig/issues/1954)中得到的信息，就是SWIG会对所有平台采用同样的Binding接口，所以我原本想的在不同平台生成不同接口的做法是不可行的。最后，我采用了Union指针的方式去做。而指针在SWIG中是没法直接Binding的，要用特殊的函数去生成管理。（毕竟C中指针的生命周期也需要手动管理）

用法参考链接：http://www.swig.org/Doc4.0/SWIGDocumentation.html#Library_nn3

代码来自waifu2x-ncnn-vulkan-python

```C++
// waifu2x_wrapped.h line 21-37
union StringType {
    std::string *str;
    std::wstring *wstr;
};

class Waifu2xWrapped : public Waifu2x
{
  public:
    Waifu2xWrapped(int gpuid, bool tta_mode = false, int num_threads = 1);
    int load(const StringType &parampath, const StringType &modelpath);
    int process(const Image &inimage, Image &outimage) const;
    int process_cpu(const Image &inimage, Image &outimage) const;
    uint32_t get_heap_budget();

  private:
    int gpuid;
};
```

```C++
// waifu2x_wrapped.cpp line 34-42

int Waifu2xWrapped::load(const StringType &parampath,
                         const StringType &modelpath)
{
#if _WIN32
    return Waifu2x::load(*parampath.wstr, *modelpath.wstr);
#else
    return Waifu2x::load(*parampath.str, *modelpath.str);
#endif
}
```

```C
// waifu2x.i
%pointer_functions(std::string, str_p);
%pointer_functions(std::wstring, wstr_p);
```

```Python
# waifu2x_ncnn_vulkan.py#Waifu2x.load()
param_path_str, model_path_str = wrapped.StringType(), wrapped.StringType()
if sys.platform in ("win32", "cygwin"):
    param_path_str.wstr = wrapped.new_wstr_p()
    wrapped.wstr_p_assign(param_path_str.wstr, str(param_path))
    model_path_str.wstr = wrapped.new_wstr_p()
    wrapped.wstr_p_assign(model_path_str.wstr, str(model_path))
else:
    param_path_str.str = wrapped.new_str_p()
    wrapped.str_p_assign(param_path_str.str, str(param_path))
    model_path_str.str = wrapped.new_str_p()
    wrapped.str_p_assign(model_path_str.str, str(model_path))

self._waifu2x_object.load(param_path_str, model_path_str)
```

通过这个例子，也可以说明使用Union指针的方式可能是解决SWIG binding跨平台不同接口的一个方案。如果有更好的，也希望和笔者讨论补充。:)到waifu2x-ncnn-vulkan-python的discuss里面提也是欢迎的。

### 传递数组
通常做C binding，就是因为需要高性能处理，而这个时候通常会需要传递大数组。这里，笔者目前就用过两种方法，如果还有的话欢迎补充。
#### Unbounded C Arrays
参考链接： http://www.swig.org/Doc4.0/Python.html#Python_nn48
其实就是调用SWIG的API，在Python中生成C数组，然后往里面填内容就好了。适合一般情况，需要手动循环生成数组内容的情况。缺点很明显，就是循环调用效率低下，特别是在大数组，图像处理的时候，特别明显。在[waifu2x-ncnn-vulkan-python的早期版本](https://github.com/ArchieMeng/waifu2x-ncnn-vulkan-python/tree/018079422e15e5a7dd4e7e3d74b4f323424252f0)中，就是使用的这种binding。结果就是相比原版慢大约20%。

```C++
// waifu2x_wrapped.h

// wrapper class of ncnn::Mat
typedef struct Image {
    unsigned char *data;
    int w;
    int h;
    int elempack;
    Image(unsigned char *d, int w, int h, int channels)
    {
        this->data = d;
        this->w = w;
        this->h = h;
        this->elempack = channels;
    }

} Image;
```

```C++
//waifu2x_wrapped.cpp line 1-11

#include "waifu2x_wrapped.h"

int Waifu2xWrapped::process(const Image &inimage, Image &outimage) const
{
    int c = inimage.elempack;
    ncnn::Mat inimagemat =
        ncnn::Mat(inimage.w, inimage.h, (void *)inimage.data, (size_t)c, c);
    ncnn::Mat outimagemat =
        ncnn::Mat(outimage.w, outimage.h, (void *)outimage.data, (size_t)c, c);
    return Waifu2x::process(inimagemat, outimagemat);
}
```

```C
// waifu2x.i

%module waifu2x_ncnn_vulkan_wrapper

%include "carrays.i"
%include "std_string.i"
%include "stdint.i"

%array_class(unsigned char, PixelBuffer);

%{
    #include "waifu2x.h"
    #include "waifu2x_wrapped.h"
%}

%include "waifu2x.h"
%include "waifu2x_wrapped.h"
```

```python
def process(self, im: Image) -> Image:
    """
    Waifu2x process incoming PIL.Image
    :param im: input PIL.Image
    :return: result PIL.Image
    """
    in_bytes = im.tobytes()
    in_buffer = raw.PixelBuffer(len(in_bytes))
    channels = int(len(in_bytes) / (im.width * im.height))
    out_buffer = raw.PixelBuffer((self._raw_w2xobj.scale ** 2) * len(in_bytes))

    for i, b in enumerate(in_bytes):
        in_buffer[i] = b

    raw_in_image = raw.Image(in_buffer, im.width, im.height, channels)
    raw_out_image = raw.Image(out_buffer, self._raw_w2xobj.scale * im.width, self._raw_w2xobj.scale * im.height, channels)

    if self.gpuid != -1:
        self._raw_w2xobj.process(raw_in_image, raw_out_image)
    else:
        self._raw_w2xobj.tilesize = max(im.width, im.height)
        self._raw_w2xobj.process_cpu(raw_in_image, raw_out_image)

    out_bytes = bytes(map(lambda i: out_buffer[i], range((self._raw_w2xobj.scale ** 2) * len(in_bytes))))

    return Image.frombytes(im.mode, (self._raw_w2xobj.scale * im.width, self._raw_w2xobj.scale * im.height), out_bytes)
```

#### Buffer interface
参考链接：http://www.swig.org/Doc4.0/SWIGDocumentation.html#Python_nn75

当Python部分要传的数组是可以序列化成Bytes的时候，就可以直接使用Buffer interface来传递。这正好是Waifu2x-ncnn-vulkan-python这种使用PIL.Image来表示图片的场景。（可以调用Image.tobytes()来拿到Bitmap的Bytes)

```C
// waifu2x.i
%include "pybuffer.i"

%pybuffer_mutable_string(unsigned char *d);
```

```Python
def process(self, image: Image) -> Image:
        """
        Process the incoming PIL.Image

        :param im: PIL.Image
        :return: PIL.Image
        """
        in_bytes = bytearray(image.tobytes())
        channels = int(len(in_bytes) / (image.width * image.height))
        out_bytes = bytearray((self._waifu2x_object.scale ** 2) * len(in_bytes))

        raw_in_image = wrapped.Image(in_bytes, image.width, image.height, channels)
        raw_out_image = wrapped.Image(
            out_bytes,
            self._waifu2x_object.scale * image.width,
            self._waifu2x_object.scale * image.height,
            channels,
        )

        if self._gpuid != -1:
            self._waifu2x_object.process(raw_in_image, raw_out_image)
        else:
            self._waifu2x_object.tilesize = max(image.width, image.height)
            self._waifu2x_object.process_cpu(raw_in_image, raw_out_image)

        return Image.frombytes(
            image.mode,
            (
                self._waifu2x_object.scale * image.width,
                self._waifu2x_object.scale * image.height,
            ),
            bytes(out_bytes),
        )
```

无需调用特别函数，直接将Bytes传入即可。也是因此，waifu2x-ncnn-vulkan-python不仅达到了无性能损失的效果，甚至在视频处理场景，因为不需要进行频繁的图片编解码，所以还会比原版还快一点点。

### 其他常见问题

#### -fexception
"-fexception"是编译SWIG binding必须的Flag，而在waifu2x-ncnn-vulkan中，可能是因为要最小化编译，所以关掉了。在waifu2x-ncnn-vulkan-python中，需要开启。

#### uint8_t & uint32_t支持
参考链接：https://stackoverflow.com/questions/10476483/how-to-generate-a-cross-platform-interface-with-swig

当Binding中存在unsigned int的时候（unsigned char也是），需要在接口文件中包含stdint.i

#### 避免链接特定版本的Python lib
参考链接：

- [Problems with Python library target (new FindPython module) when the library is statically linked into the interpreter](https://gitlab.kitware.com/cmake/cmake/-/issues/18100)
- [CMake - FindPython](https://cmake.org/cmake/help/git-stage/module/FindPython.html)

在waifu2x-ncnn-vulkan-python早期打包发布的时候，遇到过编译二进制在一些机器上找不到Python库的问题。经过排查，用ldd发现它们都链接上了特定版本的Python库。所以为了解决这些，请参考waifu2x-ncnn-vulkan-python的CMakeFileLists.txt以及上述参考链接。

## 未完待续：Waifu2x ncnn的开发过程和细节介绍
写到这里，感觉有点长了。之后再另外写一个分享Waifu2x ncnn的开发过程。尽情期待吧
