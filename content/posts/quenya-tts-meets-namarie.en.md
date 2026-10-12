+++
date = 2026-10-12T20:00:00+01:00
categories = ['Quenya']
title = "Teaching a Machine to Say Namárië"
+++

In my very [first Quenya post](/posts/paran-quenya-part-1/) (so proud of myself!), I listed [QuenyaTTS](https://annie-zhou1997.github.io/QuenyaTTS.github.io/) among the tools worth knowing: a text-to-speech system for Quenya built by [Annie Zhou](https://github.com/Annie-Zhou1997/) for her 2024 master's thesis at the University of Groningen.

I found this project quite interesting and thought that perhaps I could improve it by trying a few different techniques. First, I mined some issues of Parma Eldalamberon (published by the [Elvish Linguistic Fellowship](https://www.elvish.org/)) for pronunciation rules that Annie's code might be missing. Second, I tried to extract the phonetics from a recording of Tolkien himself reciting Namárië, [published by the Romanian Tolkien Society](https://www.youtube.com/watch?v=mVACqgDqLno).

This post is about what changed, and about what happened when I put the synthetic voice next to Tolkien's.

## How Annie's system works

A text-to-speech system like this one works in two steps:

1. **From text to sounds.** A set of rules turns written Quenya into phonetic symbols (IPA). *Elen síla lúmenn' omentielvo* becomes `ˈɛlɛn sˈiːla lˈuːmɛnn ɔmɛntiˈɛlvɔ`: which sounds to make and where the stress falls.
2. **From sounds to audio.** A neural model, [IMS-Toucan](https://github.com/DigitalPhonetics/IMS-Toucan) from the University of Stuttgart, turns those symbols into a voice.

## What changed in Annie's code

I used Claude to find gaps in how Annie's code converts Quenya into the International Phonetic Alphabet. The rules were rewritten and checked against Appendix E of The Lord of the Rings, [Eldamo](https://eldamo.org/), several published pronunciation guides, and Tolkien's own notes in Parma Eldalamberon. Some examples: *hr* and *hl* are read as plain *r* and *l*, *w* becomes *v* only at the start of a word, archaic spellings such as *þ* are read with their Third Age sound, and stress follows the Appendix E rule (never the last syllable; the second-to-last if it is heavy, otherwise the third-to-last). Every rule now has a test.

Annie's original project contained a modified copy of an older Toucan. Now the [Quenya-TTS repository](https://github.com/brunopacheco1/Quenya-TTS) holds only the Quenya parts, and Toucan comes in unchanged, at a fixed version. Quenya is given to Toucan as phonetic symbols, and since Toucan has never heard Quenya, it borrows the settings of Finnish, Tolkien's own inspiration for the language.

Toucan starts speaking at the very first moment of the file, so the first syllable came out clipped. The tool now adds a short silent pause before the first sound. A side effect was that the model filled that pause with a little noise, heard as an extra *k* or *p* before the first word, so that pause is now muted.

## Comparing with Tolkien

Tolkien recorded himself reciting Namárië. I wrote a small script that compares any recording with our output, line by line:

- A speech recogniser for many languages ([wav2vec2 XLSR](https://huggingface.co/facebook/wav2vec2-xlsr-53-espeak-cv-ft)) writes down which sounds it hears in both recordings. Comparing that with what our rules ask for gives a *phone error rate*: the lower, the closer to the rules.
- Both melodies are drawn on top of each other, in semitones.

There are two caveats. The recogniser is not perfect; for example, it hears many open *e* sounds as closed ones. And Tolkien's tape is old and noisy. So the numbers are a guide, not a grade.

Once Tolkien's recording was correctly cut into the lines of Namárië, I ran the script three times, each round fixing what the previous one showed:

| Round | What changed | Error rate (ours) | Length vs Tolkien |
|---|---|---|---|
| 1 | first comparison | 27% | 0.86 |
| 2 | slower, flatter melody | 24% | 0.99 |
| 3 | silent pause before the first word | 22% | 0.97 |

Tolkien's own recording scores 33% against our rules, but listening to it, you can tell that the recogniser misses some of his phonemes. Manual corrections could probably improve that rate, but the purpose of this exercise was to find big pronunciation gaps to fix in Annie's code.

![Line 1, Ai! laurië lantar lassi súrinen: spectrograms of Tolkien and of Quenya-TTS, and both melodies](/images/quenya-tts-namarie/line01.png)

## What I learned

**The rules agree with Tolkien.** Where the recogniser hears Tolkien clearly, he says what the rules predict. The one interesting difference is *ly*, as in *ilyë* and *hiruvalyë*: the rules use a single sound, but in Tolkien's voice the recogniser hears two phonemes, *l* followed by *y*. The sources leave both readings open (in Tengwar, *ly* is one letter with two dots below), so I kept the single sound.

**The remaining errors are probably the model's.** I don't have access to Annie's trained model, nor enough Quenya recordings to train it myself. The main remaining issues are:

- the model can't make the soft *l*, so *ilyë* comes out as "ihe";
- a final *r* is weak and gets heard as *l* (*Valimar*, *tellumar*), where it should be trilled;
- the *d* of *undulávë* drifts toward *b*, between two rounded *u*.

![Line 16, Namárië! Nai hiruvalyë Valimar: the soft l of hiruvalyë is where the model struggles most](/images/quenya-tts-namarie/line16.png)

## What comes next

The next step is to record volunteers reading Quenya sentences to train the model: mostly everyday dialogue, plus some poems. If you would like to lend your voice, see the [volunteer voices issue](https://github.com/brunopacheco1/ma-quetelye-quenya/issues/38). The recordings will be published under an open licence, so the trained model can be shared freely.

Please let me know if you have any feedback.

**Namárië | See you | Bis geschwënn | Até mais | À bientôt**
