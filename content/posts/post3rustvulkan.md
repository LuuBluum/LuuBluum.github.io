---
categories: ["Rust", "Vulkan"]
title: "Learning Vulkan with Rust, Part 3: Adding Physical and Logical Devices"
date: 2022-08-01T15:54:34-07:00
draft: false
url: ""
---

With validation layers set up back in [part 2](https://luubluum.github.io/posts/post2rustvulkan/), we can finally move on to adding [physical](https://vulkan-tutorial.com/Drawing_a_triangle/Setup/Physical_devices_and_queue_families) and [logical](https://vulkan-tutorial.com/Drawing_a_triangle/Setup/Logical_device_and_queues) devices! All of the code can be found at [this repo](https://github.com/LuuBluum/Learning-Vulkan-with-Rust).

<!--more-->

Last time, we left the code in a bit of a messy state. Let's clean it up real quick:

```rust
impl HelloTriangleApplication {
    pub fn new() -> Self {
        let (entry, instance, debug_messenger) = HelloTriangleApplication::init_vulkan();
        Self {
            entry: entry,
            instance: instance,
            debug_messenger: debug_messenger,
        }
    }
    fn init_vulkan() -> (
        ash::Entry,
        ash::Instance,
        vk::DebugUtilsMessengerEXT,
    ) {
        let entry = Entry::linked();
        let instance = HelloTriangleApplication::create_instance(&entry).unwrap();
        let debug_messenger = HelloTriangleApplication::create_debug_messenger(&entry, &instance);
        (entry, instance, debug_messenger)
    }
    fn create_instance(entry: &ash::Entry) -> prelude::VkResult<ash::Instance> {
        if !HelloTriangleApplication::check_validation_layer_support(&entry)
        {
            panic!("Could not find support for all layers!");
        }
        let app_info = vk::ApplicationInfo {
            s_type: vk::StructureType::APPLICATION_INFO,
            p_application_name: unsafe { CStr::from_bytes_with_nul_unchecked("Hello Triangle\0".as_bytes()).as_ptr() },
            application_version: vk::make_api_version(0, 1, 0, 0),
            p_engine_name: unsafe { CStr::from_bytes_with_nul_unchecked("No Engine\0".as_bytes()).as_ptr() },
            engine_version: vk::make_api_version(0, 1, 0, 0),
            api_version: vk::make_api_version(0, 1, 0, 0),
            ..Default::default()
        };
        let create_info = vk::InstanceCreateInfo {
            p_application_info: &app_info,
            enabled_layer_count: VALIDATION_LAYERS.len() as u32,
            pp_enabled_layer_names: VALIDATION_LAYERS.as_ptr(),
            enabled_extension_count: REQUIRED_EXTENSIONS.len() as u32,
            pp_enabled_extension_names: REQUIRED_EXTENSIONS.as_ptr(),
            p_next: &HelloTriangleApplication::populate_debug_messenger_create_info() as *const _ as *const c_void,
            ..Default::default()
        };
    }
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
    fn create_debug_messenger(entry: &ash::Entry, instance: &ash::Instance) -> vk::DebugUtilsMessengerEXT {
        unsafe { DebugUtils::new(&entry, &instance).create_debug_utils_messenger(&HelloTriangleApplication::populate_debug_messenger_create_info(), None).unwrap() }
    }
    fn populate_debug_messenger_create_info() -> vk::DebugUtilsMessengerCreateInfoEXT {
        vk::DebugUtilsMessengerCreateInfoEXT {
            s_type: vk::StructureType::DEBUG_UTILS_MESSENGER_CREATE_INFO_EXT,
            message_severity: vk::DebugUtilsMessageSeverityFlagsEXT::VERBOSE |
                              vk::DebugUtilsMessageSeverityFlagsEXT::WARNING |
                              vk::DebugUtilsMessageSeverityFlagsEXT::ERROR,
            message_type: vk::DebugUtilsMessageTypeFlagsEXT::GENERAL |
                          vk::DebugUtilsMessageTypeFlagsEXT::VALIDATION |
                          vk::DebugUtilsMessageTypeFlagsEXT::PERFORMANCE,
            pfn_user_callback: Some(debug_callback),
            ..Default::default()
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
    fn init_window() -> Result<(winit::event_loop::EventLoop<()>, winit::window::Window), winit::error::OsError> {
        let event_loop = EventLoop::new();
        let window = WindowBuilder::new()
            .with_resizable(false)
            .with_inner_size(PhysicalSize::new(WIDTH, HEIGHT))
            .build(&event_loop)?;
        Ok((event_loop, window))
    }
    fn cleanup(&mut self) {
        unsafe {
            DebugUtils::new(&self.entry, &self.instance).destroy_debug_utils_messenger(self.debug_messenger, None);
            self.instance.destroy_instance(None);
        }
    }
}
```

Now it more closely follows the tutorial. Well, almost: we stripped out the bulk of the `void`-returning functions and our `main_loop` is still completely encompassed within `run`, with `init`-methods being regulated to `new` (since we're keeping local copies), but that's fine. If we wanted we could drop keeping anything saved locally and instead have everything instanced within `run`, but that seems rather clunky. Window initialization, contrary to how the C++ tutorial works, does not need to happen before Vulkan initialization here, so we're fine handling things this way. Especially given that `run` completely takes over the control flow and that there's no way to return from its execution. Perhaps a more structured approach for this sort of design would create the window immediately after reading whatever necessary data is needed to configure the window creation, and then promptly enter the event loop and instantiate everything in there, but that's not how this tutorial is structured and so neither will be our code.

## Physical Devices

Anyway, we can now start adding in handling for physical devices:

```rust
    fn init_vulkan() -> (
        ash::Entry,
        ash::Instance,
        vk::DebugUtilsMessengerEXT,
    ) {
        let entry = Entry::linked();
        let instance = HelloTriangleApplication::create_instance(&entry).unwrap();
        let debug_messenger = HelloTriangleApplication::create_debug_messenger(&entry, &instance);
        HelloTriangleApplication::pick_physical_device();
        (entry, instance, debug_messenger)
    }
    fn pick_physical_device() {

    }
```

As for populating this function, it turns out we can make it closely mimic the C++ equivalent:

```rust
    fn pick_physical_device(instance: &ash::Instance) -> vk::PhysicalDevice {
        let mut physical_device: Option<vk::PhysicalDevice> = None;
        let devices = unsafe { instance.enumerate_physical_devices().unwrap() };
        if devices.len() == 0 {
            panic!("Failed to find GPUs with Vulkan support!");
        }
        for device in devices {
            if HelloTriangleApplication::is_device_suitable(&instance, &device) {
                physical_device = Some(device);
            }
        }
        physical_device.unwrap()
    }
    fn is_device_suitable(_device: &vk::PhysicalDevice) -> bool {
        true
    }
```

The only differences here are the panics, which of course are not nearly as proper a way of handling things as would be passing up results, but for the sake of simplicity we'll just keep them as panics for now. We skip down now to trying to find queue families. Funny enough, the tutorial here explains the beauty of C++17's `optional` type. Go figure. So, with a little bit of finagling, we can create something a bit more Rust-like for the queue family check than a simple for loop with an index counter:

```rust
    fn is_device_suitable(instance: &ash::Instance, device: &vk::PhysicalDevice) -> bool {
        HelloTriangleApplication::find_queue_familes(instance, device).is_some()
    }
    fn find_queue_familes(instance: &ash::Instance, device: &vk::PhysicalDevice) -> Option<usize> {
        let queue_family_properties = unsafe { instance.get_physical_device_queue_family_properties(*device) };
        queue_family_properties.iter().position(|&queue_family| queue_family.queue_flags & vk::QueueFlags::GRAPHICS == vk::QueueFlags::GRAPHICS)
    }
```

The self-check against the bitwise operation there is effectively the same as coercing it to `bool`. Just cleaner. No point adding in their quick-exit optimizations and whatnot: we already have that baked in. Though honestly in C++ if you're interating over a vector and care about the index, just go with an old-school `for` loop rather than bothering with a for-each. Otherwise it just looks silly.

## Logical Devices

The logical device starts similarly to the physical device, though in this case we're instantiating the structure rather than being provided it from the instance:

```rust


impl HelloTriangleApplication {
    pub fn new() -> Self {
        let (entry, instance, debug_messenger, physical_device, device) = HelloTriangleApplication::init_vulkan();
        Self {
            entry: entry,
            instance: instance,
            debug_messenger: debug_messenger,
            physical_device: physical_device,
            device: device,
        }
    }
    fn init_vulkan() -> (
        ash::Entry,
        ash::Instance,
        vk::DebugUtilsMessengerEXT,
        vk::PhysicalDevice,
        ash::Device,
    ) {
        let entry = Entry::linked();
        let instance = HelloTriangleApplication::create_instance(&entry).unwrap();
        let debug_messenger = HelloTriangleApplication::create_debug_messenger(&entry, &instance);
        let physical_device = HelloTriangleApplication::pick_physical_device(&instance);
        let device = HelloTriangleApplication::create_logical_device();
        (entry, instance, debug_messenger, physical_device, device)
    }
    fn create_logical_device() -> vk::Device {
        // Actually create the device here
    }
}
```

The first object of the struct we wish to populate will be info about the queue. We'll be needing the instance and the physical device to queue that again:

```rust
    fn create_logical_device(instance: &ash::Instance, physical_device: &vk::PhysicalDevice) -> ash::Device {
        let device_queue_create_info = vk::DeviceQueueCreateInfo {
            s_type: vk::StructureType::DEVICE_QUEUE_CREATE_INFO,
            queue_family_index: HelloTriangleApplication::find_queue_familes(instance, physical_device).unwrap() as u32,
            queue_count: 1,
            p_queue_priorities: &1.0,
            ..Default::default()
        };
    }
```

Still incomplete, but at least now we have the device create info. Ignore that questionable value for `p_queue_priorities`; apparently that's what it wants. Now we fill out the create info proper, and obtain our logical device:

```rust
    fn create_logical_device(instance: &ash::Instance, physical_device: &vk::PhysicalDevice) -> ash::Device {
        let device_queue_create_info = vk::DeviceQueueCreateInfo {
            s_type: vk::StructureType::DEVICE_QUEUE_CREATE_INFO,
            queue_family_index: HelloTriangleApplication::find_queue_familes(instance, physical_device).unwrap() as u32,
            queue_count: 1,
            p_queue_priorities: &1.0,
            ..Default::default()
        };
        let device_features = vk::PhysicalDeviceFeatures {
            ..Default::default()
        };
        let device_create_info = vk::DeviceCreateInfo {
            s_type: vk::StructureType::DEVICE_CREATE_INFO,
            queue_create_info_count: 1,
            p_queue_create_infos: &device_queue_create_info,
            p_enabled_features: &device_features,
            enabled_layer_count: VALIDATION_LAYERS.len() as u32,
            pp_enabled_layer_names: VALIDATION_LAYERS.as_ptr(),
            ..Default::default()
        };
        unsafe { instance.create_device(*physical_device, &device_create_info, None).unwrap() }
    }
    fn cleanup(&mut self) {
        unsafe {
            self.device.destroy_device(None);
            DebugUtils::new(&self.entry, &self.instance).destroy_debug_utils_messenger(self.debug_messenger, None);
            self.instance.destroy_instance(None);
        }
    }
```

Now we just do one last little bit to get the graphics queue:

```rust
pub struct HelloTriangleApplication {
    entry: ash::Entry,
    instance: ash::Instance,
    debug_messenger: vk::DebugUtilsMessengerEXT,
    physical_device: vk::PhysicalDevice,
    device: ash::Device,
    graphics_queue: vk::Queue,
}

impl HelloTriangleApplication {
    pub fn new() -> Self {
        let (entry, instance, debug_messenger, physical_device, device, graphics_queue) = HelloTriangleApplication::init_vulkan();
        Self {
            entry: entry,
            instance: instance,
            debug_messenger: debug_messenger,
            physical_device: physical_device,
            device: device,
            graphics_queue: graphics_queue
        }
    }
    fn init_vulkan() -> (
        ash::Entry,
        ash::Instance,
        vk::DebugUtilsMessengerEXT,
        vk::PhysicalDevice,
        ash::Device,
        vk::Queue,
    ) {
        let entry = Entry::linked();
        let instance = HelloTriangleApplication::create_instance(&entry).unwrap();
        let debug_messenger = HelloTriangleApplication::create_debug_messenger(&entry, &instance);
        let physical_device = HelloTriangleApplication::pick_physical_device(&instance);
        let device = HelloTriangleApplication::create_logical_device(&instance, &physical_device);
        let graphics_queue = unsafe { device.get_device_queue(HelloTriangleApplication::find_queue_familes(&instance, &physical_device).unwrap() as u32, 0) };
        (entry, instance, debug_messenger, physical_device, device, graphics_queue)
    }
```

And we're there! We have the complete implementation of our physical and logical devices. The next thing to tackle is actually synchronizing all of this Vulkan creation with the window itself, so that we can actually draw things.