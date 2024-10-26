<h1 style='text-align: center; margin-bottom: 1rem'> Gradio WebRTC ⚡️ </h1>

<div style="display: flex; flex-direction: row; justify-content: center">
<img style="display: block; padding-right: 5px; height: 20px;" alt="Static Badge" src="https://img.shields.io/pypi/v/gradio_webrtc"> 
<a href="https://github.com/freddyaboulton/gradio-webrtc" target="_blank"><img alt="Static Badge" src="https://img.shields.io/badge/github-white?logo=github&logoColor=black"></a>
</div>

<h3 style='text-align: center'>
Stream video and audio in real time with Gradio using WebRTC. 
</h3>

## Installation

```bash
pip install gradio_webrtc
```

to use built-in pause detection (see [conversational ai](#conversational-ai)), install the `vad` extra:

```bash
pip install gradio_webrtc[vad]
```

## Examples:
1. [Object Detection from Webcam with YOLOv10](https://huggingface.co/spaces/freddyaboulton/webrtc-yolov10n) 📷
2. [Streaming Object Detection from Video with RT-DETR](https://huggingface.co/spaces/freddyaboulton/rt-detr-object-detection-webrtc) 🎥
3. [Text-to-Speech](https://huggingface.co/spaces/freddyaboulton/parler-tts-streaming-webrtc) 🗣️
4. [Conversational AI](https://huggingface.co/spaces/freddyaboulton/omni-mini-webrtc) 🤖🗣️

## Usage

The WebRTC component supports the following three use cases:
1. [Streaming video from the user webcam to the server and back](#h-streaming-video-from-the-user-webcam-to-the-server-and-back)
2. [Streaming Video from the server to the client](#h-streaming-video-from-the-server-to-the-client)
3. [Streaming Audio from the server to the client](#h-streaming-audio-from-the-server-to-the-client)
4. [Streaming Audio from the client to the server and back (conversational AI)](#h-conversational-ai)


## Streaming Video from the User Webcam to the Server and Back

```python
import gradio as gr
from gradio_webrtc import WebRTC


def detection(image, conf_threshold=0.3):
    ... your detection code here ...


with gr.Blocks() as demo:
    image = WebRTC(label="Stream", mode="send-receive", modality="video")
    conf_threshold = gr.Slider(
        label="Confidence Threshold",
        minimum=0.0,
        maximum=1.0,
        step=0.05,
        value=0.30,
    )
    image.stream(
        fn=detection,
        inputs=[image, conf_threshold],
        outputs=[image], time_limit=10
    )

if __name__ == "__main__":
    demo.launch()

```
* Set the `mode` parameter to `send-receive` and `modality` to "video".
* The `stream` event's `fn` parameter is a function that receives the next frame from the webcam 
as a **numpy array** and returns the processed frame also as a **numpy array**.
* Numpy arrays are in (height, width, 3) format where the color channels are in RGB format.
* The `inputs` parameter should be a list where the first element is the WebRTC component. The only output allowed is the WebRTC component.
* The `time_limit` parameter is the maximum time in seconds the video stream will run. If the time limit is reached, the video stream will stop.

## Streaming Video from the server to the client

```python
import gradio as gr
from gradio_webrtc import WebRTC
import cv2

def generation():
    url = "https://download.tsi.telecom-paristech.fr/gpac/dataset/dash/uhd/mux_sources/hevcds_720p30_2M.mp4"
    cap = cv2.VideoCapture(url)
    iterating = True
    while iterating:
        iterating, frame = cap.read()
        yield frame

with gr.Blocks() as demo:
    output_video = WebRTC(label="Video Stream", mode="receive", modality="video")
    button = gr.Button("Start", variant="primary")
    output_video.stream(
        fn=generation, inputs=None, outputs=[output_video],
        trigger=button.click
    )

if __name__ == "__main__":
    demo.launch()
```

* Set the "mode" parameter to "receive" and "modality" to "video".
* The `stream` event's `fn` parameter is a generator function that yields the next frame from the video as a **numpy array**.
* The only output allowed is the WebRTC component.
* The `trigger` parameter the gradio event that will trigger the webrtc connection. In this case, the button click event.

## Streaming Audio from the Server to the Client

```python
import gradio as gr
from pydub import AudioSegment

def generation(num_steps):
    for _ in range(num_steps):
        segment = AudioSegment.from_file("/Users/freddy/sources/gradio/demo/audio_debugger/cantina.wav")
        yield (segment.frame_rate, np.array(segment.get_array_of_samples()).reshape(1, -1))

with gr.Blocks() as demo:
    audio = WebRTC(label="Stream", mode="receive", modality="audio")
    num_steps = gr.Slider(
        label="Number of Steps",
        minimum=1,
        maximum=10,
        step=1,
        value=5,
    )
    button = gr.Button("Generate")

    audio.stream(
        fn=generation, inputs=[num_steps], outputs=[audio],
        trigger=button.click
    )
```

* Set the "mode" parameter to "receive" and "modality" to "audio".
* The `stream` event's `fn` parameter is a generator function that yields the next audio segment as a tuple of (frame_rate, audio_samples).
* The numpy array should be of shape (1, num_samples).
* The `outputs` parameter should be a list with the WebRTC component as the only element.

## Conversational AI

```python
import gradio as gr
import numpy as np
from gradio_webrtc import WebRTC, StreamHandler
from queue import Queue
import time


class EchoHandler(StreamHandler):
    def __init__(self) -> None:
        super().__init__()
        self.queue = Queue()

    def receive(self, frame: tuple[int, np.ndarray] | np.ndarray) -> None:
        self.queue.put(frame)

    def emit(self) -> None:
        return self.queue.get()


with gr.Blocks() as demo:
    with gr.Column():
        with gr.Group():
            audio = WebRTC(
                label="Stream",
                rtc_configuration=None,
                mode="send-receive",
                modality="audio",
            )

        audio.stream(fn=EchoHandler(), inputs=[audio], outputs=[audio], time_limit=15)


if __name__ == "__main__":
    demo.launch()
```

* Instead of passing a function to the `stream` event's `fn` parameter, pass a `StreamHandler` implementation. The `StreamHandler` above simply echoes the audio back to the client.
* The `StreamHandler` class has two methods: `receive` and `emit`. The `receive` method is called when a new frame is received from the client, and the `emit` method returns the next frame to send to the client.
* An audio frame is represented as a tuple of (frame_rate, audio_samples) where `audio_samples` is a numpy array of shape (num_channels, num_samples).
* You can also specify the audio layout ("mono" or "stereo") in the emit method by retuning it as the third element of the tuple. If not specified, the default is "mono".
* The `time_limit` parameter is the maximum time in seconds the conversation will run. If the time limit is reached, the audio stream will stop.
* The `emit` method SHOULD NOT block. If a frame is not ready to be sent, the method should return `None`.

An easy way to get started with Conversational AI is to use the `ReplyOnPause` stream handler. This will automatically run your function when the speaker has stopped speaking. In order to use `ReplyOnPause`, the `[vad]` extra dependencies must be installed.

```python
import gradio as gr
from gradio_webrtc import WebRTC, ReplyOnPause

def response(audio: tuple[int, np.ndarray]):
    """This function must yield audio frames"""
    ...
    for numpy_array in generated_audio:
        yield (sampling_rate, numpy_array, "mono")


with gr.Blocks() as demo:
    gr.HTML(
    """
    <h1 style='text-align: center'>
    Chat (Powered by WebRTC ⚡️)
    </h1>
    """
    )
    with gr.Column():
        with gr.Group():
            audio = WebRTC(
                label="Stream",
                rtc_configuration=rtc_configuration,
                mode="send-receive",
                modality="audio",
            )
        audio.stream(fn=ReplyOnPause(response), inputs=[audio], outputs=[audio], time_limit=60)


demo.launch(ssr_mode=False)
```



## Deployment

When deploying in a cloud environment (like Hugging Face Spaces, EC2, etc), you need to set up a TURN server to relay the WebRTC traffic.
The easiest way to do this is to use a service like Twilio.

```python
from twilio.rest import Client
import os

account_sid = os.environ.get("TWILIO_ACCOUNT_SID")
auth_token = os.environ.get("TWILIO_AUTH_TOKEN")

client = Client(account_sid, auth_token)

token = client.tokens.create()

rtc_configuration = {
    "iceServers": token.ice_servers,
    "iceTransportPolicy": "relay",
}

with gr.Blocks() as demo:
    ...
    rtc = WebRTC(rtc_configuration=rtc_configuration, ...)
    ...
```