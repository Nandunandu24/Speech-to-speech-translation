Speech-to-speech translation (STST or S2ST) is a relatively new spoken language processing task. It involves translating speech from one langauge into speech in a different language:

Diagram of speech to speech translation
STST can be viewed as an extension of the traditional machine translation (MT) task: instead of translating text from one language into another, we translate speech from one language into another. STST holds applications in the field of multilingual communication, enabling speakers in different languages to communicate with one another through the medium of speech.

Suppose you want to communicate with another individual across a langauge barrier. Rather than writing the information that you want to convey and then translating it to text in the target language, you can speak it directly and have a STST system convert your spoken speech into the target langauge. The recipient can then respond by speaking back at the STST system, and you can listen to their response. This is a more natural way of communicating compared to text-based machine translation.

In this chapter, we’ll explore a cascaded approach to STST, piecing together the knowledge you’ve acquired in Units 5 and 6 of the course. We’ll use a speech translation (ST) system to transcribe the source speech into text in the target language, then text-to-speech (TTS) to generate speech in the target language from the translated text:

Diagram of cascaded speech to speech translation
We could also have used a three stage approach, where first we use an automatic speech recognition (ASR) system to transcribe the source speech into text in the same language, then machine translation to translate the transcribed text into the target language, and finally text-to-speech to generate speech in the target language. However, adding more components to the pipeline lends itself to error propagation, where the errors introduced in one system are compounded as they flow through the remaining systems, and also increases latency, since inference has to be conducted for more models.

While this cascaded approach to STST is pretty straightforward, it results in very effective STST systems. The three-stage cascaded system of ASR + MT + TTS was previously used to power many commercial STST products, including Google Translate. It’s also a very data and compute efficient way of developing a STST system, since existing speech recognition and text-to-speech systems can be coupled together to yield a new STST model without any additional training.

In the remainder of this Unit, we’ll focus on creating a STST system that translates speech from any language X to speech in English. The methods covered can be extended to STST systems that translate from any language X to any langauge Y, but we leave this as an extension to the reader and provide pointers where applicable. We further divide up the task of STST into its two constituent components: ST and TTS. We’ll finish by piecing them together to build a Gradio demo to showcase our system.

Speech translation
We’ll use the Whisper model for our speech translation system, since it’s capable of translating from over 96 languages to English. Specifically, we’ll load the Whisper Base checkpoint, which clocks in at 74M parameters. It’s by no means the most performant Whisper model, with the largest Whisper checkpoint being over 20x larger, but since we’re concatenating two auto-regressive systems together (ST + TTS), we want to ensure each model can generate relatively quickly so that we get reasonable inference speed:

Copied
import torch
from transformers import pipeline

device = "cuda:0" if torch.cuda.is_available() else "cpu"
pipe = pipeline(
    "automatic-speech-recognition", model="openai/whisper-base", device=device
)
Great! To test our STST system, we’ll load an audio sample in a non-English language. Let’s load the first example of the Italian (it) split of the VoxPopuli dataset:

Copied
from datasets import load_dataset

dataset = load_dataset("facebook/voxpopuli", "it", split="validation", streaming=True)
sample = next(iter(dataset))
To listen to this sample, we can either play it using the dataset viewer on the Hub: facebook/voxpopuli/viewer

Or playback using the ipynb audio feature:

Copied
from IPython.display import Audio

Audio(sample["audio"]["array"], rate=sample["audio"]["sampling_rate"])
Now let’s define a function that takes this audio input and returns the translated text. You’ll remember that we have to pass the generation key-word argument for the "task", setting it to "translate" to ensure that Whisper performs speech translation and not speech recognition:

Copied
def translate(audio):
    outputs = pipe(audio, max_new_tokens=256, generate_kwargs={"task": "translate"})
    return outputs["text"]
Whisper can also be ‘tricked’ into translating from speech in any language X to any language Y. Simply set the task to "transcribe" and the "language" to your target language in the generation key-word arguments, e.g. for Spanish, one would set:

generate_kwargs={"task": "transcribe", "language": "es"}

Great! Let’s quickly check that we get a sensible result from the model:

Copied
translate(sample["audio"].copy())
Copied
' psychological and social. I think that it is a very important step in the construction of a juridical space of freedom, circulation and protection of rights.'
Alright! If we compare this to the source text:

Copied
sample["raw_text"]
Copied
'Penso che questo sia un passo in avanti importante nella costruzione di uno spazio giuridico di libertà di circolazione e di protezione dei diritti per le persone in Europa.'
We see that the translation more or less lines up (you can double check this using Google Translate), barring a small extra few words at the start of the transcription where the speaker was finishing off their previous sentence.

With that, we’ve completed the first half of our cascaded STST pipeline, putting into practice the skills we gained in Unit 5 when we learnt how to use the Whisper model for speech recognition and translation. If you want a refresher on any of the steps we covered, have a read through the section on Pre-trained models for ASR from Unit 5.

