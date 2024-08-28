---
title: PDF 格式化提取调研 
date: 2024-08-15T15:54:10+08:00
tags: ['PDF', "AI"]
series: []
featured: true
---

在整理PDF 格式化内容提取方案时，发现当前网上所提及的库要么落后于时代版本要么就是各种文字描述语焉不详。因此根据调研结果重新整理

<!--more-->


## 需求：

提取PDF文件内容，如果PDF文件中数据之间包含段落关系，需保留其段落关系。

## 困难点：

由于PDF文件可以有多种来源生成，如html文件转换、markdown文件转换、文本影印等。部分来源文件天然不存在段落关系或者文字信息不清晰等，因此本文调研优先考虑由html和markdown等层级化文件转换成的PDF文件内容提取。

## 实验方案：

使用由markdown文件生成的PDF文件进行内容提取，对比各自提取结构和结果

测试文件：

[test.pdf](/images/pdf_extractor_report/test.pdf)

## 调研对象

- pdfplumber
- pdfminer.six
- PyMuPDF
- pypdfium2
- marker
- MinerU

### pdfplumber

**文档地址：** https://github.com/jsvine/pdfplumber

**简介：** 查看PDF文件以获取有关每个文本字符、矩形和线条的详细信息。并具有表格提取和可视化调试能力，基于pdfminer.six

**测试结果：** 根据PDF的单页作为内容操作的基本单位，只支持获取纯文本信息，根据段落结构获取方式不成熟，以下为测试结果

- 直接文字提取

```python
import pdfplumber

def pdf_to_markdown(pdf_path: str):

    # 使用 pdfplumber 打开 PDF 文件
    with pdfplumber.open(pdf_path) as pdf:
        for page in pdf.pages:
            # 提取每页的 HTML 内容, 
            context = page.extract_text()
            print(context)

pdf_to_markdown("pdf_test/test.pdf")
```

![Untitled](/images/pdf_extractor_report/Untitled.png)

丢失文件段落结构，标题正文无法区分。

- 段落结构获取(实验阶段)

```python
import pdfplumber

def pdf_to_markdown(pdf_path: str):

    # 使用 pdfplumber 打开 PDF 文件
    with pdfplumber.open(pdf_path) as pdf:
        for page in pdf.pages:
            # 提取每页的 HTML 内容
            context = page.extract_text(layout=True)
            print(context)

pdf_to_markdown("pdf_test/test.pdf")
```

![Untitled](/images/pdf_extractor_report/Untitled%201.png)

丢失标题结构，且增加对应换行符，增加后续解析难度

### **pdfminer.six**

**文档地址：** https://pdfminersix.readthedocs.io/en/latest/

**简介：** Pdfminer.6 是社区维护的原始 PDFMiner 的分支。它是一个从PDF文档中提取信息的工具。它专注于获取和分析文本数据。 Pdfminer.6 直接从 PDF 源代码中提取页面文本。它还可用于获取文本的确切位置、字体或颜色。

**测试结果：** 通过`extract_text` 方法直接获取整页PDF的文本信息，丢失文本层级结构

```python
from pdfminer.high_level import extract_text

def pdf_to_markdown(pdf_path: str):

    text = extract_text(pdf_path)
    with open("./page.txt", "w") as f:
        f.write(text)
    print(text)

pdf_to_markdown("pdf_test/test.pdf")
```

![Untitled](/images/pdf_extractor_report/Untitled%202.png)

丢失部分PDF文档结构，丢失标题层级

### PyMuPDF

**文档地址：** https://pymupdf.readthedocs.io/en/latest/tutorial.html

**简介：** 高性能python库，用于PDF的数据提取、分析、转换和操作。

PyMuPDF Pro还支持 office文件的支持，包括：

- DOC/DOCX 文档/文档CX
- PPT/PPTX
- XLS/XLSX
- HWP/HWPX

模块基础单元为Document实例，用户可以通过PyMuPDF打开文件，获取Document实例。Document包含许多属性和功能。其中包括元信息（如“作者”或“主题”）、总页数、大纲和加密信息。

页面处理是PyMuPDF功能的核心，用户可以通过Document作为Page的迭代器，通过循环获取不同的Page。

用户可以通过不同形式和详细程度提取页面的所有文本、图像和其他信息。

```python
text = page.get_text(opt)
```

