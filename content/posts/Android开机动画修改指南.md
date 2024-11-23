---
tags:
  - android/framework
title: Android开机动画修改指南
date: 2024-01-29
---

> 本文内容的基本骨架来源于对 frameworks/base/cmds/bootanimation/FORMAT.md 的整合翻译。以下介绍的所有内容只适用于 AOSP ，厂商可针对 BootAnimation.cpp 进行魔改，请以具体代码为准。

## 开机动画文件路径

系统按优先级顺序从以下路径选择归档为 zip 文件的开机动画，第一个路径的优先级最高，然后依次递减：

```txt
/apex/com.android.bootanimation/etc/bootanimation.zip (since Android 10)
/product/media/bootanimation.zip (since Android 9)
/oem/media/bootanimation.zip
/system/media/bootanimation-encrypted.zip (if getprop("vold.decrypt") = '1')
/system/media/bootanimation.zip
```

- **注释1**：搜索 /product 分区的特性在 Android 9 被添加，搜索/apex 分区的特性在 Android 10 才被添加，按照历史惯性而言，厂商默认的开机动画一般保存在`/system`分区。

- **注释2**：`vold.decrypt`属性表明此 Android 系统开启了全盘加密。[全盘加密](https://source.android.com/docs/security/encryption/full-disk)特性从 Android 10 开始已被废弃，只有在启用此特性的机器上才需要特别关注`bootanimation-encrypted.zip`文件。

## 文件结构

`bootanimation.zip`一般包含以下文件：

```txt
desc.txt - 描述如何执行动画的文本文档
part0  \
part1   \  文件夹，包含一段动画所有的帧，这些帧以PNG文件保存
...     /
partN  /
```

`bootanimation.zip`允许定义多个不同的动画片段，并把它们串联在一起组成完整的开机动画，这些动画片段存储在不同的`partN`文件夹里，其中 N 指的是序号的数字。

`partN`文件夹里除了包含一帧帧 PNG 图片，还可以包含一些其他资源或配置文件。

## desc.txt 

第一行定义动画的通用参数

```txt
WIDTH HEIGHT FPS [PROGRESS]
```

- **WIDTH**: 动画宽度（像素）
- **HEIGHT**: 动画高度（像素）
- **FPS**: 每秒帧数，例如60
- **PROGRESS**：（可选，since Android 12），是否显示最后一个动画片段的进度百分比
  - 百分比将水平居中，y 坐标将被设置为动画高度的1/3。

第二行以及以后的若干行定义一个动画片段：

```txt
TYPE COUNT PAUSE PATH [#RGBHEX [CLOCK1 [CLOCK2]]]
```

- **TYPE**: 单个字符，或`$SYSTEM`，指示动画片段的类型：
  - `p`: 播放这段动画，但会被开机完成事件打断
  - `c`: 完整播放这段动画，即使开机完成, 动画也不会被打断
  - `$SYSTEM`: 加载 `/system/media/bootanimation.zip` 并播放它。
- **COUNT**: 最大播放多少次动画，如果设置为 0，则动画无限循环直到启动完成
- **PAUSE**: 该部分结束后延迟多少帧再播放下一个动画片段
- **PATH**: 动画资源目录（例如`part0`）
- **RGBHEX**: （可选）背景颜色，格式为`#RRGGBB`
- **CLOCK1、CLOCK2**：（可选）绘制当前时间的坐标（对于手表）：
  - 如果仅提供`CLOCK1`，则它会被解析为时钟的 y 坐标，时钟的 x 坐标默认为`c`
  - （since Android 9）如果同时提供了`CLOCK1`和`CLOCK2`，则第一个作为 x 坐标，第二个作为 y 坐标
  - 值可以是正整数、负整数或`c`
    - `c`：将文本居中
    - 正整数`n`，：x 坐标，从屏幕左边缘开始算起的像素，y 坐标，从屏幕下边缘开始算起的像素
    - `-n`：x 坐标，从屏幕右边缘开始算起的像素，y 坐标，从屏幕上边缘开始算起的像素
    - 例子：
      - `-24`或者`c -24`，将文本定位在距屏幕顶部 24 像素处，水平居中
      - `16 c`，将文本定位在距屏幕左侧 16 像素处，垂直居中
      - `-32 32`，将文本定位在屏幕右侧 32 像素、底边缘上方 32 像素处

> 注意，同时指定时钟的 x、y 坐标是从 Android 9.0 开始支持的特性，Android 9.0 以前只支持指定 y 坐标

## clock_font.png(since Android 9)

可以使用该文件指定绘制时间使用的字体。字体文件格式要求如下：

- 该文件指定 ASCII 字符 32-127 (0x20-0x7F) 的字形，包括常规粗细和粗体粗细。
- 图像被划分为字符网格
- 有16列和6行
- 每行分为两半部分：上半部分为常规粗细字形，下半部分为粗体字形。
- 对于 NxM 大小的图像，每个字符字形的宽度为 N/16 像素，高度为 M/(12*2) 像素

## 加载和播放动画帧

每部分的动画都直接从 zip 文件中扫描并加载。在`partN`目录下，每个文件（除了`trim.txt`和`audio.wav`，请参阅下一节）都应该是一个 PNG 文件，表示该动画中的一帧（以指定的分辨率）。因此，必须按顺序命名动画帧（比如`part000.png`、`part001.png`）并按该顺序添加到 zip 文件中。

## trim.txt

可以对动画帧进行缩放，只需要在`partN`目录下提供`trim.txt` 文件即可。这个文件按顺序列出其目录中每个帧的缩放输出，因此可以将动画帧放在合适的位置上。输出应采用以下形式：`WxH+X+Y`，其中`W`和`H`表示重新放大或缩小后的动画帧大小。例如：

```txt
713x165+388+914
708x152+388+912
707x139+388+911
649x92+388+910
```

如果不提供该文件，则假定每个帧的大小与`desc.txt`中指定的宽高参数相同。

## audio.wav

每个动画片段可以选择在开始时播放`wav`。要启用此功能，请在`partN`目录下提供`audio.wav`文件。

## 退出开机动画

系统完成启动后将结束开机动画（仍会播放任何没播完甚至还没开始播放的类型为`c`的开机动画），这是通过将系统属性`service.bootanim.exit`设置为非零字符串来完成的。

## 提示

### PNG压缩

可以使用`zopflipng`或`pngcrush`来压缩 PNG 图像。例如：

```bash
for fn in *.png ; do
    zopflipng -m ${fn}s ${fn}s.new && mv -f ${fn}s.new ${fn}
    # or: pngcrush -q ....
done
```

如果允许将动画减小到256种颜色，压缩效果会更好，酌情使用：

```bash
pngquant --force --ext .png *.png
# alternatively: mogrify -colors 256 anim-tmp/*/*.png
```

### 如何创建 ZIP 

```bash
cd <path-to-pieces>
zip -0qry -i \*.txt \*.png \*.wav @ ../bootanimation.zip *.txt part*
```

**请注意**，ZIP 文件的压缩等级为0，实际上并未压缩！（使用其他压缩等级会导致读取文件失败） 因为 PNG 文件已经尽可能压缩，文件之间不太可能有任何冗余。

## 开机动画与动态颜色(since Android 12L)

从 Android 12L 开始，Google 团队将 Android 12 引入的[Dynamic color](https://source.android.com/docs/core/display/dynamic-color)特性也应用到了开机动画上。在此模式下，开机动画不再直接渲染 PNG 图像，而是将 PNG 图像的 R、G、B、A 通道视为动态颜色的 mask（掩码），根据动画的进度，在开始颜色和结束颜色之间进行插值。

要启用动态颜色特性，需要在 desc.txt 的第二行添加以下文本：

```txt
dynamic_colors PATH #RGBHEX1 #RGBHEX2 #RGBHEX3 #RGBHEX4
```

- **PATH：** 要应用动态颜色过渡的部分的文件路径。该片段之前的任何部分都将以起始颜色渲染。之后的任何部分都将以最终颜色渲染。
- **RGBHEX1：** 第一个起始颜色（masked by the R channel），指定为`#RRGGBB`。
- **RGBHEX2：** 第二个起始颜色（masked by the G channel），指定为`#RRGGBB`。
- **RGBHEX3：** 第三个起始颜色（masked by the B channel），指定为`#RRGGBB`。
- **RGBHEX4：** 第四个起始颜色（masked by the A channel），指定为`#RRGGBB`。

将从以下系统属性中读取结束颜色：

- `persist.bootanim.color1`
- `persist.bootanim.color2`
- `persist.bootanim.color3`
- `persist.bootanim.color4`

如果上面的某个系统属性为空，相应的结束颜色将默认为开始颜色，这样不会产生颜色转换。

准备您的PNG图像，使 R、G、B、A 通道分别指示要绘制`color1`、`color2`、`color3`和`color4`的区域。

### 动态颜色与开机动画的关系

简单来说，动态颜色是根据用户的相关设置（比如壁纸）等动态生成主题颜色调色板，并应用到系统上的过程。主题色变更后，系统将其更新在系统属性`persist.bootanim.color1`到`persist.bootanim.color4`上，也就是上文所说的结束颜色，开机动画通过读取系统属性并应用到动画上以支持动态颜色。

但经过上文的分析，我们也都清楚开机动画实际由一张张 PNG 图片组成的，图片上的颜色肯定是固定的，那么如何做到动态着色呢？

其实，图像本质上是保存“坐标”到“颜色”的映射的二维数组，想要支持动态颜色，我们需要给出“图像上的颜色”到“系统的动态颜色”的映射，那么解决方式就很简单了：

首先，我们从动画中提炼几种主要出现的固定颜色并把它们映射到动态颜色上，对比上面的描述，这一步相当于在`desc.txt`中填写`dynamic_colors PATH #RGBHEX1 #RGBHEX2 #RGBHEX3 #RGBHEX4`，每一个被填上去的固定颜色会被映射到相应的系统属性。

接下来，原来的动画帧中不应再保存固定颜色，而是保存这些颜色的“索引”，比如说我们简单用1、2、3、4来索引四种固定颜色，0表示没有任何颜色，那么，一个3x3的图像看起来可能像这样：

```txt
[0, 0, 0],
[0, 1, 2],
[0, 4, 3]
```

这种索引方式没有任何问题，只是稍显笨拙。除了简单的索引值，我们还可以考虑混合四种固定颜色的情况，即用四维向量来表示一个位置上的颜色，向量的分量表示混合了多少对应的固定颜色，这类似于我们用 RGB 来表示颜色，只不过现在三原色变成了“四原色”（姑且不考虑这四种颜色能不能组成正交基）。我们如何保存向量的分量值呢？原来的 PNG 图像在这时就派上了用场，PNG 图像有 ARGB 四个颜色通道，正好对应向量的四个分量！

> 如果有助于你理解的话，你还可以认为我们将图像的标准正交基从纯色变换为其他颜色

BootAnimation.cpp 中的 glsl 代码如下：

```glsl
precision mediump float;
const float cWhiteMaskThreshold = 0.05;
uniform sampler2D uTexture;
uniform float uFade;
uniform float uColorProgress;
uniform vec3 uStartColor0;
uniform vec3 uStartColor1;
uniform vec3 uStartColor2;
uniform vec3 uStartColor3;
uniform vec3 uEndColor0;
uniform vec3 uEndColor1;
uniform vec3 uEndColor2;
uniform vec3 uEndColor3;
varying highp vec2 vUv;
void main() {
    vec4 mask = texture2D(uTexture, vUv);
    float r = mask.r;
    float g = mask.g;
    float b = mask.b;
    float a = mask.a;
    // If all channels have values, render pixel as a shade of white.
    float useWhiteMask = step(cWhiteMaskThreshold, r)
        * step(cWhiteMaskThreshold, g)
        * step(cWhiteMaskThreshold, b)
        * step(cWhiteMaskThreshold, a);
    // 图像的rgba现在变为了动态颜色的系数, 这意味着它们共同指导了某个像素应当混合哪几类基色，并且每种基色混合到什么程度
    vec3 color = r * mix(uStartColor0, uEndColor0, uColorProgress)
            + g * mix(uStartColor1, uEndColor1, uColorProgress)
            + b * mix(uStartColor2, uEndColor2, uColorProgress)
            + a * mix(uStartColor3, uEndColor3, uColorProgress);
    // 如果图像的rgba都有值，那么将会求rgba的平均值, 并将这个平均值作为颜色的三个分量，也就是说颜色变成灰色或白色
    color = mix(color, vec3((r + g + b + a) * 0.25), useWhiteMask);
    gl_FragColor = vec4(color.x, color.y, color.z, (1.0 - uFade));
}
```

## 调试技巧

如果希望在运行中的系统调试开机动画，请顺序执行以下命令：

```
adb root
adb remount
// 覆盖系统中原本的开机动画，push命令的路径应根据实际情况修改
adb push .\bootanimation.zip /system/media/bootanimation.zip
// 这个属性标记了开机动画是否应当退出, 重置这个标志以保证开机动画会被执行(即使类型为p)
adb shell setprop service.bootanim.exit 0
adb shell setprop ctl.start bootanim
```

如果开机动画被配置为无限循环，再次执行以下命令（重新置位）才能终止开机动画：

```
adb shell setprop service.bootanim.exit 1
```

