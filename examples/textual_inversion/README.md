## Textual Inversion fine-tuning example

[Textual inversion](https://arxiv.org/abs/2208.01618) is a method to personalize text2image models like stable diffusion on your own images using just 3-5 examples.
The `textual_inversion.py` script shows how to implement the training procedure and adapt it for stable diffusion.

## Running on Colab 

Colab for training 
[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/huggingface/notebooks/blob/main/diffusers/sd_textual_inversion_training.ipynb)

Colab for inference
[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/huggingface/notebooks/blob/main/diffusers/stable_conceptualizer_inference.ipynb)

## Running locally with PyTorch
### Installing the dependencies

Before running the scripts, make sure to install the library's training dependencies:

**Important**

To make sure you can successfully run the latest versions of the example scripts, we highly recommend **installing from source** and keeping the install up to date as we update the example scripts frequently and install some example-specific requirements. To do this, execute the following steps in a new virtual environment:
```bash
git clone https://github.com/huggingface/diffusers
cd diffusers
pip install .
```

Then cd in the example folder  and run
```bash
pip install -r requirements.txt
```

And initialize an [🤗Accelerate](https://github.com/huggingface/accelerate/) environment with:

```bash
accelerate config
```

### Cat toy example

First, let's login so that we can upload the checkpoint to the Hub during training:

```bash
huggingface-cli login
```

Now let's get our dataset. For this example we will use some cat images: https://huggingface.co/datasets/diffusers/cat_toy_example .

Let's first download it locally:

```py
from huggingface_hub import snapshot_download

local_dir = "./cat"
snapshot_download("diffusers/cat_toy_example", local_dir=local_dir, repo_type="dataset", ignore_patterns=".gitattributes")
```

This will be our training data.
Now we can launch the training using

**___Note: Change the `resolution` to 768 if you are using the [stable-diffusion-2](https://huggingface.co/stabilityai/stable-diffusion-2) 768x768 model.___**

```bash
export MODEL_NAME="runwayml/stable-diffusion-v1-5"
export DATA_DIR="./cat"

accelerate launch textual_inversion.py \
  --pretrained_model_name_or_path=$MODEL_NAME \
  --train_data_dir=$DATA_DIR \
  --learnable_property="object" \
  --placeholder_token="<cat-toy>" --initializer_token="toy" \
  --resolution=512 \
  --train_batch_size=1 \
  --gradient_accumulation_steps=4 \
  --max_train_steps=3000 \
  --learning_rate=5.0e-04 --scale_lr \
  --lr_scheduler="constant" \
  --lr_warmup_steps=0 \
  --push_to_hub \
  --output_dir="textual_inversion_cat"
```

A full training run takes ~1 hour on one V100 GPU.

**Note**: As described in [the official paper](https://arxiv.org/abs/2208.01618) 
only one embedding vector is used for the placeholder token, *e.g.* `"<cat-toy>"`.
However, one can also add multiple embedding vectors for the placeholder token 
to inclease the number of fine-tuneable parameters. This can help the model to learn 
more complex details. To use multiple embedding vectors, you can should define `--num_vectors` 
to a number larger than one, *e.g.*:
```
--num_vectors 5
```

The saved textual inversion vectors will then be larger in size compared to the default case.

Also, To use [Prompt-plus extended textual inversion](https://prompt-plus.github.io/), 
additional argument
```
--prompt_plus
```
will do. This will generate 16 textual embeddings in total each of which is conditioned only in one cross attention layer. Generated embeddings are called, 
for example, `<cat-toy>_64d0`. `X_64d0` here implies it is conditioned in the first *(0)* cross attention layer in the down block *(d)* of resolution *64*. 
Note that down blocks have 2 cross attentions each, mid block has 1 and up blocks have 3 each. Similarly, 
`X_64d0, X_64d1, X_32d0, X_32d1, X_16d0, X_16d1, X_8m, X_16u0, X_16u1, X_16u2, X_32u0, X_32u1, X_32u2, X_64u0, X_64u1, X_64u2`.
These embeddings should be fed separately through a text prompt of length 16 when text conditioned. 
`--num_vectors` arg is still able to be specified. In this case, the saved textual inversion vectors will be of size 
`16 * num_vectors` times larger than vanilla textual inversion. 

### Inference

Once you have trained a model using above command, the inference can be done simply using the `StableDiffusionPipeline`. Make sure to include the `placeholder_token` in your prompt.

```python
from diffusers import StableDiffusionPipeline

model_id = "path-to-your-trained-model"
pipe = StableDiffusionPipeline.from_pretrained(model_id,torch_dtype=torch.float16).to("cuda")

prompt = "A <cat-toy> backpack"

image = pipe(prompt, num_inference_steps=50, guidance_scale=7.5).images[0]

image.save("cat-backpack.png")
```


## Training with Flax/JAX

For faster training on TPUs and GPUs you can leverage the flax training example. Follow the instructions above to get the model and dataset before running the script.

Before running the scripts, make sure to install the library's training dependencies:

```bash
pip install -U -r requirements_flax.txt
```

```bash
export MODEL_NAME="duongna/stable-diffusion-v1-4-flax"
export DATA_DIR="path-to-dir-containing-images"

python textual_inversion_flax.py \
  --pretrained_model_name_or_path=$MODEL_NAME \
  --train_data_dir=$DATA_DIR \
  --learnable_property="object" \
  --placeholder_token="<cat-toy>" --initializer_token="toy" \
  --resolution=512 \
  --train_batch_size=1 \
  --max_train_steps=3000 \
  --learning_rate=5.0e-04 --scale_lr \
  --output_dir="textual_inversion_cat"
```
It should be at least 70% faster than the PyTorch script with the same configuration.

### Training with xformers:
You can enable memory efficient attention by [installing xFormers](https://github.com/facebookresearch/xformers#installing-xformers) and padding the `--enable_xformers_memory_efficient_attention` argument to the script. This is not available with the Flax/JAX implementation.