使用一下option选项获取不同的格式：

- **“text”** ：（默认）带有换行符的纯文本。没有格式、没有文本位置详细信息、没有图像。
- **“blocks”** ：生成文本块（=段落）列表。
- **“words”** ：生成单词列表（不包含空格的字符串）。
- **“html”** ：创建页面的完整视觉版本，包括任何图像。这可以通过您的互联网浏览器显示。
- **“dict”** / **“json”** ：与 HTML 相同的信息级别，但以 Python 字典或 resp 形式提供。 JSON 字符串。有关其结构的详细信息，请参阅[`TextPage.extractDICT()`](https://pymupdf.readthedocs.io/en/latest/textpage.html#TextPage.extractDICT) 。
- **“rawdict”** / **“rawjson”** ： **“dict”** / **“json”**的超集。它还提供诸如 XML 之类的字符详细信息。有关其结构的详细信息，请参阅[`TextPage.extractRAWDICT()`](https://pymupdf.readthedocs.io/en/latest/textpage.html#TextPage.extractRAWDICT) 。
- **“xhtml”** ：文本信息级别为TEXT版本，但包含图像。也可以通过互联网浏览器显示。
- **“xml”** ：不包含图像，但包含完整的位置和字体信息，具体到每个文本字符。使用 XML 模块进行解释。

PyMuPDF支持使用story类通过html源生成PDF，并用于解析。然而并不提供直接通过Stroy实例转换成Document实例的方法。story类相关文档：https://pymupdf.readthedocs.io/en/latest/recipes-stories.html#stories

**测试结果：** 根据PDF的单页作为内容操作的基本单位，保留文件结构，并提供文件大纲进行标题区分，可以通过大纲和文本内容区分相关结构，以下为测试结果

```python
doc.get_toc() # 获取Document的大纲信息
```

![Untitled](/images/pdf_extractor_report/Untitled%203.png)

```python
import pymupdf

content = ""
doc = pymupdf.open("./pdf_test/test.pdf")
for page in doc:
    text = page.get_text()
    print(text)
    content += text

with open("./pdf_test/pymupdf.txt", "w") as f:
    f.write(content)
```

![Untitled](/images/pdf_extractor_report/Untitled%204.png)

![Untitled](/images/pdf_extractor_report/Untitled%205.png)

### pypdfium2

**文档地址：** https://github.com/pypdfium2-team/pypdfium2

**简介：** pypdfium2是与PDFium绑定的ABI 级Python 3，PDFium 是一个功能强大且经过自由许可的库，用于 PDF 渲染、检查、操作和创建。

**测试结果：** 与PyMuPDF类似，通过Document以及Page来实现页面文本信息的获取，也支持获取文件大纲用于判断文档标题，文本内容在获取时会丢失相关层级结构

```python
import pypdfium2 as pdfium

pdf = pdfium.PdfDocument("./pdf_test/test.pdf")

for item in pdf.get_toc():
    state = "*" if item.n_kids == 0 else "-" if item.is_closed else "+"
    target = "?" if item.page_index is None else item.page_index + 1
    print(
        "    " * item.level
        + "[%s] %s -> %s  # %s %s"
        % (
            state,
            item.title,
            target,
            item.view_mode,
            item.view_pos,
        )
    )
print(list(pdf.get_toc()))
```

![Untitled](/images/pdf_extractor_report/Untitled%206.png)

可以展示PDF文件的层级结构

```python
import pypdfium2 as pdfium

pdf = pdfium.PdfDocument("./pdf_test/test.pdf")

content = ""
for page in pdf:
    textpage = page.get_textpage()
    text_all = textpage.get_text_range()
    print(text_all)
    content += text_all

with open("pdf_test/pypdfium2.txt", "w") as f:
    f.write(content)
```

![Untitled](/images/pdf_extractor_report/Untitled%207.png)

![Untitled](/images/pdf_extractor_report/Untitled%208.png)

文本提取不如PyMuPDF结构，不过依旧可以通过大纲和标题获取文件内容的层级结构。

### MinerU

文档地址：https://github.com/VikParuchuri/marker

**简介：** 是一款一站式、开源、高质量的数据提取工具，主要提供以下主要功能：

- Magic-PDF：是一款旨在将PDF文档转换为Markdown格式的工具，能够处理本地存储或支持S3协议的对象存储上的文件。
  - 支持多种前端模型输入
  - 删除页眉、页脚、脚注和页码
  - 人类可读的布局格式
  - 保留原始文档的结构和格式，包括标题、段落、列表等
  - Markdown 图像和表格的提取和显示
  - 自动检测并转换PDF乱码
  - 将方程转换为 LaTeX 格式
  - 与CPU和GPU环境的兼容性
  - 适用于 Windows、Linux 和 macOS 平台
  - 基于PDF-Extract-Kit作为PDF内容提取的工具
- Magic-Doc：是一款旨在将网页或多格式电子书转换为Markdown格式的工具。
  - 网页提取：文本、图像、表格、公式信息的跨模态精准解析。
  - 电子书文档提取：支持epub、mobi等多种文档格式，文本、图片全面适配。
  - 语言类型识别：准确识别176种语言。

本文主要记录Magic-PDF对于PDF解析效果，Magic-PDF基于PDF-Extract-Kit进行PDF内容提取，因此需要下载PaddleOcr和其自定义训练的模型。模型大小约5G左右。

**测试结果：** 由于Magic-PDF模块安装复杂，因此使用命令行形式进行测试

```python
magic-pdf pdf-command --pdf "pdf_path/test.pdf" --inside_model true
```

成功执行后，会在设定好的temp-output-dir目录下生成magic-pdf文件夹。每单次执行会在此目录下生成一个文件夹内部包含解析文件

![Untitled](/images/pdf_extractor_report/Untitled%209.png)

layout.pdf和spans.pdf代表源PDF文件的内部元素的布局和宽度

![Untitled](/images/pdf_extractor_report/Untitled%2010.png)

![Untitled](/images/pdf_extractor_report/Untitled%2011.png)

后缀为md的文件代表其生成的Markdown文件：

![Untitled](/images/pdf_extractor_report/Untitled%2012.png)

后缀为json的文件代表各个元素的各类信息。

通过与源文件对比，可发现存在部分元素提取异常，元素结构提取错误。标题格式误识别率较高。如下图”DNS解析流程“应该为正文内容，这里却将它识别成了标题。

![Untitled](/images/pdf_extractor_report/Untitled%2013.png)

### marker

**文档地址：** https://github.com/VikParuchuri/marker

**简介：** 快速准确地将 PDF 转换为 Markdown。

- 支持多种文档（针对书籍和科学论文进行了优化）
- 支持所有语言
- 清除页眉/页脚/其他杂物
- 格式化表格和代码块
- 提取并保存图像以及相应的 markdown
- 将大多数方程转换为 latex
- 可在 GPU、CPU 或 MPS 上运行

**测试结果：通过 `convert_single_pdf`方法直接获取整页PDF的文本信息，可以提取部分文档层级结构，可以提取代码、表格、公式，页眉页脚未能正确去除。**

```python
from marker.convert import convert_single_pdf
from marker.models import load_all_models

full_text, images, out_meta = convert_single_pdf(
    "./test2.pdf",
    load_all_models(),
    langs=["Chinese", "English"],
    batch_multiplier=2,
)

print(full_text)
print(images)
print(out_meta)
```

![image.png](/images/pdf_extractor_report/image.png)

![image.png](/images/pdf_extractor_report/image%201.png)

![image.png](/images/pdf_extractor_report/image%202.png)

                                                                          部分层级结构保留

如下，部分文本应为正文，但却被识别成标题结构。

![image.png](/images/pdf_extractor_report/image%203.png)

![image.png](/images/pdf_extractor_report/image%204.png)

实际测试中，页眉页脚未能正确去除

![image.png](/images/pdf_extractor_report/image%205.png)

                                                                            页眉页脚未能去除

需要从 huggingface 导入 6 个模型，显存要求较高，测试时出现爆显存现象

`model_lst = [texify, layout, order, edit, detection, ocr]`

![image.png](/images/pdf_extractor_report/image%206.png)

## 小结：

在以上模块中，如果只需要获取PDF文本信息的前提下，PyMuPDF模块的效果最好，文本提取准确，提供大纲方法获取文本标题，从而分析得出文本结构。

若需要PDF转成Markdown格式，可以使用MinerU模块，不过当前存在模块内部依赖混乱，外部调用API不完善等问题