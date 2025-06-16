# 格式转换markdown调研报告

> 时间: 2025年3月13日

下文主要介绍doc和pdf文件格式转换成md得技术实现,技术栈语言为golang。另外目前golang有一个docx[解析库可以使用](https://github.com/qifengzhang007/gooxml)
，虽然不完美，但是可以解决md处理问题。

技术路线主要有4种方式,优先级从高到底

- 寻找高star免费开源库
- 使用程序包装命令行，包装工具调用
- 采用大模型技术
- 服务供应商

另外还有一种方案,如果实在不能转换成md则,直接提取文本，不带格式文本得方式。

# 一、pdf转换成md技术线路

曲线路径方法

- pdf转成html html转成md
- pdf转成docx docx转成md

## pdf转md库

- https://github.com/koushamad/PDFtoMD go语言实现，库本身集成了MuPDF库, 需要cgo,MuPDF本身支持生成html，受到MuPDF库得影响PDFtoMD生成的效果一般。
- https://github.com/kshard/pdf2txt go项目包装`poppler-utils套件的pdftotext`[命令](https://poppler.freedesktop.org/)
- https://github.com/iamarunbrahma/pdf-to-markdown 一个python项目，支持ocr需要安装tesseract-ocr

## pdf转docx库

- https://github.com/ArtifexSoftware/pdf2docx python库基于PyMuPDF
- 使用LibreOffice进行转换

# 二、doc技术线路

- doc转成html html转成md
- doc转成docx docx转成md

## doc转html

- 使用LibreOffice进行转换

转换后可使用 `https://github.com/JohannesKaufmann/html-to-markdown` 进行成md

## doc转docx

- 使用LibreOffice进行转换

可配合 `https://github.com/qifengzhang007/gooxml` 进行转换

# 三、采用大模型

- https://github.com/zyocum/pdf2md py项目 需要安装poppler转换成图片
- https://github.com/felipefontoura/doc2md py项目也是支持多种类型类型，利用大模型，需要充值模型token
- 其他开源得golang模型包装sdk

# 四、收费服务

- https://pdfcrowd.com/pricing/ 每月11刀 支持pdf到html,有sdk
- https://github.com/unidoc/ 收费的纯go sdk 支持多种文件类型操作sdk
- 阿里云智能文档,直接提供文档转md
- 其他云服务厂商

# 其他

以下罗列出可能有用，或者比较有意思的库

- https://github.com/opendatalab/magic-doc py项目，利用libreoffice实现PPT/PPTX/DOC/DOCX/PDF）转换为 markdown
- https://github.com/microsoft/markitdown 微软开源的py库,支持多种文件转换成md
- https://github.com/deanmalmgren/textract 也是一个支持多种文件转换成文本的库也是基于py，支不支持md还需验证
- https://github.com/gotenberg/gotenberg 一个golang项目支持大多数文件类型转换成pdf,利用docker包装了libreoffice服务、chromium、pdftk等等工具

---

# LibreOffice使用

libreoffice是一个免费切开源的产品，支持多种文件格式相互之间转换。

https://zh-cn.libreoffice.org/

## soffice命令行使用

> 提示: soffice为命令行工具,同时也是个gui程序,根据启动参数`--headless`来区别，另外可能收到不同系统或版本的影响有的叫
`libreoffcie`

pdf -> docx

```shell
soffice --headless --infilter="writer_pdf_import" --convert-to docx  ../testdata/test.pdf --outdir ../testdata/
```

提示：如果出现`Error: source file could not be loaded` 则没有指定`--outdir`参数

pdf -> html

```shell
soffice --headless  --convert-to html ./testdata/test.pdf --outdir ../testdata/ --infilter=writer_pdf_import
```

doc -> txt

```shell
soffice --convert-to txt ./testdata/test.doc ./testdata/testdoc.txt
```

doc - >docx

```shell
soffice --headless --convert-to docx  ../../doc/testdata/test.doc --outdir ../testdata/
```

### 注意这个命令行有问题,调用直接阻塞住了服务不知道为何？

```shell
soffice --headless  ./testdata/test.pdf ./testdata/testpdf.txt --infilter=writer_pdf_import
```

## unoconvert命令使用

unoconvert是一个python项目，利用python包装了libreoffice服务，支持多种文件转换

https://github.com/unoconv/unoserver

pdf -> docx

```shell
/Users/edy/Library/Python/3.9/bin/unoconvert --verbose --host=127.0.0.1 --port=2003 --host-location=remote --convert-to=docx ./testdata/test.pdf ./testdata/testpdf.docx --input-filter=writer_pdf_import
```

pdf -> txt

```shell
 /Users/edy/Library/Python/3.9/bin/unoconvert --verbose --host=127.0.0.1 --port=2003 --host-location=remote --convert-to=txt ./testdata/test.pdf ./testdata/testtext.txt --input-filter=writer_pdf_import
```

**提示:不知为啥为空转换完**

pdf -> html

```shell
/Users/edy/Library/Python/3.9/bin/unoconvert --verbose --host=127.0.0.1 --port=2003 --host-location=remote --convert-to="html" ./testdata/test.pdf ./testdata/testpdf.html --input-filter=writer_pdf_import
```

**提示: 不知为何转换出来的内容为空，但是html中有样式每有内容**

/Users/edy/Library/Python/3.9/bin/unoconvert --verbose --host=127.0.0.1 --port=2003 --host-location=remote
--convert-to=docx ./testdata/test.doc ./testdata/testdoc.docx

# libreoffice 使用相关问题

1. 使用 `unoconvert` 或者其他包装库，或代码调用，服务调用阻塞hang住了,原因是没有启动`unoserver`服务(
   这个问题我卡住了1天特此记一下)
2. 使用`unoconvert` 报告文件not exist 错误，如果本地文件确实存在,那没有可能是`unoconvert`服务启动连接方式不对也就是
   `--host-location=local`如果`unoconvert`和`unoserver`不在同一个机器上，那么`--host-location=remote`不在一太机器内则，改为
   `--host-location=remote`

## 参考项目

- https://github.com/xbmlz/uniconv 一个使用golang包装py脚本的库,golang中非常值得参考的项目
- https://github.com/unoconv/unoserver 一个py写的用于和libreoffice进行交互的服务，其中包含了unoconvert

## 值得参考libreoffice镜像

如果想把自己的代码服务和libreoffice服务构建在一个镜像中，可以参考以下镜像

- https://github.com/libreofficedocker/libreoffice-unoserver
- https://github.com/unoconv/unoserver-docker

---

# 最终方案

## doc

doc -(libreoffice)-> docx -> 调用 `https://github.com/qifengzhang007/gooxml` 库自己构建md

## pdf

### 方案一

pdf -(libreoffice)-> html -> [htmltomd](https://github.com/felipefontoura/doc2md)(golang库)

> 注意：该方案目前存在一个问题,unoserver转换有问题,使用soffice命令转换是没有问题的,但是unoserver就有问题。
> 等待issues反馈 https://github.com/unoconv/unoserver/issues/164

### 方案二

兜底方案

使用`gen2brain` 提取text

或使用`ledongthuc` 提取文本
