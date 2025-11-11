# Image-to-HTML Conversion

```python
from PIL import Image
import torch
from transformers import AutoModel, AutoTokenizer

model_path = 'openbmb/MiniCPM-V-4_5'
model = AutoModel.from_pretrained(model_path, trust_remote_code=True,
                                  # sdpa or flash_attention_2, no eager
                                  attn_implementation='sdpa', torch_dtype=torch.bfloat16)
model = model.eval().cuda()
tokenizer = AutoTokenizer.from_pretrained(
    model_path, trust_remote_code=True)

image = Image.open('./assets/picture_code.png').convert('RGB')

question = "Analyze this screenshot and convert it to clean, functional and modern HTML code."
msgs = [{'role': 'user', 'content': [image, question]}]

answer = model.chat(
    msgs=msgs,
    image=image,
    tokenizer=tokenizer
)
print(answer)
```

![alt text](./assets/picture_code.png)

Text Result:

```html
<html>
  <head>
    <meta charset="utf-8">
    <title>HTML 实例</title>
    <style>
      body {
        font-family: Arial, sans-serif;
        line-height: 1.6;
        margin: 20px;
      }
      .note {
        background-color: #f9f2e7;
        border: 1px solid #d9d4c6;
        padding: 15px;
        margin-bottom: 20px;
        color: #4a4a4a;
        font-size: 14px;
      }
      .example {
        background-color: #fff9e6;
        border: 1px solid #e6e6e6;
        padding: 20px;
        margin-bottom: 20px;
        border-radius: 4px;
        font-family: 'Courier New', Courier, monospace;
        font-size: 14px;
      }
      .example h1 {
        color: #0066b2;
        margin-top: 0;
      }
      .example pre {
        background-color: transparent;
        border: none;
        padding: 0;
        margin: 0;
      }
      .button {
        display: inline-block;
        background-color: #00cc00;
        color: white;
        text-decoration: none;
        padding: 10px 20px;
        border-radius: 5px;
        margin: 20px 0;
        cursor: pointer;
      }
      .button:hover {
        background-color: #009900;
      }
    </style>
  </head>
  <body>
    <h1>HTML 实例</h1>
    <p>本教程包含数百个 HTML 实例。</p>
    <p>使用本站的编辑器，您可以轻松实现在线修改 HTML，并查看实例运行结果。</p>
    <div class="note">
      <strong>注意：</strong>对于中文网页需要使用 <code>&lt;meta charset="utf-8"&gt;</code> 声明编码，否则会出现乱码。有些浏览器(如 360 浏览器)会设置 GBK 为默认编码，则你需要设置为 <code>&lt;meta charset="gbk"&gt;</code>。
    </div>
    <h2>实例</h2>
    <div class="example">
      <pre>&lt;!DOCTYPE html&gt;
&lt;html&gt;
&lt;head&gt;
&lt;meta charset="utf-8"&gt;
&lt;title&gt;菜鸟教程 (runoob.com)&lt;/title&gt;
&lt;/head&gt;
&lt;body&gt;
&lt;h1&gt;我的第一个标题&lt;/h1&gt;
&lt;p&gt;我的第一个段落。&lt;/p&gt;
&lt;/body&gt;
&lt;/html&gt;</pre>
    </div>
    <a href="#" class="button">尝试一下 »</a>
    <p>点击“尝试一下”按钮查看在线实例</p>
  </body>
</html>
```

## Rendering Result

___

