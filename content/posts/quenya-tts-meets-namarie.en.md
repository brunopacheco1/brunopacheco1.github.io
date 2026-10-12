+++
date = 2026-10-12T20:00:00+01:00
categories = ['Quenya']
title = "Teaching a Machine to Say Namárië"
+++

In my [first Quenya post](/posts/paran-quenya-part-1/) I listed [QuenyaTTS](https://annie-zhou1997.github.io/QuenyaTTS.github.io/) among the tools worth knowing: a text-to-speech system for Quenya built by Annie Zhou for her 2024 master's thesis at the University of Groningen. Since then I have been working on it, because I want audio for [Ma Quetelyë Quenya?](https://ma-quetelye-quenya.tolkien.lu/), the community book we are writing to teach Quenya. This post is about what changed, and about what happened when I put our synthetic voice next to the most authoritative Quenya speaker there is: Tolkien himself, reciting Namárië.

## How the system works

A text-to-speech system like this one works in two steps:

1. **From text to sounds.** A set of rules turns written Quenya into phonetic symbols (IPA). *Elen síla lúmenn' omentielvo* becomes `ˈɛlɛn sˈiːla lˈuːmɛnn ɔmɛntiˈɛlvɔ`: which sounds to make and where the stress falls.
2. **From sounds to audio.** A neural model, [IMS-Toucan](https://github.com/DigitalPhonetics/IMS-Toucan) from the University of Stuttgart, turns those symbols into a voice.

## What changed

**The rules now follow Appendix E.** I wanted the pronunciation of the Third Age, as Tolkien describes it in Appendix E of The Lord of the Rings. The rules were rewritten and checked against Appendix E, [Eldamo](https://eldamo.org/), several pronunciation guides and Tolkien's own notes published in Parma Eldalamberon. Some examples: *hr* and *hl* are read as plain *r* and *l*, *w* becomes *v* only at the start of a word, archaic spellings such as *þ* are read with their Third Age sound, and stress follows the Appendix E rule (never the last syllable; the second-to-last if it is heavy, otherwise the third-to-last). Every rule has a test.

**Toucan is no longer copied.** The original project contained a modified copy of an older Toucan. Now the [Quenya-TTS repository](https://github.com/brunopacheco1/Quenya-TTS) holds only the Quenya parts, and Toucan comes in unchanged, at a fixed version. Quenya is given to Toucan as phonetic symbols, and since Toucan has never heard Quenya it borrows the settings of Finnish, Tolkien's own inspiration for the language.

**The first syllable is no longer swallowed.** Toucan starts speaking at the very first moment of the file, so the first syllable came out clipped. The reader now adds a short silent pause before the first sound. A side effect was that the model filled that pause with a little noise, heard as an extra *k* or *p* before the first word, so that pause is now muted.

## Comparing with Tolkien

Tolkien recorded himself reading and chanting Namárië. I have a copy, which I can't share here, and wrote a small tool that compares any recording with our output, line by line:

- A speech recogniser for many languages ([wav2vec2 XLSR](https://huggingface.co/facebook/wav2vec2-xlsr-53-espeak-cv-ft)) writes down which sounds it hears, in both recordings. Comparing that with what our rules ask for gives a *phone error rate*: the lower, the closer to the rules.
- Both melodies are drawn on top of each other, in semitones.

Two caveats. The recogniser is not perfect; it hears many open *e* as closed *e*, for example. And Tolkien's tape is old and noisy. So the numbers are a guide, not a grade.

Once the tool cut the lines correctly, I ran it three times, each round fixing what the previous one showed:

| Round | What changed | Error rate (ours) | Length vs Tolkien |
|---|---|---|---|
| 1 | first comparison | 27% | 0.86 |
| 2 | slower, flatter melody | 24% | 0.99 |
| 3 | silent pause before the first word | 22% | 0.97 |

Tolkien's own recording scores 33% against our rules, mostly because of the noise and the recogniser, not because he disagrees with them.

![Line 1, Ai! laurië lantar lassi súrinen: spectrograms of Tolkien and of Quenya-TTS, and both melodies](/images/quenya-tts-namarie/line01.png)

## What we learned

**The rules agree with Tolkien.** Where the recogniser hears Tolkien clearly, he says what the rules predict. The one interesting difference is *ly*, as in *ilyë* and *hiruvalyë*: Tolkien seems to say *l* followed by *y*, while the rules use a single sound, a "soft" *l*, as in Italian *gli*. The written sources leave both readings open (in Tengwar, *ly* is one letter with two dots below, but the same mark is used for *ry* and *sty*). I decided to keep the single sound.

**The remaining errors are the model's.** The model has never heard Quenya, and it shows:

- it can't make the soft *l*, so *ilyë* comes out as "ihe";
- a final *r* is weak and gets heard as *l* (*Valimar*, *tellumar*), where it should be trilled;
- the *d* of *undulávë* drifts toward *b*, between two rounded *u*.

**Chanting is not speaking.** Tolkien chants Namárië slowly and almost on one note. Our model starts each line high and falls, which is how a normal sentence sounds. For poems, the reader now has a setting for a flatter melody; for the book's dialogues, the normal one is right.

![Line 16, Namárië! Nai hiruvalyë Valimar: the soft l of hiruvalyë is where the model struggles most](/images/quenya-tts-namarie/line16.png)

## What comes next

All of the remaining errors have the same cure: let the model hear real Quenya. The next step is recording volunteers reading Quenya sentences, mostly everyday dialogue, plus some poems. If you would like to lend your voice, see the [volunteer voices issue](https://github.com/brunopacheco1/ma-quetelye-quenya/issues/38). Recordings will be published under an open licence, so the trained model can be shared freely.

**Namárië | See you | Bis geschwënn | Até mais | À bientôt**
