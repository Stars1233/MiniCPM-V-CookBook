# Multi Images

### Initialize model

```python
from PIL import Image
import torch
from transformers import AutoModel, AutoTokenizer

model_path = 'openbmb/MiniCPM-V-4_5'
model = AutoModel.from_pretrained(model_path, trust_remote_code=True,
                                  attn_implementation='sdpa', torch_dtype=torch.bfloat16)  # sdpa or flash_attention_2, no eager
model = model.eval().cuda()
tokenizer = AutoTokenizer.from_pretrained(
    model_path, trust_remote_code=True)
```

### Chat with multiple images

```python
image1 = Image.open('assets/multi1.png').convert('RGB')
image2 = Image.open('assets/multi2.png').convert('RGB')
question = 'Compare the two images, tell me about the differences between them.'

msgs = [{'role': 'user', 'content': [image1, image2, question]}]

answer = model.chat(
    image=None,
    msgs=msgs,
    tokenizer=tokenizer
)
print(answer)
```

### Second round chat

```python
# Second round chat, pass history context of multi-turn conversation
msgs.append({"role": "assistant", "content": [answer]})
msgs.append({"role": "user", "content": ["Tell me about the similarities between them."]})
answer = model.chat(
    image=None,
    msgs=msgs,
    tokenizer=tokenizer,
    enable_thinking=True
)
print(answer)
```

### Sample Images

![alt text](./assets/multi1.png)

![alt text](./assets/multi2.png)

### Example Output

First round

```
<think>
So, let's analyze the two images. First, observe the color schemes. The first vase  is red and white. The second vase  has green, red, and blue colors. Let's check the patterns too. The first one has red cloud-like patterns and maybe dragons or mythical creatures. The second one has more geometric and floral patterns with different colors. Also, the shapes might be similar as both are gourd-shaped vases, but the decoration styles are different. The first is monochromatic red on white, the second uses multiple colors like green, blue, red. Let's list the differences:

- Color scheme: the first is red and white; the second uses green, red, blue, and other colors.
- Pattern details: the first has red cloud motifs and possibly figures; the second has floral and geometric designs with multiple colors.
- Overall design elements: the first is simpler with one color (red) on white, while the second is more colorful and complex in decoration.
</think>

To compare the two images, we analyze the following aspects:  

1. **Color Scheme**:  
   - The first vase  is decorated with red patterns on a white background.  
   - The second vase  features a combination of green, red, blue, and other colors on a white background.  

2. **Decorative Patterns**:  
   - The first vase displays red cloud - like motifs and mythical figure designs (such as a central figure surrounded by clouds).  
   - The second vase has green geometric and floral patterns, with colorful flower - shaped elements (red and orange centers with green leaves) and other decorative symbols.  

3. **Overall Aesthetic and Complexity**:  
   - The first vase has a monochromatic (red - white) and relatively simpler design.  
   - The second vase has a multi - colored and more intricate decorative scheme.  


In summary, the primary differences lie in their color schemes (red - white vs. multi - colored) and the complexity of their decorative patterns.
```

Second round

```
<think>
To find the similarities between the two vases, let's examine their shapes, materials, and some design elements:

1. **Shape and Structure**:  
   Both vases have a gourd - shaped form, consisting of a bulbous base, a narrow neck, and a small opening at the top. This unique double - gourd silhouette is a common design feature for both.

2. **Material and Background**:  
   Each vase is made of porcelain, which is evident from their smooth white background. This white ceramic base serves as a canvas for the colored designs.

3. **Decorative Style**:  
   Both pieces feature intricate, hand - painted motifs. The designs are detailed and cover significant portions of the surface, showcasing the craftsmanship typical of traditional porcelain art.

4. **Cultural Aesthetic**:  
   The overall style of both vases reflects traditional East Asian ceramic art, likely Chinese. This is suggested by the use of symbolic patterns and the gourd - shaped form, which is often seen in Chinese decorative art.

In conclusion, the main similarities are their gourd - shaped structure, porcelain material, white background, intricate decorative designs, and the cultural East Asian artistic style they represent.
```
