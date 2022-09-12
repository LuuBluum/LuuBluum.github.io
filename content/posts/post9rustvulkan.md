---
categories: ["Rust", "Vulkan"]
title: "Learning Vulkan with Rust, Part 9: Uniform Buffers"
date: 2022-09-12T12:32:40-07:00
draft: false
url: ""
---

Now that we have [vertex buffers](https://vulkan-tutorial.com/Vertex_buffers/Vertex_input_description) all set up, there remains one glaring flaw in our approach. Namely, we're still limited to 2D. What about 3D? For that, we need to implement [uniform buffers](https://vulkan-tutorial.com/Uniform_buffers/Descriptor_layout_and_buffer) and start being able to handle global variables in the Rust side of things.. All of the code can be found at [this repo](https://github.com/LuuBluum/Learning-Vulkan-with-Rust).

<!--more-->

Starting off, we're going to need to make more use of our linear algebra details. Namely, we're going to need a model-view-projection matrix. First, we're going to update our shaders as specified in the tutorial:

```c
#version 450

layout(binding = 0) uniform UniformBufferObject {
    mat4 model;
    mat4 view;
    mat4 proj;
} ubo;

layout(location = 0) in vec2 inPosition;
layout(location = 1) in vec3 inColor;

layout(location = 0) out vec3 fragColor;

void main() {
    gl_Position = ubo.proj * ubo.view * ubo.model * vec4(inPosition, 0.0, 1.0);
    fragColor = inColor;
}
```

This gives us what we want, at least in the shader side of things. After recompiling our shader, we go back to working on the Rust side of things. First, we need to have this representation in our Rust code:

```rust
#[repr(C)]
struct UniformBufferObject {
    model: glam::Mat4,
    view: glam::Mat4,
    proj: glam::Mat4,
}
```

We structure it as it would for C because this is going to be sent over to our shader and we want the formatting to match. We have our model, view, and projection matrices contained within. Now we're going to need a new function for initializing our descriptor set layout:

```rust

    fn create_descriptor_set_layout(device: &ash::Device) -> vk::DescriptorSetLayout {
        let ubo_layout_binding = vk::DescriptorSetLayoutBinding {
            binding: 0,
            descriptor_type: vk::DescriptorType::UNIFORM_BUFFER,
            descriptor_count: 1,
            stage_flags: vk::ShaderStageFlags::VERTEX,
            p_immutable_samplers: ptr::null(),
        };

        let layout_info = vk::DescriptorSetLayoutCreateInfo {
            s_type: vk::StructureType::DESCRIPTOR_SET_LAYOUT_CREATE_INFO,
            binding_count: 1,
            p_bindings: &ubo_layout_binding,
            ..Default::default()
        };

        unsafe {
            device
                .create_descriptor_set_layout(&layout_info, None)
                .unwrap()
        }
    }
```

Nothing particularly revolutionary. We create the necessary binding details (it's a uniform buffer with one descriptor in our vetex stage without any immutable samplers), feed that into our create info, and then create. This is now used in our pipeline:

```rust
        let pipeline_layout_info = vk::PipelineLayoutCreateInfo {
            s_type: vk::StructureType::PIPELINE_LAYOUT_CREATE_INFO,
            set_layout_count: 1,
            p_set_layouts: layout,
            push_constant_range_count: 0,
            p_push_constant_ranges: ptr::null(),
            ..Default::default()
        };
```

Throw it into our cleanup function and we're done with that. Now we need to actually create the layouts. More values in our Vulkan details struct! Another creation function to implement!

```rust
    fn create_uniform_buffers(
        instance: &ash::Instance,
        physical_device: &vk::PhysicalDevice,
        device: &ash::Device,
    ) -> (Vec<vk::Buffer>, Vec<vk::DeviceMemory>) {
        let buffer_size = std::mem::size_of::<UniformBufferObject>() as vk::DeviceSize;

        let mut uniform_buffers = Vec::new();
        let mut uniform_buffers_memory = Vec::new();

        uniform_buffers.reserve(MAX_FRAMES_IN_FLIGHT);
        uniform_buffers_memory.reserve(MAX_FRAMES_IN_FLIGHT);

        for _ in 0..MAX_FRAMES_IN_FLIGHT {
            let (uniform_buffer, uniform_buffer_memory) = VulkanDetails::create_buffer(
                instance,
                physical_device,
                device,
                buffer_size,
                vk::BufferUsageFlags::UNIFORM_BUFFER,
                vk::MemoryPropertyFlags::HOST_VISIBLE | vk::MemoryPropertyFlags::HOST_COHERENT,
            );
            uniform_buffers.push(uniform_buffer);
            uniform_buffers_memory.push(uniform_buffer_memory);
        }
        (uniform_buffers, uniform_buffers_memory)
    }
```

With that out of the way, now we need to actually update our uniform buffer during drawing. Otherwise all of this will have been pointless, for our uniform buffer would be static. The update function is... well, it's just the math explained in the tutorial, really, with adding the start time as a set-once internal variable initialized to the unix epoch so that we set it once on the first call and never set it again:

```rust
    fn create_uniform_buffers(
        instance: &ash::Instance,
        physical_device: &vk::PhysicalDevice,
        device: &ash::Device,
    ) -> (Vec<vk::Buffer>, Vec<vk::DeviceMemory>) {
        let buffer_size = std::mem::size_of::<UniformBufferObject>() as vk::DeviceSize;

        let mut uniform_buffers = Vec::new();
        let mut uniform_buffers_memory = Vec::new();

        uniform_buffers.reserve(MAX_FRAMES_IN_FLIGHT);
        uniform_buffers_memory.reserve(MAX_FRAMES_IN_FLIGHT);

        for _ in 0..MAX_FRAMES_IN_FLIGHT {
            let (uniform_buffer, uniform_buffer_memory) = VulkanDetails::create_buffer(
                instance,
                physical_device,
                device,
                buffer_size,
                vk::BufferUsageFlags::UNIFORM_BUFFER,
                vk::MemoryPropertyFlags::HOST_VISIBLE | vk::MemoryPropertyFlags::HOST_COHERENT,
            );
            uniform_buffers.push(uniform_buffer);
            uniform_buffers_memory.push(uniform_buffer_memory);
        }
        (uniform_buffers, uniform_buffers_memory)
    }
```

All of this should work, right?

Well, no. Running it now and we get a ton of errors vomited at us from the validation layers and nothing draws. Not great. Seems to be complaining about a lack of descriptor sets...

## The Descriptor Pool

So, it seems we need to push into [descriptor pools and descriptor sets](https://vulkan-tutorial.com/en/Uniform_buffers/Descriptor_pool_and_sets) before we get this to actually function. So, on we proceed. Now we need to configure our descriptor pool, starting with its creation:

```rust
    fn create_descriptor_pool(device: &ash::Device) -> vk::DescriptorPool {
        let pool_size = vk::DescriptorPoolSize {
            ty: vk::DescriptorType::UNIFORM_BUFFER,
            descriptor_count: MAX_FRAMES_IN_FLIGHT as u32,
        };

        let pool_info = vk::DescriptorPoolCreateInfo {
            s_type: vk::StructureType::DESCRIPTOR_POOL_CREATE_INFO,
            pool_size_count: 1,
            p_pool_sizes: &pool_size,
            max_sets: MAX_FRAMES_IN_FLIGHT as u32,
            ..Default::default()
        };

        unsafe { device.create_descriptor_pool(&pool_info, None).unwrap() }
    }
```

Nothing special about this. The allocation function for our descriptor sets is not much fancier:

```rust
    fn create_descriptor_sets(
        device: &ash::Device,
        descriptor_set_layout: &vk::DescriptorSetLayout,
        descriptor_pool: &vk::DescriptorPool,
    ) -> Vec<vk::DescriptorSet> {
        let layouts = vec![*descriptor_set_layout; MAX_FRAMES_IN_FLIGHT];
        let alloc_info = vk::DescriptorSetAllocateInfo {
            s_type: vk::StructureType::DESCRIPTOR_SET_ALLOCATE_INFO,
            descriptor_pool: *descriptor_pool,
            descriptor_set_count: MAX_FRAMES_IN_FLIGHT as u32,
            p_set_layouts: layouts.as_ptr(),
            ..Default::default()
        };

        unsafe { device.allocate_descriptor_sets(&alloc_info).unwrap() }
    }
```

We clean up the pool right before the descriptor set layout. However, this only allocates our descriptor sets. They're still unpopulated! Let's fix that:

```rust
    fn create_descriptor_sets(
        device: &ash::Device,
        uniform_buffers: &Vec<vk::Buffer>,
        descriptor_set_layout: &vk::DescriptorSetLayout,
        descriptor_pool: &vk::DescriptorPool,
    ) -> Vec<vk::DescriptorSet> {
        let layouts = vec![*descriptor_set_layout; MAX_FRAMES_IN_FLIGHT];
        let alloc_info = vk::DescriptorSetAllocateInfo {
            s_type: vk::StructureType::DESCRIPTOR_SET_LAYOUT_CREATE_INFO,
            descriptor_pool: *descriptor_pool,
            descriptor_set_count: MAX_FRAMES_IN_FLIGHT as u32,
            p_set_layouts: layouts.as_ptr(),
            ..Default::default()
        };

        let descriptor_sets = unsafe { device.allocate_descriptor_sets(&alloc_info).unwrap() };

        for i in 0..MAX_FRAMES_IN_FLIGHT {
            let buffer_info = vk::DescriptorBufferInfo {
                buffer: uniform_buffers[i],
                offset: 0,
                range: size_of::<UniformBufferObject>() as u64,
            };

            let descriptor_write = vk::WriteDescriptorSet {
                s_type: vk::StructureType::WRITE_DESCRIPTOR_SET,
                dst_set: descriptor_sets[i],
                dst_binding: 0,
                dst_array_element: 0,
                descriptor_type: vk::DescriptorType::UNIFORM_BUFFER,
                descriptor_count: 1,
                p_buffer_info: &buffer_info,
                p_image_info: ptr::null(),
                p_texel_buffer_view: ptr::null(),
                ..Default::default()
            };

            unsafe {
                device.update_descriptor_sets(
                    [descriptor_write].as_ref(),
                    &[] as &[vk::CopyDescriptorSet],
                );
            }
        }
        descriptor_sets
    }
```

Now we just need to update our command buffer details:

```rust
            self.device.cmd_bind_descriptor_sets(
                self.command_buffers[self.current_frame],
                vk::PipelineBindPoint::GRAPHICS,
                self.pipeline_layout,
                0,
                [self.descriptor_sets[self.current_frame]].as_ref(),
                &[],
            );
```

Throw that in there right above the draw command and update the rasterizer details and... we're drawing it upside-down! Whoops! Seems the GL constraints apply to this, too. Go back and multiply our projection matrix's y value by -1. Curiously enough we need to set the rasterizer back to clockwise. Now we have a spinning triangle! Or rather, rectangle, due to some details that we changed about aspect ratio. Regardless, it works! It spins! It's not baked into the shader!

Think of how far we've come with all of this!