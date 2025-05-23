# MetaVoice-1B

[![Playground](https://img.shields.io/static/v1?label=Try&message=Playground&color=fc4982&url=https://ttsdemo.themetavoice.xyz/)](https://ttsdemo.themetavoice.xyz/)
<a target="_blank" style="display: inline-block; vertical-align: middle" href="https://colab.research.google.com/github/MGSousa/metavoice-src/blob/main/colab_demo.ipynb">
  <img src="https://colab.research.google.com/assets/colab-badge.svg" alt="Open In Colab"/>
</a>
[![](https://dcbadge.vercel.app/api/server/Cpy6U3na8Z?style=flat&compact=True)](https://discord.gg/tbTbkGEgJM)
[![Twitter](https://img.shields.io/twitter/url/https/twitter.com/OnusFM.svg?style=social&label=@metavoiceio)](https://twitter.com/metavoiceio)



MetaVoice-1B is a 1.2B parameter base model trained on 100K hours of speech for TTS (text-to-speech). It has been built with the following priorities:
* **Emotional speech rhythm and tone** in English.
* **Zero-shot cloning for American & British voices**, with 30s reference audio.
* Support for (cross-lingual) **voice cloning with finetuning**.
  * We have had success with as little as 1 minute training data for Indian speakers.
* Synthesis of **arbitrary length text**

We’re releasing MetaVoice-1B under the Apache 2.0 license, *it can be used without restrictions*.


## Quickstart - tl;dr

Web UI
```bash
docker-compose up -d ui && docker-compose ps && docker-compose logs -f
```

Server
```bash
# navigate to <URL>/docs for API definitions
docker-compose up -d server && docker-compose ps && docker-compose logs -f
```

## Installation

**Pre-requisites:**
- GPU VRAM >=12GB
- Python >=3.10,<3.12
- pipx ([installation instructions](https://pipx.pypa.io/stable/installation/))

**Environment setup**
```bash
# install ffmpeg
wget https://johnvansickle.com/ffmpeg/builds/ffmpeg-git-amd64-static.tar.xz
wget https://johnvansickle.com/ffmpeg/builds/ffmpeg-git-amd64-static.tar.xz.md5
md5sum -c ffmpeg-git-amd64-static.tar.xz.md5
tar xvf ffmpeg-git-amd64-static.tar.xz
sudo mv ffmpeg-git-*-static/ffprobe ffmpeg-git-*-static/ffmpeg /usr/local/bin/
rm -rf ffmpeg-git-*

# install rust if not installed (ensure you've restarted your terminal after installation)
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

### Project dependencies installation
1. [Using poetry](#using-poetry-recommended)
2. [Using pip/conda](#using-pipconda)

#### Using poetry (recommended)
```bash
# install poetry if not installed (ensure you've restarted your terminal after installation)
pipx install poetry

# disable any conda envs that might interfere with poetry's venv
conda deactivate

# if running from Linux, keyring backend can hang on `poetry install`. This prevents that.
export PYTHON_KEYRING_BACKEND=keyring.backends.fail.Keyring

# pip's dependency resolver will complain, this is temporary expected behaviour
# full inference & finetuning functionality will still be available
poetry install && poetry run pip install torch==2.2.1 torchaudio==2.2.1
```

#### Using pip/conda
NOTE 1: When raising issues, we'll ask you to try with poetry first.
NOTE 2: All commands in this README use `poetry` by default, so you can just remove any `poetry run`.

```bash
pip install -r requirements.txt
pip install torch==2.2.1 torchaudio==2.2.1
pip install -e .
```

## Usage
1. Download it and use it anywhere (including locally) with our [reference implementation](/fam/llm/fast_inference.py)
```bash
# You can use `--quantisation_mode int4` or `--quantisation_mode int8` for experimental faster inference.  This will degrade the quality of the audio.
# Note: int8 is slower than bf16/fp16 for undebugged reasons. If you want fast, try int4 which is roughly 2x faster than bf16/fp16.
poetry run python -i fam/llm/fast_inference.py

# Run e.g. of API usage within the interactive python session
tts.synthesise(text="This is a demo of text to speech by MetaVoice-1B, an open-source foundational audio model.", spk_ref_path="assets/bria.mp3")
```
> Note: The script takes 30-90s to startup (depending on hardware). This is because we torch.compile the model for fast inference.

> On Ampere, Ada-Lovelace, and Hopper architecture GPUs, once compiled, the synthesise() API runs faster than real-time, with a Real-Time Factor (RTF) < 1.0.

2. Deploy it on any cloud (AWS/GCP/Azure), using our [inference server](serving.py) or [web UI](app.py)
```bash
# You can use `--quantisation_mode int4` or `--quantisation_mode int8` for experimental faster inference. This will degrade the quality of the audio.
# Note: int8 is slower than bf16/fp16 for undebugged reasons. If you want fast, try int4 which is roughly 2x faster than bf16/fp16.

# navigate to <URL>/docs for API definitions
poetry run python serving.py

poetry run python app.py
```

3. Use it via [Hugging Face](https://huggingface.co/metavoiceio)
4. [Google Colab Demo](https://colab.research.google.com/github/metavoiceio/metavoice-src/blob/main/colab_demo.ipynb)

## Finetuning
We support finetuning the first stage LLM (see [Architecture section](#Architecture)).

In order to finetune, we expect a "|"-delimited CSV dataset of the following format:

```csv
audio_files|captions
./data/audio.wav|./data/caption.txt
```

Note that we don't perform any dataset overlap checks, so ensure that your train and val datasets are disjoint.

Try it out using our sample datasets via:
```bash
poetry run finetune --train ./datasets/sample_dataset.csv --val ./datasets/sample_val_dataset.csv
```

Once you've trained your model, you can use it for inference via:
```bash
poetry run python -i fam/llm/fast_inference.py --first_stage_path ./my-finetuned_model.pt
```

### Configuration

In order to set hyperparameters such as learning rate, what to freeze, etc, you
can edit the [finetune_params.py](./fam/llm/config/finetune_params.py) file.

We've got a light & optional integration with W&B that can be enabled via setting
`wandb_log = True` & by installing the appropriate dependencies.

```bash
poetry install -E observable
```

## Upcoming
- [x] Faster inference ⚡
- [x] Fine-tuning code 📐
- [ ] Synthesis of arbitrary length text


## Architecture
We predict EnCodec tokens from text, and speaker information. This is then diffused up to the waveform level, with post-processing applied to clean up the audio.

* We use a causal GPT to predict the first two hierarchies of EnCodec tokens. Text and audio are part of the LLM context. Speaker information is passed via conditioning at the token embedding layer. This speaker conditioning is obtained from a separately trained speaker verification network.
  - The two hierarchies are predicted in a "flattened interleaved" manner, we predict the first token of the first hierarchy, then the first token of the second hierarchy, then the second token of the first hierarchy, and so on.
  - We use condition-free sampling to boost the cloning capability of the model.
  - The text is tokenised using a custom trained BPE tokeniser with 512 tokens.
  - Note that we've skipped predicting semantic tokens as done in other works, as we found that this isn't strictly necessary.
* We use a non-causal (encoder-style) transformer to predict the rest of the 6 hierarchies from the first two hierarchies. This is a super small model (~10Mn parameters), and has extensive zero-shot generalisation to most speakers we've tried. Since it's non-causal, we're also able to predict all the timesteps in parallel.
* We use multi-band diffusion to generate waveforms from the EnCodec tokens. We noticed that the speech is clearer than using the original RVQ decoder or VOCOS. However, the diffusion at waveform level leaves some background artifacts which are quite unpleasant to the ear. We clean this up in the next step.
* We use DeepFilterNet to clear up the artifacts introduced by the multi-band diffusion.

## Optimizations
The model supports:
1. KV-caching via Flash Decoding
2. Batching (including texts of different lengths)

## Contribute
- See all [active issues](https://github.com/metavoiceio/metavoice-src/issues)!

## Acknowledgements
We are grateful to Together.ai for their 24/7 help in marshalling our cluster. We thank the teams of AWS, GCP & Hugging Face for support with their cloud platforms.

- [A Défossez et. al.](https://arxiv.org/abs/2210.13438) for Encodec.
- [RS Roman et. al.](https://arxiv.org/abs/2308.02560) for Multiband Diffusion.
- [@liusongxiang](https://github.com/liusongxiang/ppg-vc/blob/main/speaker_encoder/inference.py) for speaker encoder implementation.
- [@karpathy](https://github.com/karpathy/nanoGPT) for NanoGPT which our inference implementation is based on.
- [@Rikorose](https://github.com/Rikorose) for DeepFilterNet.

Apologies in advance if we've missed anyone out. Please let us know if we have.
