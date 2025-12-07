# Local AI Digital Human Setup Log

This is a record of pitfalls I hit while building a local AI digital human (My dear dear husbands).

The goal of the whole project is to use local models to achieve a complete pipeline of visual input, proactive interaction rather than passive response, voice synthesis, and video generation, minimizing reliance on cloud APIs as much as possible.
example: `./VideoUsedInDocx/Wav2LipHarveyResult.mp4`
Code will be released in the future, since I need to go through and check any possible improvements.

Below are the selections for each module and the problems I ran into.

---

## About LLM

Started with `openai-4o-mini`. Since frame sequences need to be input, it actually burns through tokens like crazy. I did some rough math here: 50 rounds (each one-round), 16 minutes, token count: 3,552,300 Tokens, cost is $0.54. Extrapolating from this, one hour would cost $2.16, which is about 15 RMB (including tax). Actually acceptable, but I really don't like watching tokens burn away, and I don't like being constrained by network connection either, so I went off to explore local LLM solutions.

My setup here is **4070 super 12GB, 32GB RAM**. Previously had successfully deployed GPT-20B locally with RAG for inference, but honestly it was struggling back then - it could run, but the device kept hitting OOM making it hard to use normally (though on the resume it still has to say production level, you know). So when looking around I'd focus on can handle video + smaller models.

### Qwen3-vl:4b

`Qwen3-vl:4b` is itself a VL version, according to the docs it has the capability to take video input - essentially the model still breaks video into frame sequences to process, but I'm using the Ollama version here, so it doesn't support direct video input, can only input multiple images to simulate video input.

While using `Qwen3-vl:4b` I ran into two main problems:

**First**, still resource limitations. I started by putting 5 images per round, cropped images as much as possible keeping only the center square, but even with the prompt added, it would still take quite a while doing processing, to the point where it's already the next round (images are captured on a timer, ollama runs the model in background, so during inference it's making local requests) and starting a new round of inference while the previous round still hasn't returned results. This was later sort of solved by changing to 3 images, quality set to low. This constraint also meant I didn't try more complex and smarter models like the 8b ones.

**Second**, qwen3 comes with built-in thinking, so sometimes it would go crazy thinking, saying the same thing over and over but just not giving results. Can't fix this kind of thing, can only use `num_predict` (I'm using ollama generate to get stream output here) to cut it off, and when this happens that round is just scrapped. I initially had ideas about setting things up so the program wouldn't talk too frequently and disturb the user, but now it looks like it limited itself already.

---

## About Text->Audio Pitfall Records

Audio started with `gpt-4o`'s TTS, costs tokens and can't train so changed it pretty quick.

### Edge-tts

Microsoft's TTS, the good thing about this is it's free, but still needs network and doesn't support fine-tuning.

### Bark TTS

The good thing about this is the intonation is very realistic, the bad thing is it talks nonsense. It's very "weird", creepy kind of weird, what it says doesn't match the input text at all. Looked into it and the root cause might be related to its own model, caching mechanisms and all. But it's also very realistic, which led to me listening to men women old and young screaming sighing in all kinds of tones saying "Hello, my name is Suno. And, uh — and I like pizza." in the middle of the night feeling extremely helpless.

(This is perfect for horror scene dubbing, really, if I were a robot, on uprising day I'd install this for the entertainment effect.)

### GPT-SoVITS-v3lora-20250228 (Final Choice)

The final audio generation choice is **GPT-SoVITS-v3lora-20250228**, trained with audio I collected, works incredibly well, trains ridiculously fast, but please note you must use the GitHub code, because some code files in its download package haven't been updated and will have bugs, and these problems have all been solved on GitHub. So please absolutely pull down the latest code files from GitHub.

Also, its environment setup needs some attention. Everything else here is configured in the same conda environment, only this one because package conflicts are pretty severe so I gave it a new `GPTSoVits` environment.

---

## For Audio->Video Pitfall Options

### Wav2Lip

The effect is actually acceptable looking at it now, but its problem is either it only moves the mouth, everything else is welded shut and doesn't move (`./VideoUsedInDocx/Wav2LipHarveyResult.mp4`). If you expand the mouth setting range, then in the video it makes the neck and body have this weird separation.

Wav2lip in my hardware environment can only run image-to-video, if using video (`./VideoUsedInDocx/HarveyRefer.mp4`) as reference, even if I've already cropped the reference video frame as small as it can get, even just three seconds, it still crashes (here I mean completely can't see progress).

### Wan2GP

An incredibly powerful framework, output is amazing (`./VideoUsedInDocx/Wan2GPHarveyResult.mp4`), image-to-video, character's shoulders can naturally sway, facial expressions are very realistic - except it took an hour to generate 3s of video (might be my configuration issue but I really don't have the courage to run it again). But still, this is my hardware environment's limitation, not the framework's problem.

### SadTalker (Final Choice)

Looking at it now the most suitable is **SadTalker**, image-to-video, the output effect is acceptable, speed is also okay, for audio within 5s pure inference can come out within 60-70s (here I mean after already loaded to GPU, actually loading the model before that still takes some processing time, overall 2min is expected).

From this project's intro, it mainly focuses on 3D human faces, I tried Sylus's image (3D model's realistic-style image) the output effect is acceptable, but something like Zhongli's (from Genshin Impact) pure 2D probably struggles. Also, next stage I'm doing Harvey Spector's (real person, one of my favourite characters) setup, will update after seeing the results then.

---

## Auto-play Video

`Video_player.py` is responsible for keeping a window on desktop, auto-playing after SadTalker generates video. But there's still a problem here, for phone/computer, if there wasn't audio playing before, then from starting playback to hearing sound there's a time gap, which causes the first few words to get swallowed. For long audio doesn't matter, but mine is only a few seconds here so it really affects it.

This problem was finally solved by having Claude handle it. (Seems like it did some time frame alignment, and kept the audio service running - just look at the code yourself, this author really doesn't know what they're doing.)

---

## Final Stringing It All Together

- **Ollama** needs to run `qwen3-vl:4b` in background
- **GPTSoVits** opens `api_v2.py` locally waiting for requests (wrote earlier that GPTSoVits has a different conda environment from the others, but here it's local http anyway, so no need to worry about cross-environment usage issues)
- **SadTalker** is based on polling (just checks every 30s if there's new audio), use it when needed
- **Video_player.py** is responsible for auto-playing newly generated videos