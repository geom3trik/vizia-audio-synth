![alt text](https://github.com/geom3trik/tuix_audio_synth/blob/main/screenshot.png?raw=true)

In this tutorial we'll create a very simple audio synthesizer application from scratch with a GUI using [Vizia](https://github.com/vizia/vizia).

(WARNING: Don't have your volume too loud when using headphones)

# Step 1 - Create a new rust project

Start by creating a new rust binary project:
```
cargo new audio_synth
```

In the generated `Cargo.toml` file inside the audio_synth directory, add the following dependencies:

```toml
[dependencies]
cpal = "0.13.5"
anyhow = "1.0.58"
vizia = {git = "https://github.com/geom3trik/vizia"}
crossbeam-channel = "0.5.5"
```

We'll be using cpal for the audio generation and crossbeam-channel for communicating between our main thread and the audio thread which CPAL creates for us.

# Step 2 - Create a simple vizia application

To start with we'll just create an empty window application using vizia with the following code in our `main.rs` file: 

```Rust
use vizia::prelude::*;
use cpal::traits::{DeviceTrait, HostTrait, StreamTrait};

fn main() {

    Application::new(|cx|{
        // UI will go here
    })
    .title("Audio Synth")
    .inner_size((200, 120))
    .run();
}
```

To save some time the things we'll need from vizia and cpal have also been included at the top of the file.

# Step 3 - Generating some sound

Before we populate our window with views, let's first write the code that will generate some sound.

First we'll start a new thread for the audio generation, get the default host and output device from cpal, and then call a `run` function that will generate the audio, which we'll write next. Add the following code to the beginning of our main function:

```Rust
std::thread::spawn(move || {

    let host = cpal::default_host();

    let device = host
        .default_output_device()
        .expect("failed to find a default output device");

    let config = device.default_output_config().unwrap();

    match config.sample_format() {
        cpal::SampleFormat::F32 => {
            run::<f32>(&device, &config.into()).unwrap();
        }

        cpal::SampleFormat::I16 => {
            run::<i16>(&device, &config.into()).unwrap();
        }
            
        cpal::SampleFormat::U16 => {
            run::<u16>(&device, &config.into()).unwrap();
        }    
    }
});
```

Because we don't know what sample format the default output config will give to us, we need to match on the sample format and make our `run` function generic over the sample type.

Now we'll write that `run` function which will build an output stream and play the audio. Add this code below our main function in the `main.rs` file:

```Rust
fn run<T>(device: &cpal::Device, config: &cpal::StreamConfig) -> Result<(), anyhow::Error>
where
    T: cpal::Sample,
{

    // Get the sample rate and channels number from the config
    let sample_rate = config.sample_rate.0 as f32;
    let channels = config.channels as usize;

    let err_fn = |err| eprintln!("an error occurred on stream: {}", err);
    
    // Define some variables we need for a simple oscillator
    let mut phi = 0.0f32;
    let mut frequency = 440.0;
    let mut amplitude = 0.1;
    let mut note = 1.0;
    
    // Build an output stream
    let stream = device.build_output_stream(
        config,
        move |data: &mut [T], _: &cpal::OutputCallbackInfo| {
            for frame in data.chunks_mut(channels) {
                
                // This creates a 'phase clock' which varies between 0.0 and 1.0 with a rate of frequency
                phi = (phi + (frequency / sample_rate)).fract();

                let make_noise = |phi: f32| -> f32 {amplitude * (2.0 * 3.141592 * phi).sin()};
                
                // Convert the make_noise output into a sample
                let value: T = cpal::Sample::from::<f32>(&make_noise(phi));
                
                for sample in frame.iter_mut() {
                    *sample = value;
                }
            }
        },
        err_fn,
    )?;

    // Play the stream
    stream.play()?;
    
    // Park the thread so our noise plays continuously until the app is closed
    std::thread::park();

    Ok(())
}
```

Inside our run function are values for note, which is either 0.0 for off or 1.0 for on, amplitude, which varies between 0.0 and 1.0, and frequency, which we've set initially to 440.0 Hz.

If we run our app now with `cargo run`, a window should appear and you should hear a tone played continuously until we close the window.

# Step 4 - Setting up communication

To be able to change the note, amplitude, and frequency of the tone from the UI thread, we need to set up some inter-thread communication.

First we'll define the messages that can be sent to the audio thread with an enum. Add the following above the main function:
```Rust
#[derive(Clone, Copy, Debug, PartialEq)]
pub enum Message {
    Note(f32), 
    Frequency(f32),
    Amplitude(f32),
}
```

Next, we'll create a crossbeam channel which we can use to send these messages from a sender to a receiver. Add the following to the top of the main function before creating the application:

```Rust
let (command_sender, command_receiver) = crossbeam_channel::bounded(1024);
```

# Step 5- Creating the UI model

Before we can add some views to control the synth parameters, we need some data which the views will bind to. Vizia is a reactive framework, which means that the UI will update in response to changes in data.

Define a new struct and it's implementation like so:

```Rust
#[derive(Lens)]
struct AppData {
    command_sender: crossbeam_channel::Sender<Message>,
    amplitude: f32,
    frequency: f32,
}

impl AppData {
    pub fn new(command_sender: crossbeam_channel::Sender<Message>) -> Self {
        Self {
            command_sender,
            amplitude: 0.1,
            frequency: 0.0,
        }
    }
}
```


The `AppData` contains a crossbeam_channel sender which we'll use to send messages to the audio thread. Note also the `#[derive(Lens)]`. This is used by Vizia to bind views to pieces of data from our model.

Next we'll implement the `Model` trait for the `AppData`. This trait provides an `event` function which we can use to update the model data in response to events sent from views in our UI. For this we'll also need an enum which represents these UI events. Add the following above the main function:

```rust
pub enum AppEvent {
    SetAmplitude(f32),
    SetFrequency(f32),
}

impl Model for AppData {
    fn event(&mut self, cx: &mut Context, event: &mut Event) {
        // Respond to app events
        event.map(|app_event, _| match app_event {
            AppEvent::SetAmplitude(amp) => {
                self.amplitude = *amp;
                self.command_sender.send(Message::Amplitude(self.amplitude)).unwrap();
            }

            AppEvent::SetFrequency(freq) => {
                self.frequency = *freq;
                self.command_sender.send(Message::Frequency(self.frequency)).unwrap();
            }
        });

        // Respond to window events
        event.map(|window_event, _| match window_event {
            WindowEvent::KeyDown(code, _) if *code == Code::KeyZ => {
                self.command_sender.send(Message::Note(1.0)).unwrap();
            }

            WindowEvent::KeyUp(code, _) if *code == Code::KeyZ => {
                self.command_sender.send(Message::Note(0.0)).unwrap();
            }

            _=> {}
        })
    }
}
```

Within the `event` function of the `Model` trait we handle two different kinds of event. In the first case we handle our application events, setting the internal model state and also sending a message to the audio thread. In the second case we handle window events.

Here we use the `KeyDown` and `KeyUp` variants from `WindowEvent` and check if the input key is the Z key. Then we send a `Note` message using the command sender with `Note(1.0)` when the Z key is pressed and `Note(0.0)` when it is released.

To use this model we'll need to add it to our application and build it. Change the code inside of `Application::new(...)` so it looks like this:

```Rust
Application::new(move |cx|{
    AppData::new(command_sender.clone()).build(cx);
})
.title("Audio Synth")
.inner_size((200, 120))
.run();

```

If we run our app again now with `cargo run` nothing will have changed. This is because although we are sending messages when the `Z` key is pressed, our audio thread isn't set up to receive them yet.

# Step 6 - Reacting to messages

Now that messages are being sent by our model, we need to modify the code in our `run` function to receive these events and change our oscillator note value. Modify the `run` function to look like the following:

```Rust
fn run<T>(device: &cpal::Device, config: &cpal::StreamConfig, command_receiver: crossbeam_channel::Receiver<Message>) -> Result<(), anyhow::Error>
where
    T: cpal::Sample,
{

    // Get the sample rate and channels number from the config
    let sample_rate = config.sample_rate.0 as f32;
    let channels = config.channels as usize;

    let err_fn = |err| eprintln!("an error occurred on stream: {}", err);
    
    // Define some variables we need for a simple oscillator
    let mut phi = 0.0f32;
    let mut frequency = 440.0;
    let mut amplitude = 1.0;
    let mut note = 0.0;
    
    // Build an output stream
    let stream = device.build_output_stream(
        config,
        move |data: &mut [T], _: &cpal::OutputCallbackInfo| {
            for frame in data.chunks_mut(channels) {

                while let Ok(command) = command_receiver.try_recv() {
                    // println!("Received Message: {:?}", command);
                     match command {
                        Message::Note(val) => {
                            note = val;
                        }
                       
                        Message::Amplitude(val) => {
                            amplitude = val;
                        }
 
                        Message::Frequency(val) => {
                            frequency = (val * (2000.0 - 440.0)) + 440.0;
                        }
 
                         _=> {}
                     }
                     
                 }
                
                // This creates a 'phase clock' which varies between 0.0 and 1.0 with a rate of frequency
                phi = (phi + (frequency / sample_rate)).fract();

                let make_noise = |phi: f32| -> f32 {note * amplitude * (2.0 * 3.141592 * phi).sin()};
                
                // Convert the make_noise output into a sample
                let value: T = cpal::Sample::from::<f32>(&make_noise(phi));
                
                for sample in frame.iter_mut() {
                    *sample = value;
                }

            }

        },
        err_fn,
    )?;

    // Play the stream
    stream.play()?;
    
    // Park the thread so out noise plays continuously until the app is closed
    std::thread::park();

    Ok(())
}
```

Note that the value received from the `Frequency` message is a normalized value, which is why we need to convert the frequency in the `run()` function with the following code:

```rs
frequency = (val * (2000.0 - 440.0)) + 440.0;
```

The run function now takes an additional crossbeam_channel receiver parameter, and notice that we've also now multiplied the sine function inside of the `make_noise` closure by the note value. Make sure to pass the `command_receiver` to the run function call inside our main function like so:

```Rust
...
thread::spawn(move || {

    let host = cpal::default_host();
    
    let device = host
        .default_output_device()
        .expect("failed to find a default output device");
    
    let config = device.default_output_config().unwrap();
    
    match config.sample_format() {
        cpal::SampleFormat::F32 => {
            run::<f32>(&device, &config.into(), command_receiver.clone()).unwrap();
        }
    
        cpal::SampleFormat::I16 => {
            run::<i16>(&device, &config.into(), command_receiver.clone()).unwrap();
        }
                
        cpal::SampleFormat::U16 => {
            run::<u16>(&device, &config.into(), command_receiver.clone()).unwrap();
        }    
    }
});
...
```

If we run this now the tone should play when we hit the Z key.

# Step 7 - Adding control knobs

Time to add some controls for the amplitude and frequency of our simple oscillator. Inside the application closure, add the following code:

```rust
Application::new(move |cx|{
    AppData::new(command_sender.clone()).build(cx);

    // A row of knobs
    HStack::new(cx, |cx|{

        Knob::new(cx, 0.5, AppData::amplitude, false)
            .on_changing(|cx, val| cx.emit(AppEvent::SetAmplitude(val)));

        
        Knob::new(cx, 0.0, AppData::frequency, false)
            .on_changing(|cx, val| cx.emit(AppEvent::SetFrequency(val)));
    })
    .class("content");
})
.title("Audio Synth")
.inner_size((200, 120))
.run();

```
Let's break this down. The `HStack` is a container view which arranges its children into a row. To add children to the `HStack`, we simply declare them inside the closure. The `Knob` view takes three parameters after `cx`, an initial normalized value, a lens to the model data, and a boolean which determines whether the knob starts from the beginning or the center.

The `#[derive(Lens)]` on our `AppData` allows us to pass a lens to a piece of our model with the convenient syntax `AppData::amplitude`. Now, if another view or event were to change the value of `amplitude` in our `AppData`, the knob would update to show the changed value.

To update the values in the model the knobs have an `on_changing` callback which emit the appropriate `AppEvent`. This event propagates up the UI tree to the model where it is handled.

Before we can run this and see our controls, we need to style them. Create a new file called `theme.css` in the src directory of this project. Then, add the following lines to this css file:

```CSS
.content {
    background-color: #262a2d;
    child-space: 1s;
    col-between: 1s;
}

knob {
    width: 76px;
    height: 76px;
    background-color: #262a2d;
    border-radius: 50%;
}

knob .track {
    background-color: #ffb74d;
}
```

Then add this line somewhere at the top of the main.rs file:

```Rust
static THEME: &'static str = include_str!("theme.css");
```

And then add this line inside the `Application::new()` closure, before the call to `AppData::new()`:

```Rust
cx.add_theme(THEME);
```

And that's it! If we run our app now and press the Z key to play the tone we can now change the amplitude and frequency of the tone using the two knobs, even while the tone is playing. Note that we don't have any smoothing in place so sudden changes in amplitude or frequency may cause a crackling sound.