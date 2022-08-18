---
categories: ["Rust", "Vulkan"]
title: "Learning Vulkan with Rust, Part 8: Vertex Buffers"
date: 2022-08-17T20:14:35-07:00
draft: false
url: ""
---

We [drew](https://vulkan-tutorial.com/Drawing_a_triangle/Drawing/Framebuffers) a triangle! However, that's not enough; there are plenty of inefficiencies and not-so-good design practices that we did to get there. For one, we baked in our vertex information into the shader itself! That's not scalable. So, we need to start exploring [vertex buffers](https://vulkan-tutorial.com/Vertex_buffers/Vertex_input_description) to load in vertex data from the code itself. All of the code can be found at [this repo](https://github.com/LuuBluum/Learning-Vulkan-with-Rust).

<!--more-->


First, we need to change our shader to actually accept an input position and color. I will not bother keeping around different copies of the shaders for this, because they're all identical to what's in the tutorial. Now, we need to create a vertex struct with some vertices to store things in the CPU side of things... except we don't have a linear algebra dependency yet. For this tutorial, we'll go with [glam](https://github.com/bitshifter/glam-rs) since it best matches the tutorial and is purpose-designed for what we're trying to do here.

```
[dependencies]
ash = {version = "0.37.0", features = ["linked"]}
winit = "0.27.1"
raw-window-handle = "0.5.0"
glam = "0.21.3"
```

Nothing like a new dependency. Anyway, we set things up with our new vertices:

```rust
struct Vertex {
    pos: glam::Vec2,
    color: glam::Vec3,
}
```

And our constant vector of vertices, which is... clunkier:

```rust
const VERTICES: [Vertex; 3] = [
    Vertex {
        pos: Vec2 { x: 0.0, y: -0.5 },
        color: Vec3 {
            x: 1.0,
            y: 0.0,
            z: 0.0,
        },
    },
    Vertex {
        pos: Vec2 { x: 0.5, y: 0.5 },
        color: Vec3 {
            x: 0.0,
            y: 1.0,
            z: 0.0,
        },
    },
    Vertex {
        pos: Vec2 { x: -0.5, y: 0.5 },
        color: Vec3 {
            x: 0.0,
            y: 0.0,
            z: 1.0,
        },
    },
];
```

This works well enough. We also need to implement a function to get a binding description for our vertices:

```rust
impl Vertex {
    fn get_binding_description() -> vk::VertexInputBindingDescription {
        vk::VertexInputBindingDescription {
            binding: 0,
            stride: size_of::<Vertex>() as u32,
            input_rate: vk::VertexInputRate::VERTEX,
        }
    }
}
```

Now we need another function, for getting the attribute descriptions:

```rust
            vk::VertexInputAttributeDescription {
                binding: 0,
                location: 0,
                format: vk::Format::R32G32_SFLOAT,
                offset: // What do we do here?
            },
```

So... we have a problem. There is no `offsetof` macro in Rust. There is, however, a crate for this functionality, but it also means our Vertex needs to be `#[repr(C)]` for it to work. Which makes sense, because we're sending this over to the GPU anyway. Add the `memoffset` crate to our project:

```
[dependencies]
ash = {version = "0.37.0", features = ["linked"]}
winit = "0.27.1"
raw-window-handle = "0.5.0"
glam = "0.21.3"
memoffset = "0.6.5"
```

Now our function is nice and coherent:

```rust
    fn get_attribute_descriptions() -> [vk::VertexInputAttributeDescription; 2] {
        [
            vk::VertexInputAttributeDescription {
                binding: 0,
                location: 0,
                format: vk::Format::R32G32_SFLOAT,
                offset: offset_of!(Vertex, pos) as u32,
            },
            vk::VertexInputAttributeDescription {
                binding: 0,
                location: 1,
                format: vk::Format::R32G32B32_SFLOAT,
                offset: offset_of!(Vertex, color) as u32,
            },
        ]
    }
```

With that out of the way, we can now have our pipeline understand that we have some vertex inputs:

```rust
        //...
        let binding_description = Vertex::get_binding_description();
        let attribute_descriptions = Vertex::get_attribute_descriptions();

        let vertex_input_info = vk::PipelineVertexInputStateCreateInfo {
            s_type: vk::StructureType::PIPELINE_VERTEX_INPUT_STATE_CREATE_INFO,
            vertex_binding_description_count: 1,
            p_vertex_binding_descriptions: &binding_description,
            vertex_attribute_description_count: attribute_descriptions.len() as u32,
            p_vertex_attribute_descriptions: attribute_descriptions.as_ptr(),
            ..Default::default()
        };
        //...
```

Our program now crashes. Progress!

## Creating the Vertex Buffer

So, now we actually need to create our [vertex buffer](https://vulkan-tutorial.com/en/Vertex_buffers/Vertex_buffer_creation) so that we can access our vertices. This means yet another function, this time to create our buffer:

```rust
struct VulkanDetails {
    entry: ash::Entry,
    instance: ash::Instance,
    debug_messenger: vk::DebugUtilsMessengerEXT,
    surface: vk::SurfaceKHR,
    physical_device: vk::PhysicalDevice,
    device: ash::Device,
    graphics_queue: vk::Queue,
    present_queue: vk::Queue,
    swap_chain: vk::SwapchainKHR,
    swap_chain_images: Vec<vk::Image>,
    swap_chain_image_format: vk::Format,
    swap_chain_extent: vk::Extent2D,
    swap_chain_image_views: Vec<vk::ImageView>,
    render_pass: vk::RenderPass,
    pipeline_layout: vk::PipelineLayout,
    graphics_pipeline: vk::Pipeline,
    swap_chain_framebuffers: Vec<vk::Framebuffer>,
    command_pool: vk::CommandPool,
    vertex_buffer: vk::Buffer,
    command_buffers: Vec<vk::CommandBuffer>,
    image_available_semaphores: Vec<vk::Semaphore>,
    render_finished_semaphores: Vec<vk::Semaphore>,
    in_flight_fences: Vec<vk::Fence>,
    framebuffer_resized: bool,
    current_frame: usize,
}

    fn create_vertex_buffer(device: &ash::Device) -> vk::Buffer {
        let buffer_info = vk::BufferCreateInfo {
            s_type: vk::StructureType::BUFFER_CREATE_INFO,
            size: (VERTICES.len() * size_of::<Vertex>()) as u64,
            usage: vk::BufferUsageFlags::VERTEX_BUFFER,
            sharing_mode: vk::SharingMode::EXCLUSIVE,
            ..Default::default()
        };
        unsafe { device.create_buffer(&buffer_info, None).unwrap() }
    }
    fn cleanup(&mut self) {
        unsafe {
            self.cleanup_swap_chain();
            self.device.destroy_buffer(self.vertex_buffer, None);
            self.device.destroy_pipeline(self.graphics_pipeline, None);
            self.device
                .destroy_pipeline_layout(self.pipeline_layout, None);
            self.device.destroy_render_pass(self.render_pass, None);
            for i in 0..MAX_FRAMES_IN_FLIGHT {
                self.device
                    .destroy_semaphore(self.image_available_semaphores[i], None);
                self.device
                    .destroy_semaphore(self.render_finished_semaphores[i], None);
                self.device.destroy_fence(self.in_flight_fences[i], None);
            }
            self.device.destroy_command_pool(self.command_pool, None);
            self.device.destroy_device(None);
            DebugUtils::new(&self.entry, &self.instance)
                .destroy_debug_utils_messenger(self.debug_messenger, None);
            Surface::new(&self.entry, &self.instance).destroy_surface(self.surface, None);
            self.instance.destroy_instance(None);
        }
    }
```

That gives us a vertex buffer, but no memory. So, we need a function to find our specified memory:

```rust
    fn find_memory_type(
        instance: &ash::Instance,
        physical_device: &vk::PhysicalDevice,
        type_filter: u32,
        properties: vk::MemoryPropertyFlags,
    ) -> u32 {
        let mem_properties =
            unsafe { instance.get_physical_device_memory_properties(*physical_device) };
        for i in 0..mem_properties.memory_type_count {
            if (type_filter & (1 << i)) != 0
                && mem_properties.memory_types[i as usize].property_flags & properties == properties
            {
                return i;
            }
        }
        panic!("Unable to find suitable memory type!")
    }
```

Plus modifying our buffer creation to account for this:

```rust
    fn create_vertex_buffer(
        instance: &ash::Instance,
        physical_device: &vk::PhysicalDevice,
        device: &ash::Device,
    ) -> (vk::Buffer, vk::DeviceMemory) {
        let buffer_info = vk::BufferCreateInfo {
            s_type: vk::StructureType::BUFFER_CREATE_INFO,
            size: (VERTICES.len() * size_of::<Vertex>()) as u64,
            usage: vk::BufferUsageFlags::VERTEX_BUFFER,
            sharing_mode: vk::SharingMode::EXCLUSIVE,
            ..Default::default()
        };
        let buffer = unsafe { device.create_buffer(&buffer_info, None).unwrap() };

        let mem_requirements = unsafe { device.get_buffer_memory_requirements(buffer) };

        let alloc_info = vk::MemoryAllocateInfo {
            s_type: vk::StructureType::MEMORY_ALLOCATE_INFO,
            allocation_size: mem_requirements.size,
            memory_type_index: VulkanDetails::find_memory_type(
                instance,
                physical_device,
                mem_requirements.memory_type_bits,
                vk::MemoryPropertyFlags::HOST_VISIBLE | vk::MemoryPropertyFlags::HOST_COHERENT,
            ),
            ..Default::default()
        };

        let buffer_memory = unsafe { device.allocate_memory(&alloc_info, None).unwrap() };

        unsafe { device.bind_buffer_memory(buffer, buffer_memory, 0).unwrap() };

        (buffer, buffer_memory)
    }
    fn cleanup(&mut self) {
        unsafe {
            self.cleanup_swap_chain();
            self.device.destroy_buffer(self.vertex_buffer, None);
            self.device.free_memory(self.vertex_buffer_memory, None);
            self.device.destroy_pipeline(self.graphics_pipeline, None);
            self.device
                .destroy_pipeline_layout(self.pipeline_layout, None);
            self.device.destroy_render_pass(self.render_pass, None);
            for i in 0..MAX_FRAMES_IN_FLIGHT {
                self.device
                    .destroy_semaphore(self.image_available_semaphores[i], None);
                self.device
                    .destroy_semaphore(self.render_finished_semaphores[i], None);
                self.device.destroy_fence(self.in_flight_fences[i], None);
            }
            self.device.destroy_command_pool(self.command_pool, None);
            self.device.destroy_device(None);
            DebugUtils::new(&self.entry, &self.instance)
                .destroy_debug_utils_messenger(self.debug_messenger, None);
            Surface::new(&self.entry, &self.instance).destroy_surface(self.surface, None);
            self.instance.destroy_instance(None);
        }
    }
```

Now we need to actually bind the memory. The Vulkan tutorial gives us this:

```c++
void* data;
vkMapMemory(device, vertexBufferMemory, 0, bufferInfo.size, 0, &data);
```

This is a problem. Casually casting away `void*` is... nontrivial in Rust. This is going to be messy. Let's try... `copy_from_nonoverlapping`. That seems right. Maybe.

```rust
    fn create_vertex_buffer(
        instance: &ash::Instance,
        physical_device: &vk::PhysicalDevice,
        device: &ash::Device,
    ) -> (vk::Buffer, vk::DeviceMemory) {
        let buffer_info = vk::BufferCreateInfo {
            s_type: vk::StructureType::BUFFER_CREATE_INFO,
            size: (VERTICES.len() * size_of::<Vertex>()) as u64,
            usage: vk::BufferUsageFlags::VERTEX_BUFFER,
            sharing_mode: vk::SharingMode::EXCLUSIVE,
            ..Default::default()
        };
        let buffer = unsafe { device.create_buffer(&buffer_info, None).unwrap() };

        let mem_requirements = unsafe { device.get_buffer_memory_requirements(buffer) };

        let alloc_info = vk::MemoryAllocateInfo {
            s_type: vk::StructureType::MEMORY_ALLOCATE_INFO,
            allocation_size: mem_requirements.size,
            memory_type_index: VulkanDetails::find_memory_type(
                instance,
                physical_device,
                mem_requirements.memory_type_bits,
                vk::MemoryPropertyFlags::HOST_VISIBLE | vk::MemoryPropertyFlags::HOST_COHERENT,
            ),
            ..Default::default()
        };

        let buffer_memory = unsafe { device.allocate_memory(&alloc_info, None).unwrap() };

        unsafe { device.bind_buffer_memory(buffer, buffer_memory, 0).unwrap() };
        unsafe {
            let data = device
                .map_memory(
                    buffer_memory,
                    0,
                    buffer_info.size,
                    vk::MemoryMapFlags::empty(),
                )
                .unwrap();

            data.copy_from_nonoverlapping(VERTICES.as_ptr() as *const c_void, VERTICES.len());
            device.unmap_memory(buffer_memory);
        }

        (buffer, buffer_memory)
    }
```

Then we link it into the command buffer:

```rust
    fn record_command_buffer(&self, image_index: usize) {
        let begin_info = vk::CommandBufferBeginInfo {
            s_type: vk::StructureType::COMMAND_BUFFER_BEGIN_INFO,
            ..Default::default()
        };
        unsafe {
            self.device
                .begin_command_buffer(self.command_buffers[self.current_frame], &begin_info)
                .unwrap();
        }
        let clear_color = vk::ClearValue {
            color: vk::ClearColorValue {
                float32: [0.0, 0.0, 0.0, 1.0],
            },
        };
        let render_pass_info = vk::RenderPassBeginInfo {
            s_type: vk::StructureType::RENDER_PASS_BEGIN_INFO,
            render_pass: self.render_pass,
            framebuffer: self.swap_chain_framebuffers[image_index],
            render_area: vk::Rect2D {
                offset: vk::Offset2D { x: 0, y: 0 },
                extent: self.swap_chain_extent,
            },
            clear_value_count: 1,
            p_clear_values: &clear_color,
            ..Default::default()
        };
        unsafe {
            self.device.cmd_begin_render_pass(
                self.command_buffers[self.current_frame],
                &render_pass_info,
                vk::SubpassContents::INLINE,
            );
            self.device.cmd_bind_pipeline(
                self.command_buffers[self.current_frame],
                vk::PipelineBindPoint::GRAPHICS,
                self.graphics_pipeline,
            );
        }
        let viewport = vk::Viewport {
            x: 0.0,
            y: 0.0,
            width: self.swap_chain_extent.width as f32,
            height: self.swap_chain_extent.height as f32,
            min_depth: 0.0,
            max_depth: 1.0,
        };
        unsafe {
            self.device
                .cmd_set_viewport(self.command_buffers[self.current_frame], 0, &[viewport]);
        }
        let scissor = vk::Rect2D {
            offset: vk::Offset2D { x: 0, y: 0 },
            extent: self.swap_chain_extent,
        };
        let vertex_buffers = [self.vertex_buffer];
        let offsets: [vk::DeviceSize; 1] = [0];
        unsafe {
            self.device
                .cmd_set_scissor(self.command_buffers[self.current_frame], 0, &[scissor]);
            self.device.cmd_bind_vertex_buffers(
                self.command_buffers[self.current_frame],
                0,
                &vertex_buffers,
                &offsets,
            );
            self.device
                .cmd_draw(self.command_buffers[self.current_frame], 3, 1, 0, 0);
            self.device
                .cmd_end_render_pass(self.command_buffers[self.current_frame]);
            self.device
                .end_command_buffer(self.command_buffers[self.current_frame])
                .unwrap();
        }
    }
```

It compiles, it runs, and... we get a black box! I think our attempt at copying didn't work. Maybe instead of converting the array into a `c_void` pointer, we should do it the other way around:

```rust
        unsafe {
            device.bind_buffer_memory(buffer, buffer_memory, 0).unwrap();
            let data = device
                .map_memory(
                    buffer_memory,
                    0,
                    buffer_info.size,
                    vk::MemoryMapFlags::empty(),
                )
                .unwrap();

            (data as *mut [Vertex; 3]).write(VERTICES);
            device.unmap_memory(buffer_memory);
        }
```

There we go! Now we have a triangle again! One where the vertex details live in our Rust code, rather than in the shader itself! We can do better, though.

## The Staging Buffer

We can use an intermediary [staging buffer](https://vulkan-tutorial.com/en/Vertex_buffers/Staging_buffer) to store memory before it ends up copied over. Now, the tutorial gives us an option to use a dedicated transfer queue. We'll consider that once we work out the other details. First, we abstract out our buffer creation:

```rust
    fn create_buffer(
        instance: &ash::Instance,
        physical_device: &vk::PhysicalDevice,
        device: &ash::Device,
        size: vk::DeviceSize,
        usage: vk::BufferUsageFlags,
        properties: vk::MemoryPropertyFlags,
    ) -> (vk::Buffer, vk::DeviceMemory) {
        let buffer_info = vk::BufferCreateInfo {
            s_type: vk::StructureType::BUFFER_CREATE_INFO,
            size,
            usage,
            sharing_mode: vk::SharingMode::EXCLUSIVE,
            ..Default::default()
        };
        let buffer = unsafe { device.create_buffer(&buffer_info, None).unwrap() };

        let mem_requirements = unsafe { device.get_buffer_memory_requirements(buffer) };

        let alloc_info = vk::MemoryAllocateInfo {
            s_type: vk::StructureType::MEMORY_ALLOCATE_INFO,
            allocation_size: mem_requirements.size,
            memory_type_index: VulkanDetails::find_memory_type(
                instance,
                physical_device,
                mem_requirements.memory_type_bits,
                properties,
            ),
            ..Default::default()
        };

        let buffer_memory = unsafe { device.allocate_memory(&alloc_info, None).unwrap() };

        unsafe {
            device.bind_buffer_memory(buffer, buffer_memory, 0).unwrap();
        }
        (buffer, buffer_memory)
    }
```

Then we can deconstruct our original `create_vertex_buffer` to play with this:

```rust
    fn create_vertex_buffer(
        instance: &ash::Instance,
        physical_device: &vk::PhysicalDevice,
        device: &ash::Device,
    ) -> (vk::Buffer, vk::DeviceMemory) {
        let buffer_size = (VERTICES.len() * size_of::<Vertex>()) as u64;
        let (buffer, buffer_memory) = VulkanDetails::create_buffer(
            instance,
            physical_device,
            device,
            buffer_size,
            vk::BufferUsageFlags::VERTEX_BUFFER,
            vk::MemoryPropertyFlags::HOST_VISIBLE | vk::MemoryPropertyFlags::HOST_COHERENT,
        );

        unsafe {
            let data = device
                .map_memory(buffer_memory, 0, buffer_size, vk::MemoryMapFlags::empty())
                .unwrap();

            (data as *mut [Vertex; 3]).write(VERTICES);
            device.unmap_memory(buffer_memory);
        }

        (buffer, buffer_memory)
    }
```

Almost there. We need to copy over the data between the buffers, meaning another function:

```rust
    fn copy_buffer(
        device: &ash::Device,
        command_pool: &vk::CommandPool,
        graphics_queue: &vk::Queue,
        src_buffer: &vk::Buffer,
        dst_buffer: &mut vk::Buffer,
        size: vk::DeviceSize,
    ) {
        let alloc_info = vk::CommandBufferAllocateInfo {
            s_type: vk::StructureType::COMMAND_BUFFER_ALLOCATE_INFO,
            level: vk::CommandBufferLevel::PRIMARY,
            command_pool: *command_pool,
            command_buffer_count: 1,
            ..Default::default()
        };
        let command_buffer = unsafe { device.allocate_command_buffers(&alloc_info).unwrap()[0] };

        let begin_info = vk::CommandBufferBeginInfo {
            s_type: vk::StructureType::COMMAND_BUFFER_BEGIN_INFO,
            flags: vk::CommandBufferUsageFlags::ONE_TIME_SUBMIT,
            ..Default::default()
        };

        let copy_region = vk::BufferCopy {
            src_offset: 0,
            dst_offset: 0,
            size,
        };

        let submit_info = vk::SubmitInfo {
            s_type: vk::StructureType::SUBMIT_INFO,
            command_buffer_count: 1,
            p_command_buffers: &command_buffer,
            ..Default::default()
        };

        unsafe {
            device
                .begin_command_buffer(command_buffer, &begin_info)
                .unwrap();
            device.cmd_copy_buffer(command_buffer, *src_buffer, *dst_buffer, &[copy_region]);
            device.end_command_buffer(command_buffer).unwrap();
            device
                .queue_submit(*graphics_queue, &[submit_info], vk::Fence::null())
                .unwrap();
            device.queue_wait_idle(*graphics_queue).unwrap();
            device.free_command_buffers(*command_pool, &[command_buffer]);
        }
    }
```

Notably our command pool and graphics queue are now involved in all of this. With that, we can finish out our vertex buffer creation:

```rust
    fn create_vertex_buffer(
        instance: &ash::Instance,
        physical_device: &vk::PhysicalDevice,
        device: &ash::Device,
        command_pool: &vk::CommandPool,
        graphics_queue: &vk::Queue,
    ) -> (vk::Buffer, vk::DeviceMemory) {
        let buffer_size = (VERTICES.len() * size_of::<Vertex>()) as u64;
        let (staging_buffer, staging_buffer_memory) = VulkanDetails::create_buffer(
            instance,
            physical_device,
            device,
            buffer_size,
            vk::BufferUsageFlags::TRANSFER_SRC,
            vk::MemoryPropertyFlags::HOST_VISIBLE | vk::MemoryPropertyFlags::HOST_COHERENT,
        );

        unsafe {
            let data = device
                .map_memory(
                    staging_buffer_memory,
                    0,
                    buffer_size,
                    vk::MemoryMapFlags::empty(),
                )
                .unwrap();

            (data as *mut [Vertex; VERTICES.len()]).write(VERTICES);
            device.unmap_memory(staging_buffer_memory);
        }
        let (mut buffer, buffer_memory) = VulkanDetails::create_buffer(
            instance,
            physical_device,
            device,
            buffer_size,
            vk::BufferUsageFlags::TRANSFER_DST | vk::BufferUsageFlags::VERTEX_BUFFER,
            vk::MemoryPropertyFlags::DEVICE_LOCAL,
        );

        VulkanDetails::copy_buffer(
            device,
            command_pool,
            graphics_queue,
            &staging_buffer,
            &mut buffer,
            buffer_size,
        );

        unsafe {
            device.destroy_buffer(staging_buffer, None);
            device.free_memory(staging_buffer_memory, None);
        }

        (buffer, buffer_memory)
    }
```

Now, there's some details about allocation here, and there's that earlier point about using a separate queue for all of this. All of that seems grossly overkill for what simple stuff we're doing here so we're not going to do any of it. Maybe at some point, but not right now. We're not even done yet, after all: we have one more buffer to take care of.

## Index Buffer

With the introduction of the [index buffer](https://vulkan-tutorial.com/en/Vertex_buffers/Index_buffer), our triangle will become a rectangle. Mostly because it allows us to have overlapping vertices that we want to avoid keeping in memory twice. Firstly, a new set of vertices:

```rust
const VERTICES: [Vertex; 4] = [
    Vertex {
        pos: Vec2 { x: -0.5, y: -0.5 },
        color: Vec3 {
            x: 1.0,
            y: 0.0,
            z: 0.0,
        },
    },
    Vertex {
        pos: Vec2 { x: 0.5, y: -0.5 },
        color: Vec3 {
            x: 0.0,
            y: 1.0,
            z: 0.0,
        },
    },
    Vertex {
        pos: Vec2 { x: 0.5, y: 0.5 },
        color: Vec3 {
            x: 0.0,
            y: 0.0,
            z: 1.0,
        },
    },
    Vertex {
        pos: Vec2 { x: -0.5, y: 0.5 },
        color: Vec3 {
            x: 1.0,
            y: 1.0,
            z: 1.0,
        },
    },
];
```

We also create some indices:

```rust
const INDICES: [u16; 6] = [0, 1, 2, 2, 3, 0];
```

The details of creating the index buffers are more-or-less identical to the vertex buffers:

```rust
    fn create_index_buffer(
        instance: &ash::Instance,
        physical_device: &vk::PhysicalDevice,
        device: &ash::Device,
        command_pool: &vk::CommandPool,
        graphics_queue: &vk::Queue,
    ) -> (vk::Buffer, vk::DeviceMemory) {
        let buffer_size = (INDICES.len() * size_of::<u16>()) as u64;
        let (staging_buffer, staging_buffer_memory) = VulkanDetails::create_buffer(
            instance,
            physical_device,
            device,
            buffer_size,
            vk::BufferUsageFlags::TRANSFER_SRC,
            vk::MemoryPropertyFlags::HOST_VISIBLE | vk::MemoryPropertyFlags::HOST_COHERENT,
        );

        unsafe {
            let data = device
                .map_memory(
                    staging_buffer_memory,
                    0,
                    buffer_size,
                    vk::MemoryMapFlags::empty(),
                )
                .unwrap();

            (data as *mut [u16; INDICES.len()]).write(INDICES);
            device.unmap_memory(staging_buffer_memory);
        }
        let (mut buffer, buffer_memory) = VulkanDetails::create_buffer(
            instance,
            physical_device,
            device,
            buffer_size,
            vk::BufferUsageFlags::TRANSFER_DST | vk::BufferUsageFlags::INDEX_BUFFER,
            vk::MemoryPropertyFlags::DEVICE_LOCAL,
        );

        VulkanDetails::copy_buffer(
            device,
            command_pool,
            graphics_queue,
            &staging_buffer,
            &mut buffer,
            buffer_size,
        );

        unsafe {
            device.destroy_buffer(staging_buffer, None);
            device.free_memory(staging_buffer_memory, None);
        }

        (buffer, buffer_memory)
    }
```

Then all that's left is modifying our drawing:

```rust
        unsafe {
            self.device
                .cmd_set_scissor(self.command_buffers[self.current_frame], 0, &[scissor]);
            self.device.cmd_bind_vertex_buffers(
                self.command_buffers[self.current_frame],
                0,
                &vertex_buffers,
                &offsets,
            );
            self.device.cmd_bind_index_buffer(
                self.command_buffers[self.current_frame],
                self.index_buffer,
                0,
                vk::IndexType::UINT16,
            );
            self.device.cmd_draw_indexed(
                self.command_buffers[self.current_frame],
                INDICES.len() as u32,
                1,
                0,
                0,
                0,
            );
            self.device
                .cmd_end_render_pass(self.command_buffers[self.current_frame]);
            self.device
                .end_command_buffer(self.command_buffers[self.current_frame])
                .unwrap();
        }
```

With that, we now have a rectangle instead of a triangle! Plus, our vertex data can now come from anywhere, not just our shaders themselves!