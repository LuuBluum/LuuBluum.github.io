---
categories: ["Rust", "Vulkan"]
title: "Learning Vulkan with Rust, Part 2: Creating a Vulkan Instance with Validation Layers"
date: 2022-07-31T14:45:18-07:00
draft: false
url: ""
---

Having created the basic code structure for setting up a Vulkan project in Rust back in [part 1](https://luubluum.github.io/posts/post1rustvulkan/), we can now actually deal with creating a [Vulkan instance](https://vulkan-tutorial.com/en/Drawing_a_triangle/Setup/Instance) and setting up [validation layers](https://vulkan-tutorial.com/Drawing_a_triangle/Setup/Validation_layers). All of the code can be found at [this repo](https://github.com/LuuBluum/Learning-Vulkan-with-Rust).

<!--more-->

Of course, since we're using Ash for this, the code won't necessarily be 1:1 with the equivalent in C++. Last time, we depopulated our app struct to just have a `new()` function and a `run()` function. For simplicity's sake we can go and restore the `init_window()` function, just using it during `new()` instead:

```rust
    fn init_window() -> Result<(winit::event_loop::EventLoop<()>, winit::window::Window), winit::error::OsError> {
        let event_loop = EventLoop::new();
        let window = WindowBuilder::new()
            .with_resizable(false)
            .with_inner_size(PhysicalSize::new(WIDTH, HEIGHT))
            .build(&event_loop)?;
        Ok((event_loop, window))
    }
    pub fn new() -> Self {
        let (event_loop, window) = HelloTriangleApplication::init_window().unwrap();
        Self {
            event_loop : event_loop,
            window : window,
        }
    }
```

Now if we want to expand on our window functionality, it should be easier to do so. Plus, it gives us back a mechanism for handling errors. That leaves us with setting up Vulkan. We'll follow the same approach, with an `init_vulkan()` function that returns the desired instance:

```rust

    fn init_vulkan() -> prelude::VkResult<ash::Instance> {
        let entry = Entry::linked();
        let app_info = vk::ApplicationInfo {
            api_version: vk::make_api_version(0, 1, 0, 0),
            ..Default::default()
        };
        let create_info = vk::InstanceCreateInfo {
            p_application_info: &app_info,
            ..Default::default()
        };
        unsafe { entry.create_instance(&create_info, None) }
    }
```

Notably here we needed to break out `unsafe`, since everything handled in Ash is unsafe. Additionally, we didn't need to deal with the window at all for setting up Vulkan. As for cleanup, for this we will need to implement Drop. Certainly better than explicitly needing to invoke a cleanup function which we can't otherwise explicitly invoke anyway. So, we can quickly implement that:

```rust
impl Drop for HelloTriangleApplication {
    fn drop(&mut self)
    {
        unsafe {
            self.instance.destroy_instance(None);
        }
    }
}
```

And everything should be fine. Right?

```
error[E0509]: cannot move out of type `HelloTriangleApplication`, which implements the `Drop` trait
  --> src\instance.rs:49:9
   |
49 |         self.event_loop.run(move |event, _, control_flow| {
   |         ^^^^^^^^^^^^^^^
   |         |
   |         cannot move out of here
   |         move occurs because `self.event_loop` has type `EventLoop<()>`, which does not implement the `Copy` trait
```

Well, no. The issue here is that this actually moves `event_loop` out of our application. We have `Drop` implemented, meaning we want things to be... well, sane. This explicitly makes things not sane. This whole structure here is a bit of a pickle: we're partially moving `self` (specifically our event loop) over to this new thread, but not the rest of it. This is a problem. What we need to do is move the rest of `self` into this loop. We can do that fairly easily, with `let Self { .. } = self;` within. However, there's another problem: we can't actually do that because we already partially moved `self` into the loop to begin with.

What we need to do is decouple `event_loop` from `self` entirely. Fortunately we just created an `init_window()` function which trivializes this:

```rust
pub struct HelloTriangleApplication {
    instance: ash::Instance,
}

impl HelloTriangleApplication {
    fn init_window() -> Result<(winit::event_loop::EventLoop<()>, winit::window::Window), winit::error::OsError> {
        let event_loop = EventLoop::new();
        let window = WindowBuilder::new()
            .with_resizable(false)
            .with_inner_size(PhysicalSize::new(WIDTH, HEIGHT))
            .build(&event_loop)?;
        Ok((event_loop, window))
    }
    fn init_vulkan() -> prelude::VkResult<ash::Instance> {
        let entry = Entry::linked();
        let app_info = vk::ApplicationInfo {
            api_version: vk::make_api_version(0, 1, 0, 0),
            ..Default::default()
        };
        let create_info = vk::InstanceCreateInfo {
            p_application_info: &app_info,
            ..Default::default()
        };
        unsafe { entry.create_instance(&create_info, None) }
    }
    pub fn new() -> Self {
        let instance = HelloTriangleApplication::init_vulkan().unwrap();
        Self {
            instance: instance,
        }
    }
    pub fn run(&self) {
        let (event_loop, window) = HelloTriangleApplication::init_window().unwrap();
        event_loop.run(move |event, _, control_flow| {
            *control_flow = ControlFlow::Wait;
    
            match event {
                Event::WindowEvent {
                    event: WindowEvent::CloseRequested,
                    window_id,
                } if window_id == window.id() => *control_flow = ControlFlow::Exit,
                _ => (),
            }
        });
    }
}

impl Drop for HelloTriangleApplication {
    fn drop(&mut self)
    {
        unsafe {
            self.instance.destroy_instance(None);
        }
    }
}
```

With that, we can move on to validation layers.

## Validation Layers

So, starting off with where we were before, our next goal is to make sure that things aren't actually breaking. They seem fine given that we do create a window, but that doesn't mean anything. As with the tutorial, we'll create a constant vector of strings for our layers, though we won't bother with the debug build options:

```rust
const VALIDATION_LAYERS : &[*const i8] = &[
    unsafe { CStr::from_bytes_with_nul_unchecked("VK_LAYER_KHRONOS_validation\0".as_bytes()).as_ptr() }
];
```

It's hideous because Vulkan wants C strings. That's how you get C strings in Rust. Now, we want to implement checking of validation layers. Notably, this actually requires an instantiated entry, so we insert this as a call unassociated with our overall app directly into our `init_vulkan()` function, taking the entry as an argument:

```rust
    fn check_validation_layer_support(entry: &ash::Entry) -> bool
    {
        let layer_properties = entry.enumerate_instance_layer_properties().unwrap();
        for layer in VALIDATION_LAYERS {
            if let None = layer_properties.iter().find(|l| {
                // This horrible construction is because Vulkan operates with C strings and Rust does not
               unsafe { &CStr::from_ptr(l.layer_name.as_ptr()).to_str().unwrap() == &CStr::from_ptr(*layer).to_str().unwrap() }
            })
            {
                return false
            }
        }
        true
    }
```

Just, uh, ignore that unsafe block.

With that done, we can now start worrying about required extensions. This is actually a bit of a problem: the tutorial expects glfw to handle all of this for us. We don't have that luxury in Rust; we need to know what extensions are relevant on what platforms. Given that I'm running on Windows, I'll just stick with that for the sake of this tutorial. More broadly-handled multiplatform stuff is a bit beyond scope here. So, we'll create another array:

```rust
use ash::extensions::khr::Win32Surface;
use ash::extensions::khr::Surface;
use ash::extensions::ext::DebugUtils;

const REQUIRED_EXTENSIONS : &[*const i8] = &[
    Surface::name().as_ptr(),
    Win32Surface::name().as_ptr(),
    DebugUtils::name().as_ptr(),
];
```

Now we can integrate these required extensions into our creating of an instance:

```rust
    fn init_vulkan() -> prelude::VkResult<ash::Instance> {
        let entry = Entry::linked();
        if !check_validation_layer_support(&entry)
        {
            panic!("Could not find support for all layers!");
        }
        let app_info = vk::ApplicationInfo {
            api_version: vk::make_api_version(0, 1, 0, 0),
            ..Default::default()
        };
        let mut create_info = vk::InstanceCreateInfo {
            p_application_info: &app_info,
            ..Default::default()
        };
        create_info.enabled_extension_count = REQUIRED_EXTENSIONS.len() as u32;
        create_info.pp_enabled_extension_names = REQUIRED_EXTENSIONS.as_ptr();
        unsafe { entry.create_instance(&create_info, None) }
    }
```

Everything runs, there's no issues, and surely this isn't a house of cards balanced precariously over a burning oil drum. With that, we can move on to constructing a callback function. We can start with the basic skeleton:

```rust
extern "system" fn debug_callback(
    message_severity: vk::DebugUtilsMessageSeverityFlagsEXT,
    message_type: vk::DebugUtilsMessageTypeFlagsEXT,
    callback_data: *const vk::DebugUtilsMessengerCallbackDataEXT,
    _: *mut c_void,
    ) -> vk::Bool32 {
        print!("validation layer: {}", unsafe { CStr::from_ptr((*callback_data).p_message).to_str().unwrap() });
        vk::FALSE
    }
```

Note that this is declared `extern "system"` so that Vulkan can use it as a callback. The structure is otherwise identical as the equivalent in C++, other than we explicitly ignore the user data because that's not useful for our purposes and any sane not-bare-metal Vulkan API would bury such a thing anyway. As is the running theme here, there is yet another `unsafe` block to deal with a C string, though this one also serves double-duty in dereferencing our callback data pointer. Naturally, we need to bake this callback directly into our overall struct:

```rust
pub struct HelloTriangleApplication {
    entry: ash::Entry,
    instance: ash::Instance,
    debug_messenger: vk::DebugUtilsMessengerEXT,
}
```

Fortunately for us, ash already provides the function we want to construct our messenger, so we merely need to call it, with some fiddling to make sure that we have access to everything we need:

```rust
    fn init_vulkan(entry: &ash::Entry) -> prelude::VkResult<ash::Instance> {
        if !HelloTriangleApplication::check_validation_layer_support(&entry)
        {
            panic!("Could not find support for all layers!");
        }
        let app_info = vk::ApplicationInfo {
            api_version: vk::make_api_version(0, 1, 0, 0),
            ..Default::default()
        };
        let mut create_info = vk::InstanceCreateInfo {
            p_application_info: &app_info,
            ..Default::default()
        };
        create_info.enabled_extension_count = REQUIRED_EXTENSIONS.len() as u32;
        create_info.pp_enabled_extension_names = REQUIRED_EXTENSIONS.as_ptr();
        unsafe { entry.create_instance(&create_info, None) }
    }
    fn setup_debug_messenger() -> vk::DebugUtilsMessengerCreateInfoEXT {
        vk::DebugUtilsMessengerCreateInfoEXT {
            s_type: vk::StructureType::DEBUG_UTILS_MESSENGER_CREATE_INFO_EXT,
            message_severity: vk::DebugUtilsMessageSeverityFlagsEXT::VERBOSE | vk::DebugUtilsMessageSeverityFlagsEXT::WARNING | vk::DebugUtilsMessageSeverityFlagsEXT::ERROR,
            message_type: vk::DebugUtilsMessageTypeFlagsEXT::GENERAL | vk::DebugUtilsMessageTypeFlagsEXT::VALIDATION | vk::DebugUtilsMessageTypeFlagsEXT::PERFORMANCE,
            pfn_user_callback: Some(debug_callback),
            ..Default::default()
        }
    }
    pub fn new() -> Self {
        let entry = Entry::linked();
        let instance = HelloTriangleApplication::init_vulkan(&entry).unwrap();
        let debug_messenger = unsafe { DebugUtils::new(&entry, &instance).create_debug_utils_messenger(&HelloTriangleApplication::setup_debug_messenger(), None).unwrap() };
        Self {
            entry: entry,
            instance: instance,
            debug_messenger: debug_messenger,
        }
    }
```

And then we add it to our `Drop` implementation:

```rust
impl Drop for HelloTriangleApplication {
    fn drop(&mut self)
    {
        unsafe {
            DebugUtils::new(&self.entry, &self.instance).destroy_debug_utils_messenger(self.debug_messenger, None);
            self.instance.destroy_instance(None);
        }
    }
}
```

Now, for setting up our debug messenger to play nicely with creating an instance, we have to create two messengers, in a sense. One is as we already create, and the other attaches itself to our instance:

```rust
    fn init_vulkan(entry: &ash::Entry) -> prelude::VkResult<ash::Instance> {
        if !HelloTriangleApplication::check_validation_layer_support(&entry)
        {
            panic!("Could not find support for all layers!");
        }
        let app_info = vk::ApplicationInfo {
            api_version: vk::make_api_version(0, 1, 0, 0),
            ..Default::default()
        };
        let mut create_info = vk::InstanceCreateInfo {
            p_application_info: &app_info,
            ..Default::default()
        };
        create_info.enabled_layer_count = VALIDATION_LAYERS.len() as u32;
        create_info.pp_enabled_layer_names = VALIDATION_LAYERS.as_ptr();
        create_info.enabled_extension_count = REQUIRED_EXTENSIONS.len() as u32;
        create_info.pp_enabled_extension_names = REQUIRED_EXTENSIONS.as_ptr();
        create_info.p_next = &HelloTriangleApplication::setup_debug_messenger() as *const _ as *const c_void;
        unsafe { entry.create_instance(&create_info, None) }
    }
```

(don't question that cast)

Now, if we test this, we'll see all the validation layers working properly... except `Drop` isn't being called. As it turns out, using `Drop` here is a problem: the event loop hijacks the main thread and keeps anything not brought in from dropping. So, the better approach here is a cleanup function:

```rust
    fn cleanup(&mut self)
    {
        unsafe {
            DebugUtils::new(&self.entry, &self.instance).destroy_debug_utils_messenger(self.debug_messenger, None);
            self.instance.destroy_instance(None);
        }
    }
    pub fn run(mut self) -> ! {
        let (event_loop, window) = HelloTriangleApplication::init_window().unwrap();
        event_loop.run(move |event, _, control_flow| {
            *control_flow = ControlFlow::Wait;

            match event {
                Event::WindowEvent {
                    event: WindowEvent::CloseRequested,
                    window_id,
                } if window_id == window.id() => {
                    self.cleanup();
                    *control_flow = ControlFlow::Exit
                },
                _ => (),
            }
        });
    }
```

With that, not only are we configuring our Vulkan setup, but we're even setting up validation layers to keep up with any sort of errors we make along the way! For which there will be plenty! Next time, we'll start tackling fetching details on physical devices and possibly logical devices as well!