<html>
  <head>
    <meta charset="utf-8">
    <title>HTML 实例</title>
    <style>
      body {
        font-family: Arial, sans-serif;
        line-height: 1.6;
        margin: 20px;
      }
      .note {
        background-color: #f9f2e7;
        border: 1px solid #d9d4c6;
        padding: 15px;
        margin-bottom: 20px;
        color: #4a4a4a;
        font-size: 14px;
      }
      .example {
        background-color: #fff9e6;
        border: 1px solid #e6e6e6;
        padding: 20px;
        margin-bottom: 20px;
        border-radius: 4px;
        font-family: 'Courier New', Courier, monospace;
        font-size: 14px;
      }
      .example h1 {
        color: #0066b2;
        margin-top: 0;
      }
      .example pre {
        background-color: transparent;
        border: none;
        padding: 0;
        margin: 0;
      }
      .button {
        display: inline-block;
        background-color: #00cc00;
        color: white;
        text-decoration: none;
        padding: 10px 20px;
        border-radius: 5px;
        margin: 20px 0;
        cursor: pointer;
      }
      .button:hover {
        background-color: #009900;
      }
    </style>
  </head>
  <body>
    <h1>HTML 实例</h1>
    <p>本教程包含数百个 HTML 实例。</p>
    <p>使用本站的编辑器，您可以轻松实现在线修改 HTML，并查看实例运行结果。</p>
    <div class="note">
      <strong>注意：</strong>对于中文网页需要使用 <code>&lt;meta charset="utf-8"&gt;</code> 声明编码，否则会出现乱码。有些浏览器(如 360 浏览器)会设置 GBK 为默认编码，则你需要设置为 <code>&lt;meta charset="gbk"&gt;</code>。
    </div>
    <h2>实例</h2>
    <div class="example">
      <pre>&lt;!DOCTYPE html&gt;
&lt;html&gt;
&lt;head&gt;
&lt;meta charset="utf-8"&gt;
&lt;title&gt;菜鸟教程 (runoob.com)&lt;/title&gt;
&lt;/head&gt;
&lt;body&gt;
&lt;h1&gt;我的第一个标题&lt;/h1&gt;
&lt;p&gt;我的第一个段落。&lt;/p&gt;
&lt;/body&gt;
&lt;/html&gt;</pre>
    </div>
    <a href="#" class="button">尝试一下 »</a>
    <p>点击“尝试一下”按钮查看在线实例</p>
  </body>
</html>

___

# Convert Chart image to Python Matplotlib Code

```python
from PIL import Image
import torch
from transformers import AutoModel, AutoTokenizer

model_path = 'openbmb/MiniCPM-V-4_5'
model = AutoModel.from_pretrained(model_path, trust_remote_code=True,
                                  # sdpa or flash_attention_2, no eager
                                  attn_implementation='sdpa', torch_dtype=torch.bfloat16)
model = model.eval().cuda()
tokenizer = AutoTokenizer.from_pretrained(
    model_path, trust_remote_code=True)

image = Image.open('./assets/bar_graph.png').convert('RGB')

question = "Convert this chart image to Python matplotlib code which can reproduce the chart"
msgs = [{'role': 'user', 'content': [image, question]}]

answer = model.chat(
    msgs=msgs,
    image=image,
    tokenizer=tokenizer
)
print(answer)
```

![alt text](./assets/bar_graph.png)

## Resulting Code

```txt
```python
import matplotlib.pyplot as plt
import numpy as np

# Data
categories = ['A', 'B', 'C', 'D']
values = [650, 480, 920, 710]

# Create bar plot
x = np.arange(len(categories))  # the label locations
width = 0.35  # the width of the bars

fig, ax = plt.subplots(figsize=(6, 6))

bars = ax.bar(x, values, width, color=['#4ECDC4', '#FFC107', '#FF9800', '#03A9F4'], edgecolor='black')

# Add labels
ax.set_yticks(np.arange(0, 1100, 200))
ax.set_ylim(0, 1000)
ax.set_xticks(x)
ax.set_xticklabels(categories)

# Annotate data value on the chart
ax.bar_label(bars, padding=3)

# Customize appearance
ax.set_title('Bar Chart Example')
ax.set_xlabel('Categories')
ax.set_ylabel('Values')
ax.grid(True)
ax.set_facecolor('lightgray')

plt.tight_layout()
plt.savefig('chart.png')  # Save the figure
plt.show()
```

This code will generate a bar chart with:
- Categories A, B, C, D on the x-axis
- Values approximately matching the heights in your image (650, 480, 920, 710)
- Colored bars similar to your original chart colors
- Grid lines and proper axis labeling
- Data labels above each bar for clarity

You can adjust the colors by changing the `color` list or modify the values as needed to match your specific data more closely. The chart is saved as `chart.png` when run.
```

## Rendered Chart

![alt text](./assets/bar_graph_rendered.png)