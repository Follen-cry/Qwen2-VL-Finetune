# Fine-tuning Qwen2-VL Series

This repository contains a script for training [Qwen2-VL](https://huggingface.co/Qwen/Qwen2-VL-7B-Instruct) and [Qwen2.5-VL](https://huggingface.co/Qwen/Qwen2.5-VL-7B-Instruct) with only using HuggingFace and [Liger-Kernel](https://github.com/linkedin/Liger-Kernel).

## Notification

**Currently Qwen2.5-VL has a bug when using Flash-attention2. ~~You need to disable to train the model.~~**<br>
**I made a quick fix monkey-patching code for it.**

## Other projects

**[[Phi3-Vision Finetuning]](https://github.com/2U1/Phi3-Vision-Finetune)**<br>
**[[Llama3.2-Vision Finetuning]](https://github.com/2U1/Llama3.2-Vision-Ft)**<br>
**[[Molmo Finetune]](https://github.com/2U1/Molmo-Finetune)**<br>
**[[Pixtral Finetune]](https://github.com/2U1/Pixtral-Finetune)**<br>
**[[SmolVLM Finetune]](https://github.com/2U1/SmolVLM-Finetune)**

## Update

- [2025/02/11] 🔥Support all kind of mixed-modality (Video + Image + Text)
- [2025/02/05] Fixed code for properly use image
- [2025/02/03] Support Liger-kernel for Qwen2.5-VL
- [2025/02/03] 🔥Supports Qwen2.5-VL.
- [2025/01/24] Add option for using DoRA.
- [2025/01/24] Fix error in LoRA training.
- [2025/01/18] 🔥Supports mixed-modality data.
- [2025/01/11] Updated 8-bit training with ms_amp fp8 with opt_level O3.
- [2024/11/05] Add memory efficient 8-bit training.
- [2024/09/12] 🔥Now the model is trained using [Liger-Kernel](https://github.com/linkedin/Liger-Kernel).
- [2024/09/11] Supports setting different learning rates to projector and vision model.
- [2024/09/11] 🔥Supports multi-image and video training.

## Table of Contents

- [Fine-tuning Qwen2-VL Series](#fine-tuning-qwen2-vl-series)
  - [Other projects](#other-projects)
  - [Update](#update)
  - [Table of Contents](#table-of-contents)
  - [Supported Features](#supported-features)
  - [Installation](#installation)
    - [Using `environment.yaml`](#using-environmentyaml)
  - [Dataset Preparation](#dataset-preparation)
  - [Training](#training)
    - [Full Finetuning](#full-finetuning)
    - [Full Finetuning with 8-bit](#full-finetuning-with-8-bit)
    - [Finetune with LoRA](#finetune-with-lora)
    - [Train with video dataset](#train-with-video-dataset)
      - [Merge LoRA Weights](#merge-lora-weights)
      - [Image Resolution for performance boost](#image-resolution-for-performance-boost)
      - [Issue for libcudnn error](#issue-for-libcudnn-error)
  - [Inference](#inference)
    - [Gradio Infernce (WebUI)](#gradio-infernce-webui)
  - [TODO](#todo)
  - [Known Issues](#known-issues)
  - [License](#license)
  - [Citation](#citation)
  - [Acknowledgement](#acknowledgement)

## Supported Features

- Deepspeed
- LoRA/QLoRA
- Full-finetuning
- Enable finetuning `vision_model` while using LoRA.
- Disable/enable Flash Attention 2
- Multi-image and video training
- Training optimized with liger kernel

## Installation

Install the required packages using `environment.yaml`.

### Using `environment.yaml`

```bash
conda env create -f environment.yaml
conda activate qwen2
pip install qwen-vl-utils
pip install flash-attn==2.5.8 --no-build-isolation
```

Please install the latest transformers from git to finetune Qwen2.5-VL

```
pip install git+https://github.com/huggingface/transformers accelerate
```

**Note:** You should install flash-attn after installing the other packages.

## Dataset Preparation

The script requires a dataset formatted according to the LLaVA specification. The dataset should be a JSON file where each entry contains information about conversations and images. Ensure that the image paths in the dataset match the provided `--image_folder`.<br>

**When using a multi-image dataset, the image tokens should all be `<image>`, and the image file names should have been in a list.**<br><br>
**Please see the example below and follow format your data.**

<details>
<summary>Example for single image dataset</summary>

```json
[
  {
    "id": "000000033471",
    "image": "000000033471.jpg",
    "conversations": [
      {
        "from": "human",
        "value": "<image>\nWhat are the colors of the bus in the image?"
      },
      {
        "from": "gpt",
        "value": "The bus in the image is white and red."
      },
      {
        "from": "human",
        "value": "What feature can be seen on the back of the bus?"
      },
      {
        "from": "gpt",
        "value": "The back of the bus features an advertisement."
      },
      {
        "from": "human",
        "value": "Is the bus driving down the street or pulled off to the side?"
      },
      {
        "from": "gpt",
        "value": "The bus is driving down the street, which is crowded with people and other vehicles."
      }
    ]
  }
  ...
]
```

</details>

<details>
<summary>Example for multi image dataset</summary>

```json
[
  {
    "id": "000000033471",
    "image": ["000000033471.jpg", "000000033472.jpg"],
    "conversations": [
      {
        "from": "human",
        "value": "<image>\n<image>\nIs the perspective of the camera differnt?"
      },
      {
        "from": "gpt",
        "value": "Yes, It the perspective of the camera is different."
      }
    ]
  }
  ...
]
```

</details>

<details>
<summary>Example for video dataset</summary>

```json
[
  {
    "id": "sample1",
    "video": "sample1.mp4",
    "conversations": [
      {
        "from": "human",
        "value": "<video>\nWhat is going on in this video?"
      },
      {
        "from": "gpt",
        "value": "A man is walking down the road."
      }
    ]
  }
  ...
]
```

**Note:** Qwen2-VL uses a video as a sequential of images.

</details>
<br><br>

Adding the new domain-specific data on top of the general data from open-source data will enhance downstream capabilities while retaining the foundational skills. Of course, you can also choose to fine-tune solely on the new data based on your requirements.

## Training

**Note:** Deepspedd zero2 is faster than zero3, however it consumes more memory. Also, most of the time zero2 is more stable than zero3.

To run the training script, use the following command:

### Full Finetuning

```bash
bash scripts/finetune.sh
```

### Full Finetuning with 8-bit

```bash
bash scripts/finetune_8bit.sh
```

**You need to install [ms-amp](https://github.com/Azure/MS-AMP) to use this script.**<br>
This script will finetune the model with fp8 model dtype. If you run out of vram, you could use this.<br>
You can even use offloading with fp8 training. For detailed config, you could change the deepspeed config files.

### Finetune with LoRA

If you want to train only the language model with LoRA and perform full training for the vision model:

```bash
bash scripts/finetune_lora.sh
```

If you want to train both the language model and the vision model with LoRA:

```bash
bash scripts/finetune_lora_vision.sh
```

**IMPORTANT:** If you want to tune the `embed_token` with LoRA, You need to tune `lm_head` together.

<details>
<summary>Training arguments</summary>

- `--deepspeed` (str): Path to DeepSpeed config file (default: "scripts/zero2.json").
- `--data_path` (str): Path to the LLaVA formatted training data (a JSON file). **(Required)**
- `--image_folder` (str): Path to the images folder as referenced in the LLaVA formatted training data. **(Required)**
- `--model_id` (str): Path to the Qwen2-VL model. **(Required)**
- `--output_dir` (str): Output directory for model checkpoints
- `--num_train_epochs` (int): Number of training epochs (default: 1).
- `--per_device_train_batch_size` (int): Training batch size per GPU per forwarding step.
- `--gradient_accumulation_steps` (int): Gradient accumulation steps (default: 4).
- `--freeze_vision_tower` (bool): Option to freeze vision_model (default: False).
- `--freeze_llm` (bool): Option to freeze LLM (default: False).
- `--tune_merger` (bool): Option to tune projector (default: True).
- `--num_lora_modules` (int): Number of target modules to add LoRA (-1 means all layers).
- `--vision_lr` (float): Learning rate for vision_model.
- `--merger_lr` (float): Learning rate for merger(projector).
- `--learning_rate` (float): Learning rate for language module.
- `--bf16` (bool): Option for using bfloat16.
- `--fp16` (bool): Option for using fp16.
- `--image_min_pixels` (int): Option for minimum input tokens for image.
- `--image_max_pixles` (int): Option for maximum maxmimum tokens for image.
- `--video_min_pixels` (int): Option for minimum input tokens for video.
- `--video_max_pixles` (int): Option for maximum maxmimum tokens for video.
- `--lora_enable` (bool): Option for using LoRA.
- `--vision_lora` (bool): Option for including `vision_tower` in LoRA module. `lora_enable` should be `True` to use this option.
- `--use_dora` (bool): Option for using DoRA instead of LoRA. `lora_enable` should be `True` to use this option.
- `--lora_namespan_exclude` (str): Exclude modules with namespans to add LoRA.
- `--max_seq_length` (int): Maximum sequence length (default: 32K).
- `--bits` (int): Quantization bits (default: 16).
- `--disable_flash_attn2` (bool): Disable Flash Attention 2.
- `--report_to` (str): Reporting tool (choices: 'tensorboard', 'wandb', 'none') (default: 'tensorboard').
- `--logging_dir` (str): Logging directory (default: "./tf-logs").
- `--lora_rank` (int): LoRA rank (default: 128).
- `--lora_alpha` (int): LoRA alpha (default: 256).
- `--lora_dropout` (float): LoRA dropout (default: 0.05).
- `--logging_steps` (int): Logging steps (default: 1).
- `--dataloader_num_workers` (int): Number of data loader workers (default: 4).

**Note:** The learning rate of `vision_model` should be 10x ~ 5x smaller than the `language_model`.

</details>

### Train with video dataset

You can train the model using a video dataset. You can set LoRA configs and use for LoRA too.<br>
**IMPORTANT:** If your dataset are composed with (image + video) please use [zero2](./scripts/zero2.json)

```bash
bash scripts/finetune_video.sh
```

**Note:** When training with video, it just as multi-image so you should adjust the `max_pixels` for maximum resolution and `fps` based on the available VRAM.

If you run out of vram, you can use [zero3_offload](./scripts/zero3_offload.json) instead of [zero3](./scripts/zero3_offload.json).<br>
You could use [zero2_offload](./scripts/zero2_offload.json) for mixed-modality data.

#### Merge LoRA Weights

```
bash scripts/merge_lora.sh
```

**Note:** Remember to replace the paths in `finetune.sh` or `finetune_lora.sh` with your specific paths. (Also in `merge_lora.sh` when using LoRA.)

#### Image Resolution for performance boost

The model supprots a wide range of resolution inputs. By default, it uses the native resolution for input.
For better performance using native or higer pixel numbers are recommended, however it takes too much memory and computation time for large images. So you could adjust the pixel numbers for it.
The model splits the image into `token * 28 * 28` so you could just change the the token_num part in the script. <br>
For example:

```
image_min_pixels = 256 * 28 * 28
image_max_pixels = 1280 * 28 * 28
video_min_pixels = 128 * 28 * 28
video_max_pixels = 768 * 28 * 28
```

#### Issue for libcudnn error

```
Could not load library libcudnn_cnn_train.so.8. Error: /usr/local/cuda-12.1/lib/libcudnn_cnn_train.so.8: undefined symbol: _ZN5cudnn3cnn34layerNormFwd_execute_internal_implERKNS_7backend11VariantPackEP11CUstream_stRNS0_18LayerNormFwdParamsERKNS1_20NormForwardOperationEmb, version libcudnn_cnn_infer.so.8
```

You could run `unset LD_LIBRARY_PATH` for this error.
You could see this [issue](https://github.com/andimarafioti/florence2-finetuning/issues/2)

## Inference

**Note:** You should use the merged weight when trained with LoRA.

### Gradio Infernce (WebUI)

1. Install gradio

```
pip install gradio
```

2. Launch app

```
python -m src.serve.app \
    --model-path /path/to/merged/weight
```

You can launch gradio based demo with this command. This can also set some other generation configs like `repetition_penalty`, `temperature` etc.

## TODO

- [x] Support for video data
- [x] Add demo for multi-image and video
- [x] Handle mixed-modality data in dataset and collator
- [x] Support Qwen2.5-VL
- [x] Monkey-patch liger-kernel for Qwen2.5-VL

## Known Issues

- [libcudnn issue](#issue-for-libcudnn-error)

## License

This project is licensed under the Apache-2.0 License. See the [LICENSE](LICENSE) file for details.

## Citation

If you find this repository useful in your project, please consider giving a :star: and citing:

```bibtex
@misc{Qwen2-VL-Finetuning,
  author = {Yuwon Lee},
  title = {Qwen2-VL-Finetune},
  year = {2024},
  publisher = {GitHub},
  url = {https://github.com/2U1/Qwen2-VL-Finetune}
}
```

## Acknowledgement

This project is based on

- [LLaVA-NeXT](https://github.com/LLaVA-VL/LLaVA-NeXT): An amazing open-source project of LMM.
- [Mipha](https://github.com/zhuyiche/llava-phi): Open-source projcet of SMM with amazing capabilites.
- [Qwen2-VL-7B-Instruct](https://huggingface.co/Qwen/Qwen2-VL-7B-Instruct): Awesome pretrained MLLM based on Qwen2.
- [Liger-Kernel](https://github.com/linkedin/Liger-Kernel): Collection of Tirton kernels designed specifically for LLM training.
