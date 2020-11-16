+++
date = "2020-11-15"
draft = false
title = "Writing an embedded display driver in Rust"
description = "Shows how the TSic temperature sensor code is ported from C to Rust"
tags = ["display", "rust",  "ssd1327", "embedded"]
+++

A year or so ago when I started my yourney into embedded development (to build a PID controller for my coffee machine) I bought a [1.5 inch RGB OLED display](https://www.waveshare.com/wiki/1.5inch_RGB_OLED_Module), which uses a SSD1351 controller. I got lucky, because a display driver for it [already existed](https://crates.io/crates/ssd1351).

Unfortunately, very recently it stopped working and I'm not sure why. Okay, maybe I tightened those screws a bit too much. Anyways, since I did not need a color display, I bought a very similar [1.5 inch OLED display](https://www.waveshare.com/wiki/1.5inch_OLED_Module). After plugging it in and not seeing anything rendered on the screen (and not touching my screwdriver), I realized that it uses a different controller: the SSD1327. This time I wasn't as lucky, because there happened to be no embedded rust driver out there yet.

When life gives you lemons, write a display driver for them - and then blog about it so everyone gets to learn maybe a thing or two.

![Display with Text](/img/display-clean.jpg)

The basic driver we are going to implement in this post works, but is neither flexible nor configurable. The full driver (which is still evolving) is goint to be a bit more complex and can be found [here](https://github.com/daschl/ssd1327).

# Preparations

Before we dive in, we first need to figure out how to talk to the display and what protocol to use. The [Datasheet](https://www.waveshare.com/w/upload/a/ac/SSD1327-datasheet.pdf) is very handy and I recommend to keep it open in a separate tab while working on the code.

The controller supports different modes of communication: 4-wire SPI, 3-wire SPI, I2C, Parallel 6800 and Parallel 8080. In this blog post we'll use 4-wire SPI which is the one the display comes preconfigured with. The bytes we send over SPI can either be commands (for example to turn the display on or off) or actual pixel data to render on the screen. The datasheet thankfully has all that info, but it can be a little tough to digest at first if you need to start from zero.

Thankfully, we don't need to start from zero. There are two very useful codebases out there which will help us hit the ground running:

 - Download the demo code from the [Waveshare Website](https://www.waveshare.com/wiki/1.5inch_OLED_Module), which contains C code we can look at, which is very helpful especially to understand what commands we need to send during startup.
 - A controller driver for the very similar [ssd1306](https://github.com/jamwaffles/ssd1306) exists, and it is very well written. We can borrow some concepts like the command structure.

# The Display Interface & Embedded Graphics Crates

We could directly talk to our hardware SPI interface, but that is not going to be very portable. Ideally we want to plug this display driver into the established embedded rust ecosystem and benefit from code reuse and portability. For our purpose, three crates are important:

 - [display-interface](https://github.com/therealprof/display-interface): traits that bridge between a bus driver (i.e. SPI) and a display driver (our ssd1327). By using this crate, we also open the door to support I2C later.
 - [embedded-graphics](https://github.com/embedded-graphics/embedded-graphics): once we can talk to the device, we also need to render pixels. This crate will make it super easy to render text, lines and more.
 - [embedded-hal](https://github.com/rust-embedded/embedded-hal): this is the main hardware abstraction layer crate and we'll use it to talk to specific hardware pins and use time delay functionality.

We are going to make use of all three in the next sections.

# Bootstrapping

Before we can render pixels on the screen, we need to initialize our display. The datasheet contains *a lot* of different commands, many of them are hardware oriented and it is pretty hard to figure out what we need. Thankfully, if we peek into the waveshare sample code, we can see a function like this:

```c
void OLED_Init(OLED_SCAN_DIR OLED_ScanDir)
{
  //Hardware reset
  OLED_Reset();

  //Set the initialization register
  OLED_InitReg();

  //Set the display scan and color transfer modes
  OLED_SetGramScanWay(OLED_ScanDir );
  Driver_Delay_ms(200);

  //Turn on the OLED display
  OLED_WriteReg(0xAF);
}
```

This gives us a first idea: we need to reset the display, then initialize it, sleep and then write `0xAF`. Searching for `AF` in the datasheet tells us that this command will turn the display on. Alright, let's start with reset:

```c
static void OLED_Reset(void)
{
  OLED_RST_1;
  Driver_Delay_ms(100);
  OLED_RST_0;
  Driver_Delay_ms(100);
  OLED_RST_1;
  Driver_Delay_ms(100);
}
```

The code sets the RST ("reset") pin from our display to high, then low and then high again, sleeping 100ms in between. So how would we translate that to rust?

```rs
use embedded_hal::blocking::delay::DelayMs;
use embedded_hal::digital::v2::OutputPin;

pub fn reset<RST, DELAY>(&mut self, rst: &mut RST, delay: &mut DELAY)
where
    RST: OutputPin,
    DELAY: DelayMs<u8>,
{
    rst.set_high().ok().unwrap();
    delay.delay_ms(100);

    rst.set_low().ok().unwrap();
    delay.delay_ms(100);

    rst.set_high().ok().unwrap();
    delay.delay_ms(100);
}
```

The `embedded-hal` crate defines two important traits: one for an `OutputPin` and one to `Delay` for a certain number of milliseconds. The actual implementation comes from your board HAL (hardware abstraction layer), one of which we'll see later. Also note that error handling is omitted here to simplify the code for this blog post.

Next up, we need to initialize the display. The C code is not very well documented, but we can see that a sequence of commands is sent to the display:

```c
static void OLED_InitReg(void)
{
  OLED_WriteReg(0xae);//--turn off oled panel

  OLED_WriteReg(0x15);    //set column address
  OLED_WriteReg(0x00);    //start column   0
  OLED_WriteReg(0x7f);    //end column   127

  *** many more commands here ***

  OLED_WriteReg(0xfd);
  OLED_WriteReg(0x12);

}
```

For our rust code we want to turn those commands into an actual `enum` so it is easier to understand. We also need to implement the equivalent of `OLED_WriteReg`, which will send those commands to the display. This is where the `display-interface` crate comes in. The `WriteOnlyDataCommand` trait it defines abstracts the actual low level communication for us and we just need to call the implementation of the trait.

The idea of the following code is shamelessly taken from the excellent `ssd1306` crate. Every command gets its own enum variant and we can `send` it to our display:

```rs
use display_interface::{DataFormat::U8, DisplayError, WriteOnlyDataCommand};

pub enum Command {
    /// Turn display off (0xAE)
    DisplayOff,
    /// Turn display on (0xAF)
    DisplayOn,
    /// Function Selection B (0xD5)
    CommandLock(u8),
}

impl Command {
    pub fn send<DI>(self, display: &mut DI) -> Result<(), DisplayError>
    where
        DI: WriteOnlyDataCommand,
    {
        let (data, len) = match self {
            Self::DisplayOn => ([0xAF, 0, 0], 1),
            Self::DisplayOff => ([0xAE, 0, 0], 1),
            Self::CommandLock(value) => ([0xFD, value, 0], 2),
        };
        display.send_commands(U8(&data[0..len]))
    }
}
```

For the actual driver we need more commands, but the approach is always the same. On `send` we match on the enum variant and turn it into a byte slice which is then sent over the connection through the `WriteOnlyDataCommand` trait.

*If you wonder why it specifies the length and data in such a weird way (as I did): if you try to return the array in the match arms rust will complain that the match arms have different array sizes. So the workaround by [jamwaffles](https://github.com/jamwaffles) is to always return a byte array with the same length, but then pass a slice of the correct size to the display. This makes the compiler happy and keeps the implementation short.*

Before we can glue it all together, we need to figure out how we can send the pixel information to the display.

# Rendering Pixels

Our display has 128 rows and 128 colums with a potential of 16 colors each. 16 colors are represented by 4 bits, which is why the internal graphic display data RAM (GDDRAM) contains a buffer of `128 * 128 * 4 = 65536` bits. This allows to store all information for our 16384 pixels. Since rust does not have a `u4` datatype, we'll use a single byte `u8` representation instead. This allows us to store the information of two pixels in one byte, so we need a buffer size of `65536 / 8 = 8192` bytes.

This is where `embedded-graphics` comes in. It defines a `DrawTarget` trait which we need to implement for our display. All we need to tell it is how to draw a pixel and what our display size is, and it will figure out the rest for us (i.e. drawing text or a shape):

```rs
use display_interface::DisplayError;
use embedded_graphics::{
    drawable::Pixel,
    pixelcolor::{Gray4, GrayColor},
    prelude::Size,
    DrawTarget,
};

impl<DI> DrawTarget<Gray4> for Ssd1327<DI> {
    type Error = DisplayError;

    fn draw_pixel(&mut self, pixel: Pixel<Gray4>) -> Result<(), Self::Error> {
        let Pixel(point, color) = pixel;

        let idx = (point.x / 2 + point.y * 64) as usize;
        if point.x % 2 == 0 {
            self.buffer[idx] = (color.luma() << 4) | self.buffer[idx];
        } else {
            self.buffer[idx] = (color.luma() & 0x0f) | self.buffer[idx];
        }

        Ok(())
    }

    fn size(&self) -> Size {
        Size::new(128, 128)
    }
}
```

The first thing to note is `Gray4` on the `DrawTarget`. Since this trait supports all kinds of displays, we need to tell it which pixel colors we support. `Gray4` indicates 4 bit grayscale, so exactly what we need. This is also highlights how we utilize rust's type system to our benefit. We will only accept pixel colors that our device actually supports, any attempts to pass in i.e. a red color will fail at compile time.

In the `draw_pixel` method, we extract the `x` and `y` coordinates of the pixel as well as its color through the `luma()` method. We first need to calculate the actual absolute position in our buffer based on the coordinates and then set the color on the element. Since we store 2 colors per byte the code needs to perform some bitshifting and either set the high or low 4 bits of the byte. This code has been taken from the waveshare C code and ported over to rust.

Since we only wrote the information into an in-memory buffer, we also need the capability to send it over the wire to the display. For this, we add a `flush` method which takes the full buffer and sends it:

```rs
pub fn flush(&mut self) -> Result<(), DisplayError> {
    self.display.send_data(U8(&self.buffer))
}
```

This method will always take the whole buffer and send it to the display, even if we only changed one pixel. There are plenty of optimization opportunities here which we won't cover in this post.

Ok, so what's the `buffer` exactly? Time to put it all together in a single struct:

```rs
use crate::command::Command;
use display_interface::{DataFormat::U8, DisplayError, WriteOnlyDataCommand};
use embedded_graphics::{
    drawable::Pixel,
    pixelcolor::{Gray4, GrayColor},
    prelude::Size,
    DrawTarget,
};
use embedded_hal::blocking::delay::DelayMs;
use embedded_hal::digital::v2::OutputPin;

pub const DISPLAY_WIDTH: u32 = 128;
pub const DISPLAY_HEIGHT: u32 = 128;

pub struct Ssd1327<DI> {
    display: DI,
    buffer: [u8; 8192],
}

impl<DI: WriteOnlyDataCommand> Ssd1327<DI> {
    pub fn new(display: DI) -> Self {
        Self {
            display,
            buffer: [0; 8192],
        }
    }

    pub fn reset<RST, DELAY>(&mut self, rst: &mut RST, delay: &mut DELAY)
    where
        RST: OutputPin,
        DELAY: DelayMs<u8>,
    {
        rst.set_high().ok().unwrap();
        delay.delay_ms(100);

        rst.set_low().ok().unwrap();
        delay.delay_ms(100);

        rst.set_high().ok().unwrap();
        delay.delay_ms(100);
    }

    pub fn init(&mut self) -> Result<(), DisplayError> {
        self.send_command(Command::DisplayOff)?;
        self.send_command(Command::ColumnAddress { start: 0, end: 127 })?;
        self.send_command(Command::RowAddress { start: 0, end: 127 })?;
        self.send_command(Command::Contrast(0x80))?;
        self.send_command(Command::SetRemap(0x51))?;
        self.send_command(Command::StartLine(0x00))?;
        self.send_command(Command::Offset(0x00))?;
        self.send_command(Command::DisplayModeNormal)?;
        self.send_command(Command::MuxRatio(0x7f))?;
        self.send_command(Command::PhaseLength(0xf1))?;
        self.send_command(Command::FrontClockDivider(0x00))?;
        self.send_command(Command::FunctionSelectionA(0x01))?;
        self.send_command(Command::SecondPreChargePeriod(0x0f))?;
        self.send_command(Command::ComVoltageLevel(0x0f))?;
        self.send_command(Command::PreChargeVoltage(0x08))?;
        self.send_command(Command::FunctionSelectionB(0x62))?;
        self.send_command(Command::CommandLock(0x12))?;
        self.send_command(Command::DisplayOn)?;

        Ok(())
    }

    fn send_command(&mut self, command: Command) -> Result<(), DisplayError> {
        command.send(&mut self.display)
    }

    pub fn flush(&mut self) -> Result<(), DisplayError> {
        self.display.send_data(U8(&self.buffer))
    }
}

impl<DI> DrawTarget<Gray4> for Ssd1327<DI> {
    type Error = DisplayError;

    fn draw_pixel(&mut self, pixel: Pixel<Gray4>) -> Result<(), Self::Error> {
        let Pixel(point, color) = pixel;

        let idx = (point.x / 2 + point.y * 64) as usize;
        if point.x % 2 == 0 {
            self.buffer[idx] = (color.luma() << 4) | self.buffer[idx];
        } else {
            self.buffer[idx] = (color.luma() & 0x0f) | self.buffer[idx];
        }

        Ok(())
    }

    fn size(&self) -> Size {
        Size::new(DISPLAY_WIDTH, DISPLAY_HEIGHT)
    }
}
```

The `buffer` is initialized as a `[u8; 8192]`. Note that we fill it with zeroes when the display is constructred, which will sets every pixel to black. Also you can see that we now send more commands to the display on init, but the way it works is exactly the same as described above.

# Real World Usage

In this example I'm using the [Adafruit Feather nRF52840 Express](https://www.adafruit.com/product/4062) board, together with the [nrf52840-hal](https://docs.rs/nrf52840-hal/0.11.0/nrf52840_hal/).

I'm connected to the board via the [Segger J-Link Mini](https://www.segger.com/products/debug-probes/j-link/models/j-link-edu-mini/) and use [probe-run](https://github.com/knurling-rs/probe-run) to flash the board.

If you do not own a J-Link device and don't want to buy one, the [nRF52840 DK](https://www.nordicsemi.com/Software-and-Tools/Development-Kits/nRF52840-DK) has one already integrated. Check out [my blog post](https://nitschinger.at/Getting-Started-with-the-nRF52840-in-Rust/) for an introduction to it.

First, we need to grab our device peripherals:

```rs
let peripherals = Peripherals::take().unwrap();
let port0 = p0::Parts::new(peripherals.P0);
```

To get a `DelayMs` trait implementation, we can use a `Timer`:

```rs
let mut timer = Timer::new(peripherals.TIMER0);
```

Next, we wire up all our `SPI` pins. The specific pins for your device of course might be different.

```rs
let mut rst_pin = port0.p0_28.into_push_pull_output(Level::Low).degrade();
let dc_pin = port0.p0_02.into_push_pull_output(Level::Low).degrade();
let cs_pin = port0.p0_03.into_push_pull_output(Level::Low).degrade();

let spi_pins = Pins {
    sck: port0.p0_14.into_push_pull_output(Level::Low).degrade(),
    mosi: Some(port0.p0_13.into_push_pull_output(Level::Low).degrade()),
    miso: None,
};
let spi = Spim::new(peripherals.SPIM0, spi_pins, Frequency::M2, MODE_0, 0);
```

Since we are using the `display-interface` crate, we need to wrap our SPI implementation in its wrapper struct:

```rs
let spii = SPIInterface::new(spi, dc_pin, cs_pin);
```

Finally, we can plug it into our display, reset and initialize it:

```rs
let mut display = Ssd1327::new(spii);

display.reset(&mut rst_pin, &mut timer);
display.init(&mut timer).unwrap();
```

Time to render something! Also we should not forget to `flush` at the end. In this example we render one white pixel in every corner of the display as well as some text:

```rs
Pixel(Point::new(0, 0), Gray4::WHITE)
    .draw(&mut display)
    .unwrap();
Pixel(Point::new(0, 127), Gray4::WHITE)
    .draw(&mut display)
    .unwrap();
Pixel(Point::new(127, 0), Gray4::WHITE)
    .draw(&mut display)
    .unwrap();
Pixel(Point::new(127, 127), Gray4::WHITE)
    .draw(&mut display)
    .unwrap();

let style = TextStyleBuilder::new(Font6x8)
    .text_color(Gray4::WHITE)
    .background_color(Gray4::BLACK)
    .build();

Text::new("Hello Rust!", Point::new(10, 10))
    .into_styled(style)
    .draw(&mut display)
    .unwrap();

display.flush().unwrap();
```

And there we go! Here is what gets rendered on the display:

![Display wired up](/img/display-wired.png)

At this point we have a working driver for our display. It doesn't have many bells and whistles, but that can all be added later now that we understand how it works.

I hope you enjoyed this post as much as I did writing it! You can find the current version of the driver [here](https://github.com/daschl/ssd1327).