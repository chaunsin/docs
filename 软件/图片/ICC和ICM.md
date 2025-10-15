# ICC和ICM

> - 说明：以下内容来自ai生成，部分内容拼凑而来，有错误请指正
> - 时间: 2025-10-14
> - 作者: chaunsin


ICC和ICM这一块正是图像底层处理和色彩管理领域的核心知识。
我们现在来深入讲讲 **ICC** 和 **ICM** —— 这两个经常一起出现、但很多人容易混淆的概念。

---

## 🧠 一、ICC 与 ICM 的本质区别（先说结论）

| 项目   | ICC                                  | ICM                                   |
|------|--------------------------------------|---------------------------------------|
| 全称   | **International Color Consortium**   | **Image Color Management**            |
| 性质   | **组织 / 标准制定者**                       | **实现 / 系统功能**                         |
| 内容   | ICC Profile 规范（文件格式 `.icc` / `.icm`） | 操作系统或软件实现的色彩管理系统                      |
| 主要作用 | 定义如何描述设备的色彩特性                        | 负责应用这些 ICC Profile 进行颜色转换             |
| 文件类型 | `.icc` 或 `.icm`（完全相同格式）              | 无文件，是功能模块（例如 Windows ICM）             |
| 举例   | sRGB.icc, AdobeRGB.icc               | Windows ICM、Apple ColorSync、Adobe CMM |

所以一句话总结：

> **ICC 是制定标准的机构，ICM 是操作系统实现 ICC 标准的功能。**

---

## 🧩 二、ICC（International Color Consortium）

### 1️⃣ 背景

ICC（国际色彩联盟）成立于 **1993 年**，由 Adobe、Apple、Kodak、Microsoft、Silicon Graphics 等公司共同创建，目的就是：

> 让图像在不同设备（相机、显示器、打印机）之间的颜色保持一致。

### 2️⃣ ICC Profile 是什么？

ICC 定义了一种称为 **ICC Profile（色彩描述文件）** 的标准格式，用来描述**设备的色彩特性**。

一个 ICC Profile 文件通常后缀为：

```
*.icc
*.icm
```

这两个扩展名其实是**同一种文件格式**，只是历史上：

* Windows 系统喜欢用 `.icm`；
* macOS / Adobe 等系统喜欢用 `.icc`。

ICC 配置文件是也 TIFF 图像标准系列的成员之一。它包含一个文件头、一个“标签”目录以及标签值。

---

## 📦 三、ICC Profile 文件内部结构

一个 ICC Profile 是一个二进制文件，它描述了设备如何把自己的颜色空间映射到一个“标准色彩空间”（Profile Connection Space，简称
**PCS**）。

典型结构如下：

```
+--------------------------+
| Header (128 bytes)       | ← 文件头
+--------------------------+
| Tag Table                | ← 列出有哪些数据块（如色度、Gamma等）
+--------------------------+
| Tag Data                 | ← 各种数据块内容
|   - 'desc' Profile描述
|   - 'rXYZ', 'gXYZ', 'bXYZ' RGB原色坐标
|   - 'rTRC', 'gTRC', 'bTRC' 色调响应曲线
|   - 'wtpt' 白点信息
|   - 'bkpt' 黑点信息
|   - 'cprt' 版权信息
+--------------------------+
```

---

## 🎨 四、ICC Profile 的类型

| 类型                        | 描述          | 示例                            |
|---------------------------|-------------|-------------------------------|
| **Device Profile**        | 设备特有的色彩描述文件 | 显示器、打印机、相机                    |
| **Working Space Profile** | 通用工作色彩空间    | sRGB, Adobe RGB, ProPhoto RGB |
| **Output Profile**        | 输出设备特性      | EpsonPrinter.icm              |
| **Input Profile**         | 输入设备特性      | CanonCamera.icc               |
| **Monitor Profile**       | 显示器         | Display.icc                   |

---

## ⚙️ 五、ICM（Image Color Management）