Text-to-speech
The second half of our cascaded STST system involves mapping from English text to English speech. For this, we’ll use the pre-trained SpeechT5 TTS model for English TTS. 🤗 Transformers currently doesn’t have a TTS pipeline, so we’ll have to use the model directly ourselves. This is no biggie, you’re all experts on using the model for inference following Unit 6!

First, let’s load the SpeechT5 processor, model and vocoder from the pre-trained checkpoint:

Copied
from transformers import SpeechT5Processor, SpeechT5ForTextToSpeech, SpeechT5HifiGan

processor = SpeechT5Processor.from_pretrained("microsoft/speecht5_tts")

model = SpeechT5ForTextToSpeech.from_pretrained("microsoft/speecht5_tts")
vocoder = SpeechT5HifiGan.from_pretrained("microsoft/speecht5_hifigan")
Here we're using SpeechT5 checkpoint trained specifically for English TTS. Should you wish to translate into a language other than English, either swap the checkpoint for a SpeechT5 TTS model fine-tuned on your language of choice, or use an MMS TTS checkpoint pre-trained in your target langauge.
As with the Whisper model, we’ll place the SpeechT5 model and vocoder on our GPU accelerator device if we have one:

Copied
model.to(device)
vocoder.to(device)
Great! Let’s load up the speaker embeddings:

Copied
embeddings_dataset = load_dataset("Matthijs/cmu-arctic-xvectors", split="validation")
speaker_embeddings = torch.tensor(embeddings_dataset[7306]["xvector"]).unsqueeze(0)
We can now write a function that takes a text prompt as input, and generates the corresponding speech. We’ll first pre-process the text input using the SpeechT5 processor, tokenizing the text to get our input ids. We’ll then pass the input ids and speaker embeddings to the SpeechT5 model, placing each on the accelerator device if available. Finally, we’ll return the generated speech, bringing it back to the CPU so that we can play it back in our ipynb notebook:

Copied
def synthesise(text):
    inputs = processor(text=text, return_tensors="pt")
    speech = model.generate_speech(
        inputs["input_ids"].to(device), speaker_embeddings.to(device), vocoder=vocoder
    )
    return speech.cpu()
Let’s check it works with a dummy text input:

Copied
speech = synthesise("Hey there! This is a test!")

Audio(speech, rate=16000)
Sounds good! Now for the exciting part - piecing it all together.

Creating a STST demo
Before we create a Gradio demo to showcase our STST system, let’s first do a quick sanity check to make sure we can concatenate the two models, putting an audio sample in and getting an audio sample out. We’ll do this by concatenating the two functions we defined in the previous two sub-sections, such that we input the source audio and retrieve the translated text, then synthesise the translated text to get the translated speech. Finally, we’ll convert the synthesised speech to an int16 array, which is the output audio file format expected by Gradio. To do this, we first have to normalise the audio array by the dynamic range of the target dtype (int16), and then convert from the default NumPy dtype (float64) to the target dtype (int16):

Copied
import numpy as np

target_dtype = np.int16
max_range = np.iinfo(target_dtype).max


def speech_to_speech_translation(audio):
    translated_text = translate(audio)
    synthesised_speech = synthesise(translated_text)
    synthesised_speech = (synthesised_speech.numpy() * max_range).astype(np.int16)
    return 16000, synthesised_speech
Let’s check this concatenated function gives the expected result:

Copied
sampling_rate, synthesised_speech = speech_to_speech_translation(sample["audio"])

Audio(synthesised_speech, rate=sampling_rate)
Perfect! Now we’ll wrap this up into a nice Gradio demo so that we can record our source speech using a microphone input or file input and playback the system’s prediction:

Copied
import gradio as gr

demo = gr.Blocks()

mic_translate = gr.Interface(
    fn=speech_to_speech_translation,
    inputs=gr.Audio(source="microphone", type="filepath"),
    outputs=gr.Audio(label="Generated Speech", type="numpy"),
)

file_translate = gr.Interface(
    fn=speech_to_speech_translation,
    inputs=gr.Audio(source="upload", type="filepath"),
    outputs=gr.Audio(label="Generated Speech", type="numpy"),
)

with demo:
    gr.TabbedInterface([mic_translate, file_translate], ["Microphone", "Audio File"])

demo.launch(debug=True)
This will launch a Gradio demo similar to the one running on the Hugging Face Space:


You can duplicate this demo and adapt it to use a different Whisper checkpoint, a different TTS checkpoint, or relax the constraint of outputting English speech and follow the tips provide for translating into a langauge of your choice!

Going forwards
While the cascaded system is a compute and data efficient way of building a STST system, it suffers from the issues of error propagation and additive latency described above. Recent works have explored a direct approach to STST, one that does not predict an intermediate text output and instead maps directly from source speech to target speech. These systems are also capable of retaining the speaking characteristics of the source speaker in the target speech (such a prosody, pitch and intonation). If you’re interested in finding out more about these systems, check-out the resources listed in the section on supplemental reading.# Speech-to-speech-translation
