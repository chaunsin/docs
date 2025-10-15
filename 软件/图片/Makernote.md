# Makernotes（制造商注释区）

> - 来源: [Exiv2 官方文档 - Makernotes](https://exiv2.org/makernote.html)
> - 说明: 使用ChatGPT进行了翻译整理
> - 时间: 2025-08-31 10:14 UTC
---

## 📑 目录

- [简介](#简介)
- [Makernote 的作用](#makernote-的作用)
- [厂商实现与兼容性问题](#厂商实现与兼容性问题)
- [Makernote 数据结构汇总表](#makernote-数据结构汇总表)
- [参考文献](#参考文献)

---

## 简介

Makernote 是 Exif IFD（图像文件目录）中的标签 `0x927C`，通常Makernote也叫[IFD](https://exiv2.org/book/index.html#MakerNotes), 根据 [Exif 2.2 标准](https://www.exif.org/Exif2-2.PDF)：
> “Makernote 标签供 Exif 写入器制造商使用，用于记录任何所需信息，其内容由制造商自行定义，但不应滥用于非预期目的。”

---

## Makernote 的作用

多数厂商并未公开其 Makernote 格式规范。现有的资料多为社区开发者通过逆向工程得出。

Makernote 中通常包含非常多的相机信息，例如：

- 使用的镜头信息
- 对比度、饱和度、锐度设置
- 图像质量模式等

某些高级相机功能在 Exif 标准中没有定义对应标签，因此厂商只能将其写入 Makernote。例如，**Nikon 的 ISO 设置**就只存在于其专有
Makernote 字段中。

---

## 厂商实现与兼容性问题

多数厂商将 Makernote 编码为 **TIFF 格式**，即与 Exif 其他部分使用相同的结构。然而这样会导致一个潜在问题：
> 若修改 Exif 数据导致 Makernote 字段偏移位置，Makernote 区块可能被破坏。

原因在于：  
TIFF 写入器在保存数据时必须理解所有已知和未知扩展标签，否则一旦字段偏移改变，未知字段可能被错误重写。

部分厂商（如 Nikon、Fuji）为了解决此问题，会在 Makernote 内使用**相对偏移**地址，以避免因整体文件结构调整导致数据损坏。

---

## Makernote 数据结构汇总表

| 厂商 (Make)                | 格式 (Format)                     | 头部标识 (Header)                         | 字节序 (Endian)#1  | 偏移规则 (Offsets)#2 | 参考文献 (Ref) | 备注 (Remarks)                                                                                     |
|--------------------------|---------------------------------|---------------------------------------|-----------------|------------------|------------|--------------------------------------------------------------------------------------------------|
| Canon                    | IFD                             | None                                  | —               | —                | [2]        | 部分 CR2 文件包含非零 next-IFD 指针                                                                        |
| CASIO                    | IFD                             | None                                  | —               | —                | [4]        |                                                                                                  |
| CASIO                    | IFD，offset 6                    | `"QVC\0\0\0"`                         | Big (MM)        | 相对 Makernote 起始  | [4]        |                                                                                                  |
| FUJI                     | IFD, 通常偏移量 12                   | 以字符串“FUJI”和指向 IFD 的 4 字节指针开头          | Little (II)     | 相对 Makernote 起始  | [1]        | Exif 主体为大端 (MM)                                                                                  |
| Minolta / KONICA MINOLTA | IFD                             | None                                  | —               | —                | [5]        | 相机设置标签采用大端 (MM) 格式编码，与 Exif 数据的编码格式无关                                                            |
| NIKON                    | IFD                             | None                                  | —               | —                | [3]        | 使用此 makernote 的型号包括 E990、D1                                                                      |
| NIKON                    | IFD @ offset 8                  | `"Nikon\0"` + 2 未知字节                  | —               | —                | [1]        | 使用此 makernote 的型号包括 E700、E800、E900、E900S、E910、E950                                               |
| NIKON                    | IFD, 通常偏移量 18                   | “Nikon\0” 后跟 4 个字节，看起来像版本代码和 TIFF 标头  | 来自 TIFF 头       | 相对 TIFF 起始       | —          | 使用此 makernote 的型号包括 E5400、SQ、D2H、D70、D100 和 D200。D200 的 makernote IFD 没有下一个 IFD 指针。（这是一个 bug 吗？） |
| OLYMPUS                  | IFD, 偏移量 8                      | `"OLYMP\0"` + 2 未知字节                  | —               | —                | [1]        |                                                                                                  |
| OLYMPUS                  | IFD, 偏移量 12                     | `"OLYMPUS\0II"` + 2 未知字节              | 相对 Makernote 起始 | —                | —          |                                                                                                  |
| Panasonic                | IFD,（偏移量为 12 处没有下一个 IFD 指针的 IFD | `"Panasonic\0\0\0"`                   | —               | —                | [8]        |                                                                                                  |
| PENTAX                   | IFD, 偏移量 6                      | `"AOC\0"` + 2 未知字节                    | —               | —                | [11]       |                                                                                                  |
| SAMSUNG                  | IFD                             | None                                  | —               | 相对 Makernote 起始  | —          |                                                                                                  |
| Sanyo                    | IFD                             | —                                     | —               | —                | [6]        | Exiv2 工具尚不支持                                                                                     |
| SIGMA / FOVEON           | IFD, 偏移量 10                     | “SIGMA\0\0\0”或“FOVEON\0\0”后跟两个含义不明的字节 | —               | —                | [7]        |                                                                                                  |
| SONY                     | IFD,（偏移量为 12 处没有下一个 IFD 指针的 IFD | `"SONY DSC \0\0\0"`                   | —               | —                | —          | 可以在 Jpeg 图像中看到，例如 DSC-W7、DSC-R1                                                                  |
| SONY                     | IFD                             | None                                  | —               | —                | —          | 在 SR2 图像中看到，例如来自 DSC-R1                                                                          |

> **注释：**
> - #1 若未指定，字节序与 Exif 主体一致。
> - #2 若未指定，偏移相对于 TIFF 头起始位置。

[Exif.org](http://www.exif.org/)
上还有另一张包含类似信息和样本图片的表格：[数码相机样本图片](http://www.exif.org/samples.html)。据此来源称，（至少部分）理光和柯达相机不会以
IFD 格式写入 makernote。

---

## 参考文献

以下链接有些失效，请参考 https://exiv2.org/makernote.html 原文

1. [Exif File Format by TsuruZoh Tachibanaya](http://park2.wakwak.com/~tsuruzoh/Computer/Digicams/exif-e.html)
2. [EXIF Makernote of Canon – David Burren](http://www.burren.cx/photo/exif.html)
3. [Makernote Tag of Nikon 990 – Max Lyons](http://www.tawbaware.com/990exif.htm)
4. [Makernote Exif Tag of Casio – Eckhard Henkel](http://www.dicasoft.de/casio.html)
5. [Minolta MakerNote – Dalibor Jelinek](http://www.dalibor.cz/minolta/makernote.html)
6. [Sanyo MakerNote – John Hawkins](https://www.exif.org)
7. [SIGMA / FOVEON EXIF MakerNote Docs – Foveon](http://www.x3f.info)
8. [Panasonic MakerNote Info – Tom Hughes](http://www.compton.nu/panasonic.html)
9. [Various Makernote Specs – Evan Hunter (PHP JPEG Metadata Toolkit)](http://www.ozhiker.com/electronics/pjmt/)
10. [ExifTool – Phil Harvey](https://exiftool.org)
11. [Pentax Type3 Makernotes – Ger Vermeulen](http://www.gvsoft.homedns.org/exif/makernote-pentax.html)
12. [Casio MakerNote Info – GVSoft](http://www.gvsoft.homedns.org/exif/makernote-casio.html)
13. https://web.archive.org/web/20111025004429/http://park2.wakwak.com/~tsuruzoh/Computer/Digicams/exif-e.html
14. https://exiv2.org/book/index.html#MakerNotes