ICM 是一个通用术语，用来表示 **“色彩管理系统”**。
它是操作系统或图像应用程序中的一个模块，用来**解析和应用 ICC Profile**，从而实现颜色匹配。

### 典型实现：

| 平台       | 名称                                 | 说明                           |
|----------|------------------------------------|------------------------------|
| Windows  | ICM（Image Color Management）        | 微软的 ICC 实现，支持 `.icm` 文件      |
| macOS    | ColorSync                          | 苹果的色彩管理系统                    |
| Adobe 系列 | Adobe CMM（Color Management Module） | 独立实现，可跨平台                    |
| Linux    | LittleCMS（开源实现）                    | 很多图像库（如 ImageMagick、GIMP）使用它 |

---

## 🧠 六、它们如何协作工作（数据流图）

假设你打开一张 JPEG 图片：

```
JPEG文件
  ├─ 嵌入 ICC Profile (sRGB.icc)
  ↓
系统 ICM (Color Management)
  ↓
显示器 ICC Profile (Display.icc)
  ↓
屏幕显示正确颜色
```

解释：

1. JPEG 文件内部包含一个 **ICC Profile（比如 sRGB.icc）**；
2. ICM 系统（如 Windows ICM 或 macOS ColorSync）读取这个 Profile；
3. 同时获取显示器自身的 Profile；
4. 通过数学变换（颜色空间映射），把图像颜色转换成显示器的正确色域。

---

## 🧾 七、ICC 文件中可以嵌入的数据类型

ICC Profile 是一种非常规范的二进制文件格式，内部 Tag 数据类型包括：

| Tag 名称                 | 含义                          | 类型                  |
|------------------------|-----------------------------|---------------------|
| `desc`                 | Profile 描述                  | TextType            |
| `cprt`                 | 版权声明                        | TextType            |
| `rXYZ`, `gXYZ`, `bXYZ` | RGB 三基色的色度坐标                | XYZType             |
| `rTRC`, `gTRC`, `bTRC` | Tone Response Curve (Gamma) | CurveType           |
| `wtpt`                 | 白点坐标                        | XYZType             |
| `bkpt`                 | 黑点坐标                        | XYZType             |
| `chad`                 | 色适应矩阵（Chromatic Adaptation） | s15Fixed16ArrayType |

---

## 📚 八、文件嵌入与识别位置

常见图片格式中，ICC Profile 嵌入位置如下：

| 图片格式            | 是否支持 ICC | 嵌入方式                         |
|-----------------|----------|------------------------------|
| **JPEG**        | ✅ 支持     | APP2 段（Marker：`ICC_PROFILE`） |
| **TIFF**        | ✅ 支持     | 专用 Tag（Tag ID = 34675）       |
| **PNG**         | ✅ 支持     | 独立块：`iCCP`                   |
| **WebP**        | ✅ 支持     | Chunk：`ICCP`                 |
| **HEIF / HEIC** | ✅ 支持     | `colr` box                   |
| **BMP / GIF**   | ❌ 一般不支持  | —                            |

---

## 🧩 九、总结一览表

| 项     | ICC                            | ICM                          |
|-------|--------------------------------|------------------------------|
| 全称    | International Color Consortium | Image Color Management       |
| 本质    | 标准                             | 实现（系统功能）                     |
| 文件扩展名 | .icc / .icm                    | 无文件                          |
| 定义内容  | 色彩描述文件格式                       | 颜色管理流程                       |
| 示例    | sRGB.icc, AdobeRGB.icc         | Windows ICM, Apple ColorSync |
| 作用    | 描述设备的颜色特性                      | 将图像颜色转换成目标设备匹配的颜色            |

---

## ✅ 十、一句话总结

> **ICC 是“标准”，ICM 是“实现”。**
> ICC 定义了如何描述颜色，ICM 负责让颜色在不同设备间保持一致。

## 参考

- https://www.color.org/icc_specs2.xalter 规范
