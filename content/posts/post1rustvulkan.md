---
categories: ["Rust", "Vulkan"]
title: "Learning Vulkan with Rust, Part 1: Base Code"
date: 2022-07-30T21:05:46-07:00
draft: true
url: ""
---

I've decided to go and work through and learn Vulkan in Rust. There will surely be no issues with this and everything will go smoothly without any errors.

<!--more-->

I've decided to go and work through and learn Vulkan in Rust. I would usually work through something like this in C++, but more often than not I find setting up a project in C++ far more of a headache than it should be. Rust, at least, doesn't require interacting with CMake files, which is always a perk. Plus, this gives me an opportunity to more fully learn Rust.


## Getting Started

Starting with the standard [Vulkan tutorial](https://vulkan-tutorial.com), the first relevant code-bits are in [setting up the development environment](https://vulkan-tutorial.com/Development_environment). However, outside of installing the Vulkan SDK itself, the setup for the equivalent in Rust is far simpler. No need for Visual Studio; a standard Rust environment will do. As for the relevant crates, those will be tackled as they come up during the tutorial.

The [base code](https://vulkan-tutorial.com/Drawing_a_triangle/Setup/Base_code) section is where we start encountering actual code. The first thing we encounter is a standard C++ class with a public `run()` method alongside several private methods that execute in sequence. Since we're just going to be modifying this object rather than performing specific implementations, we can keep things simple and just replicate the structure:

```rust
pub struct HelloTriangleApplication {

}

impl HelloTriangleApplication {
    fn init_vulkan(&self) {

    }
    fn main_loop(&self) {

    }
    fn cleanup(&self) {

    }
    pub fn run(&self) {
        self.init_vulkan();
        self.main_loop();
        self.cleanup();
    }
}
```

For sanity we'll keep main in its own file:

```rust
mod basecode;

use basecode::HelloTriangleApplication;

fn main() {
    let app = HelloTriangleApplication {};
    app.run();
}
```

There's an immediate concern, however. While the original code handles errors with exceptions, here we have no error-handling mechanism at all. As much fun as it sounds to just panic if anything goes wrong during the drawing loop, that'll no doubt cause problems down the line, so best to actually implement some amount of error-handling:

```rust
use std::error::Error;

pub struct HelloTriangleApplication {

}

impl HelloTriangleApplication {
    fn init_vulkan(&self) -> Result<(), Box<dyn Error>> {
        Ok(())
    }
    fn main_loop(&self) -> Result<(), Box<dyn Error>>  {
        Ok(())
    }
    fn cleanup(&self) -> Result<(), Box<dyn Error>>  {
        Ok(())
    }
    pub fn run(&self) -> Result<(), Box<dyn Error>>  {
        self.init_vulkan()?;
        self.main_loop()?;
        self.cleanup()?;
        Ok(())
    }
}
```

That handles the error handling (though we might refine that error return later), but we still don't actually import any sort of Vulkan API here. While there are a lot of Vulkan crates for Rust, the one I will use for this is `ash`. Why `ash`? Well, it more-or-less does the least amount of work for us, which is perfect for learning Vulkan. We'll just stick a `use ash::{vk, Entry};` at the top of the `basecode.rs` file; the first module is for the actual API, and the second for initializing everything. Just as with the tutorial, resource management is on us. Or at least, the most that it can be while using Rust.

Now, working our way through the tutorial, window management becomes a concern. This is no different in Rust, though our solution will be a little different here. While `glfw` does have bindings for Rust in the form of `glfw-rs`, for this we'll use `winit`. It's standardized enough and should for the most part mirror what we want enough that the differences should be negligible. Similar to the tutorial, we start by creating a function for handling window creation:

```rust
impl HelloTriangleApplication {
    fn init_window(&self) -> Result<(), Box<dyn Error>> {
        Ok(())
    }
    // ...
    
    pub fn run(&self) -> Result<(), Box<dyn Error>>  {
        self.init_window()?;
        self.init_vulkan()?;
        self.main_loop()?;
        self.cleanup()?;
        Ok(())
    }
}
```

Then, we populate the whole thing:

```rust
use std::error::Error;
use ash::{vk, Entry};
use winit::{
    event::{Event, WindowEvent},
    event_loop::{ControlFlow, EventLoop},
    window::WindowBuilder,
    dpi::PhysicalSize,
};

const width : u32 = 800;
const height : u32 = 600;

pub struct HelloTriangleApplication {
    window : winit::window::Window,
    event_loop : winit::event_loop::EventLoop<()>,
}

impl HelloTriangleApplication {
    fn init_window(&self) -> Result<(), Box<dyn Error>> {
        self.event_loop = EventLoop::new();
        self.window = WindowBuilder::new()
            .with_resizable(false)
            .with_inner_size(PhysicalSize::new(width, height))
            .build(&self.event_loop)?;
        Ok(())
    }
    fn init_vulkan(&self) -> Result<(), Box<dyn Error>> {
        Ok(())
    }
    fn main_loop(&self) -> Result<(), Box<dyn Error>>  {
        self.event_loop.run(|event, _, control_flow| {
            *control_flow = ControlFlow::Wait;
    
            match event {
                Event::WindowEvent {
                    event: WindowEvent::CloseRequested,
                    window_id,
                } if window_id == self.window.id() => *control_flow = ControlFlow::Exit,
                _ => (),
            }
        });
    }
    fn cleanup(&self) -> Result<(), Box<dyn Error>>  {
        Ok(())
    }
    pub fn run(&self) -> Result<(), Box<dyn Error>>  {
        self.init_window()?;
        self.init_vulkan()?;
        self.main_loop()?;
        self.cleanup()?;
        Ok(())
    }
}
```

Notably, we do not bother adding window closing to `cleanup()` since `winit` handles this for us. Well, everything seems to be working just fine so far! that's rather surpr—

```
error: cannot construct `HelloTriangleApplication` with struct literal syntax due to inaccessible fields
 --> src\main.rs:6:15
  |
6 |     let app = HelloTriangleApplication {};
  |               ^^^^^^^^^^^^^^^^^^^^^^^^
```

Well. It would seem our naïve approach has run aground. We need to reconfigure our window initialization into a `::new()` function so that everything is initialized properly.

```rust
    pub fn new() -> Self {
        let event_loop = EventLoop::new();
        let window = WindowBuilder::new()
            .with_resizable(false)
            .with_inner_size(PhysicalSize::new(WIDTH, HEIGHT))
            .build(&event_loop).unwrap();
        Self {
            event_loop : event_loop,
            window : window,
        }
    }
```

Straightforward enough. This rather removes needing the `init_window()` function, so we drop that. Everything now should be just fine. Just f—

```
error[E0507]: cannot move out of `self.event_loop` which is behind a shared reference
  --> src\basecode.rs:34:9
   |
34 |         self.event_loop.run(|event, _, control_flow| {
   |         ^^^^^^^^^^^^^^^ move occurs because `self.event_loop` has type `EventLoop<()>`, which does not implement the `Copy` trait

error[E0521]: borrowed data escapes outside of associated function
  --> src\basecode.rs:34:9
   |
33 |       fn main_loop(&self) -> Result<(), Box<dyn Error>>  {
   |                    -----
   |                    |
   |                    `self` is a reference that is only valid in the associated function body
   |                    let's call the lifetime of this reference `'1`
34 | /         self.event_loop.run(|event, _, control_flow| {
35 | |             *control_flow = ControlFlow::Wait;
36 | |
37 | |             match event {
...  |
43 | |             }
44 | |         });
   | |          ^
   | |          |
   | |__________`self` escapes the associated function body here
   |            argument requires that `'1` must outlive `'static`
```

Uh oh. So, lots of problems here, namely that our main loop is not running on the main thread and passes off our reference to self to this other thread. This ruins the lifetime semantics, so Rust disagrees with the decision. The solution, then, is to scrap all of this fiddling-with-other-function stuff completely and just keep things simple:

```rust
use std::error::Error;
use ash::{vk, Entry};
use winit::{
    event::{Event, WindowEvent},
    event_loop::{ControlFlow, EventLoop},
    window::WindowBuilder,
    dpi::PhysicalSize,
};

const WIDTH : u32 = 800;
const HEIGHT : u32 = 600;

pub struct HelloTriangleApplication {
    event_loop : winit::event_loop::EventLoop<()>,
    window : winit::window::Window,
}

impl HelloTriangleApplication {
    pub fn new() -> Self {
        let event_loop = EventLoop::new();
        let window = WindowBuilder::new()
            .with_resizable(false)
            .with_inner_size(PhysicalSize::new(WIDTH, HEIGHT))
            .build(&event_loop).unwrap();
        Self {
            event_loop : event_loop,
            window : window,
        }
    }
    pub fn run(self) -> Result<(), Box<dyn Error>>  {
        self.event_loop.run(move |event, _, control_flow| {
            *control_flow = ControlFlow::Wait;
    
            match event {
                Event::WindowEvent {
                    event: WindowEvent::CloseRequested,
                    window_id,
                } if window_id == self.window.id() => *control_flow = ControlFlow::Exit,
                _ => (),
            }
        });
    }
}
```

Now, all we have is a `new()` to create the app and `run()` to execute. Cleanup is implicit (at least so far), and any initialization happens when we create the object in the first place. Everything builds fine now! All the fires are out! There will surely be no more fires the moment we start touching Vulkan stuff!