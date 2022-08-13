---
categories: ["Rust", "Vulkan"]
title: "Learning Vulkan with Rust, Part 6: The Graphics Pipeline"
date: 2022-08-13T12:32:47-07:00
draft: false
url: ""
---

Now that we have [the swapchain](https://vulkan-tutorial.com/Drawing_a_triangle/Presentation/Swap_chain) and [image views](https://vulkan-tutorial.com/Drawing_a_triangle/Presentation/Image_views) set up, we can finally start work on something a bit more broadly relevant: the [graphics pipeline](https://vulkan-tutorial.com/Drawing_a_triangle/Graphics_pipeline_basics/Introduction). All of the code can be found at [this repo](https://github.com/LuuBluum/Learning-Vulkan-with-Rust).

<!--more-->

This will be our first venturing into the actual process of drawing something with Vulkan. How exciting! Of course this means there's a good chunk of this that isn't on Rust at all; I'll leave the GLSL discussion to the Vulkan tutorial, because nothing will change there.

In the meantime, let's set up our configuration so we can do things properly:

```rust
    fn init_vulkan(
        window: &winit::window::Window,
    ) -> (
        ash::Entry,
        ash::Instance,
        vk::DebugUtilsMessengerEXT,
        vk::SurfaceKHR,
        vk::PhysicalDevice,
        ash::Device,
        vk::Queue,
        vk::Queue,
        vk::SwapchainKHR,
        Vec<vk::Image>,
        vk::Format,
        vk::Extent2D,
        Vec<vk::ImageView>,
    ) {
        let entry = Entry::linked();
        let instance = VulkanDetails::create_instance(&entry).unwrap();
        let debug_messenger = VulkanDetails::create_debug_messenger(&entry, &instance);
        let surface = VulkanDetails::create_surface(&window, &entry, &instance).unwrap();
        let physical_device =
            VulkanDetails::pick_physical_device(&entry, &instance, &surface).unwrap();
        let device =
            VulkanDetails::create_logical_device(&entry, &instance, &physical_device, &surface);
        let (graphics_queue_index, present_queue_index) =
            VulkanDetails::find_queue_familes(&entry, &instance, &physical_device, &surface);
        let graphics_queue =
            unsafe { device.get_device_queue(graphics_queue_index.unwrap() as u32, 0) };
        let present_queue =
            unsafe { device.get_device_queue(present_queue_index.unwrap() as u32, 0) };
        let (swap_chain, swap_chain_images, swap_chain_image_format, swap_chain_extent) =
            VulkanDetails::create_swap_chain(
                window,
                &entry,
                &instance,
                &physical_device,
                &device,
                &surface,
            );
        let image_views = VulkanDetails::create_image_views(
            &device,
            &swap_chain_images,
            &swap_chain_image_format,
        );
        VulkanDetails::create_graphics_pipeline();
        (
            entry,
            instance,
            debug_messenger,
            surface,
            physical_device,
            device,
            graphics_queue,
            present_queue,
            swap_chain,
            swap_chain_images,
            swap_chain_image_format,
            swap_chain_extent,
            image_views,
        )
    }    
    fn create_graphics_pipeline() {
        
    }
```

## Shader Modules

For the [shaders](https://vulkan-tutorial.com/en/Drawing_a_triangle/Graphics_pipeline_basics/Shader_modules), that will all be the same as the tutorial, so I'll elude discussing it here. I'd mostly be repeating the tutorial anyway. We'll similarly create a nice helper function for loading our shaders into the program:

```rust
fn read_file(filename: &str) -> Vec<u8> {
    fs::read(filename).expect("Failed to read the file!")
}
```

Yes, it's very impressive. Honestly this is so trivial to do we don't even need this function. Anyway, lets fetch our shader code for creating our pipeline:

```rust
    fn create_graphics_pipeline() {
        let vert_shader_code = fs::read("shaders/vert.spv").unwrap();
        let frag_shader_code = fs::read("shaders/frag.spv").unwrap();
    }
```

Then we need to create the actual shader modules out of our code:

```rust
    fn create_shader_module(device: &ash::Device, code: Vec<u8>) -> vk::ShaderModule {
        let create_info = vk::ShaderModuleCreateInfo {
            s_type: vk::StructureType::SHADER_MODULE_CREATE_INFO,
            code_size: code.len(),
            p_code: code.as_ptr() as *const u32,
            ..Default::default()
        };
        unsafe { device.create_shader_module(&create_info, None).unwrap() }
    }
```

As we've seen many times already, the pattern is the same: create a `create_info` object for our specific construct, and then queue up a `create` function off of the specific necessary context (such as in this case, our logical device). And of course, we insert these calls into our graphics pipeline creation, where we conveniently at the end of the creation destroy the shader modules:

```rust
    fn create_graphics_pipeline(device: &ash::Device) {
        let vert_shader_code = fs::read("shaders/vert.spv").unwrap();
        let frag_shader_code = fs::read("shaders/frag.spv").unwrap();

        let vert_shader_module = VulkanDetails::create_shader_module(device, vert_shader_code);
        let frag_shader_module = VulkanDetails::create_shader_module(device, frag_shader_code);

        unsafe {
            device.destroy_shader_module(frag_shader_module, None);
            device.destroy_shader_module(vert_shader_module, None);
        }
    }
```

Don't worry! We'll make use of those modules at some point in that chunk of code, so it won't be for naught. One last step is setting up our stages:

```rust
    fn create_graphics_pipeline(device: &ash::Device) {
        let vert_shader_code = fs::read("shaders/vert.spv").unwrap();
        let frag_shader_code = fs::read("shaders/frag.spv").unwrap();

        let vert_shader_module = VulkanDetails::create_shader_module(device, vert_shader_code);
        let frag_shader_module = VulkanDetails::create_shader_module(device, frag_shader_code);

        let vert_shader_stage_info = vk::PipelineShaderStageCreateInfo {
            s_type: vk::StructureType::PIPELINE_SHADER_STAGE_CREATE_INFO,
            stage: vk::ShaderStageFlags::VERTEX,
            module: vert_shader_module,
            p_name: CStr::from_bytes_with_nul("main\0".as_bytes())
                .unwrap()
                .as_ptr(),
            ..Default::default()
        };

        let frag_shader_stage_info = vk::PipelineShaderStageCreateInfo {
            s_type: vk::StructureType::PIPELINE_SHADER_STAGE_CREATE_INFO,
            stage: vk::ShaderStageFlags::FRAGMENT,
            module: frag_shader_module,
            p_name: CStr::from_bytes_with_nul("main\0".as_bytes())
                .unwrap()
                .as_ptr(),
            ..Default::default()
        };

        let shader_stages = vec![vert_shader_stage_info, frag_shader_stage_info];

        unsafe {
            device.destroy_shader_module(frag_shader_module, None);
            device.destroy_shader_module(vert_shader_module, None);
        }
    }
```

That gives us step one: our shader code is now compiled and configured properly. Now on to stage two.

## Fixed Functions

Now that we have our shaders set up, we can move on to the [fixed functions](https://vulkan-tutorial.com/en/Drawing_a_triangle/Graphics_pipeline_basics/Fixed_functions). First, we configure our dynamic state:

```rust
        let dynamic_states = vec![vk::DynamicState::VIEWPORT, vk::DynamicState::SCISSOR];

        let dynamic_state = vk::PipelineDynamicStateCreateInfo {
            s_type: vk::StructureType::PIPELINE_DYNAMIC_STATE_CREATE_INFO,
            dynamic_state_count: dynamic_states.len() as u32,
            p_dynamic_states: dynamic_states.as_ptr(),
            ..Default::default()
        };
```

Then we move on to our vertex input:

```rust
        let vertex_input_info = vk::PipelineVertexInputStateCreateInfo {
            s_type: vk::StructureType::PIPELINE_VERTEX_INPUT_STATE_CREATE_INFO,
            vertex_binding_description_count: 0,
            p_vertex_binding_descriptions: null(),
            vertex_attribute_description_count: 0,
            p_vertex_attribute_descriptions: null(),
            ..Default::default()
        };
```

Nothing particularly special, and basically identical to the C++. Then the input assembly:

```rust
        let input_assembly = vk::PipelineInputAssemblyStateCreateInfo {
            s_type: vk::StructureType::PIPELINE_INPUT_ASSEMBLY_STATE_CREATE_INFO,
            topology: vk::PrimitiveTopology::TRIANGLE_LIST,
            primitive_restart_enable: vk::FALSE,
            ..Default::default()
        };
```

Because we're not bothering with static creation of our viewport and scissor, we can just defer handling them until during drawing. So, we just move right on to the viewport state:

```rust
        let viewport_state = vk::PipelineViewportStateCreateInfo {
            s_type: vk::StructureType::PIPELINE_VIEWPORT_STATE_CREATE_INFO,
            viewport_count: 1,
            scissor_count: 1,
            ..Default::default()
        };
```
The next meaty `create_info` is the rasterizer:

```rust
        let rasterizer = vk::PipelineRasterizationStateCreateInfo {
            s_type: vk::StructureType::PIPELINE_RASTERIZATION_STATE_CREATE_INFO,
            depth_clamp_enable: vk::FALSE,
            rasterizer_discard_enable: vk::FALSE,
            polygon_mode: vk::PolygonMode::FILL,
            line_width: 1.0,
            cull_mode: vk::CullModeFlags::BACK,
            front_face: vk::FrontFace::CLOCKWISE,
            depth_bias_enable: vk::FALSE,
            depth_bias_constant_factor: 0.0,
            depth_bias_clamp: 0.0,
            depth_bias_slope_factor: 0.0,
            ..Default::default()
        };
```

And then multisampling, which we leave disabled for now:

```rust
        let multisampling = vk::PipelineMultisampleStateCreateInfo {
            s_type: vk::StructureType::PIPELINE_MULTISAMPLE_STATE_CREATE_INFO,
            sample_shading_enable: vk::FALSE,
            rasterization_samples: vk::SampleCountFlags::TYPE_1,
            min_sample_shading: 1.0,
            p_sample_mask: null(),
            alpha_to_coverage_enable: vk::FALSE,
            alpha_to_one_enable: vk::FALSE,
            ..Default::default()
        };
```

We then set up our color blending, which we will also leave disabled for now:

```rust
        let color_blend_attachment = vk::PipelineColorBlendAttachmentState {
            color_write_mask: vk::ColorComponentFlags::R
                | vk::ColorComponentFlags::G
                | vk::ColorComponentFlags::B
                | vk::ColorComponentFlags::A,
            blend_enable: vk::FALSE,
            src_color_blend_factor: vk::BlendFactor::ONE,
            dst_color_blend_factor: vk::BlendFactor::ZERO,
            color_blend_op: vk::BlendOp::ADD,
            src_alpha_blend_factor: vk::BlendFactor::ONE,
            dst_alpha_blend_factor: vk::BlendFactor::ZERO,
            alpha_blend_op: vk::BlendOp::ADD,
        };
```

Then the global color blending, which we just defer to the framebuffer-specific color blending:

```rust
        let color_blending = vk::PipelineColorBlendStateCreateInfo {
            s_type: vk::StructureType::PIPELINE_COLOR_BLEND_STATE_CREATE_INFO,
            logic_op_enable: vk::FALSE,
            logic_op: vk::LogicOp::COPY,
            attachment_count: 1,
            p_attachments: &color_blend_attachment,
            blend_constants: [0.0, 0.0, 0.0, 0.0],
            ..Default::default()
        };
```

And with all of that we can finally make our pipeline... layout!

```rust
        let pipeline_layout_info = vk::PipelineLayoutCreateInfo {
            s_type: vk::StructureType::PIPELINE_LAYOUT_CREATE_INFO,
            set_layout_count: 0,
            p_set_layouts: null(),
            push_constant_range_count: 0,
            p_push_constant_ranges: null(),
            ..Default::default()
        };

        unsafe {
            device.destroy_shader_module(frag_shader_module, None);
            device.destroy_shader_module(vert_shader_module, None);

            device
                .create_pipeline_layout(&pipeline_layout_info, None)
                .unwrap()
        }
```

## Render Pass

And of course during all of that we still have half a dozen structs we made and then never used. Why is that? Because we still need to set up the [render pass](https://vulkan-tutorial.com/en/Drawing_a_triangle/Graphics_pipeline_basics/Render_passes). This means a new function, which we first fill with our details on the color attachment:

```rust
    fn create_render_pass(swap_chain_image_format: &vk::Format) {
        let color_attachment = vk::AttachmentDescription {
            format: *swap_chain_image_format,
            samples: vk::SampleCountFlags::TYPE_1,
            load_op: vk::AttachmentLoadOp::CLEAR,
            store_op: vk::AttachmentStoreOp::STORE,
            stencil_load_op: vk::AttachmentLoadOp::DONT_CARE,
            stencil_store_op: vk::AttachmentStoreOp::DONT_CARE,
            initial_layout: vk::ImageLayout::UNDEFINED,
            final_layout: vk::ImageLayout::PRESENT_SRC_KHR,
            ..Default::default()
        };
    }
```

Then we set up our subpass:

```rust
        let color_attachment_ref = vk::AttachmentReference {
            attachment: 0,
            layout: vk::ImageLayout::COLOR_ATTACHMENT_OPTIMAL,
        };

        let subpass = vk::SubpassDescription {
            pipeline_bind_point: vk::PipelineBindPoint::GRAPHICS,
            color_attachment_count: 1,
            p_color_attachments: &color_attachment_ref,
            ..Default::default()
        };
```

With that all taken care of, we can create our render pass:

```rust
    fn create_render_pass(
        device: &ash::Device,
        swap_chain_image_format: &vk::Format,
    ) -> vk::RenderPass {
        let color_attachment = vk::AttachmentDescription {
            format: *swap_chain_image_format,
            samples: vk::SampleCountFlags::TYPE_1,
            load_op: vk::AttachmentLoadOp::CLEAR,
            store_op: vk::AttachmentStoreOp::STORE,
            stencil_load_op: vk::AttachmentLoadOp::DONT_CARE,
            stencil_store_op: vk::AttachmentStoreOp::DONT_CARE,
            initial_layout: vk::ImageLayout::UNDEFINED,
            final_layout: vk::ImageLayout::PRESENT_SRC_KHR,
            ..Default::default()
        };

        let color_attachment_ref = vk::AttachmentReference {
            attachment: 0,
            layout: vk::ImageLayout::COLOR_ATTACHMENT_OPTIMAL,
        };

        let subpass = vk::SubpassDescription {
            pipeline_bind_point: vk::PipelineBindPoint::GRAPHICS,
            color_attachment_count: 1,
            p_color_attachments: &color_attachment_ref,
            ..Default::default()
        };

        let render_pass_info = vk::RenderPassCreateInfo {
            s_type: vk::StructureType::RENDER_PASS_CREATE_INFO,
            attachment_count: 1,
            p_attachments: &color_attachment,
            subpass_count: 1,
            p_subpasses: &subpass,
            ..Default::default()
        };
        unsafe { device.create_render_pass(&render_pass_info, None).unwrap() }
    }
```

With the render pass taken care of, the goal is on the horizon. We're almost there!

## The Graphics Pipeline

All of those unused structs at the moment can now be used for the creation of the [graphics pipeline](https://vulkan-tutorial.com/en/Drawing_a_triangle/Graphics_pipeline_basics/Conclusion). The net result looks like this:

```rust
    fn create_graphics_pipeline(
        device: &ash::Device,
        render_pass: &vk::RenderPass,
    ) -> (vk::PipelineLayout, vk::Pipeline) {
        let vert_shader_code = fs::read("shaders/vert.spv").unwrap();
        let frag_shader_code = fs::read("shaders/frag.spv").unwrap();

        let vert_shader_module = VulkanDetails::create_shader_module(device, vert_shader_code);
        let frag_shader_module = VulkanDetails::create_shader_module(device, frag_shader_code);

        let vert_shader_stage_info = vk::PipelineShaderStageCreateInfo {
            s_type: vk::StructureType::PIPELINE_SHADER_STAGE_CREATE_INFO,
            stage: vk::ShaderStageFlags::VERTEX,
            module: vert_shader_module,
            p_name: CStr::from_bytes_with_nul("main\0".as_bytes())
                .unwrap()
                .as_ptr(),
            ..Default::default()
        };

        let frag_shader_stage_info = vk::PipelineShaderStageCreateInfo {
            s_type: vk::StructureType::PIPELINE_SHADER_STAGE_CREATE_INFO,
            stage: vk::ShaderStageFlags::FRAGMENT,
            module: frag_shader_module,
            p_name: CStr::from_bytes_with_nul("main\0".as_bytes())
                .unwrap()
                .as_ptr(),
            ..Default::default()
        };

        let shader_stages = vec![vert_shader_stage_info, frag_shader_stage_info];

        let vertex_input_info = vk::PipelineVertexInputStateCreateInfo {
            s_type: vk::StructureType::PIPELINE_VERTEX_INPUT_STATE_CREATE_INFO,
            vertex_binding_description_count: 0,
            p_vertex_binding_descriptions: null(),
            vertex_attribute_description_count: 0,
            p_vertex_attribute_descriptions: null(),
            ..Default::default()
        };

        let input_assembly = vk::PipelineInputAssemblyStateCreateInfo {
            s_type: vk::StructureType::PIPELINE_INPUT_ASSEMBLY_STATE_CREATE_INFO,
            topology: vk::PrimitiveTopology::TRIANGLE_LIST,
            primitive_restart_enable: vk::FALSE,
            ..Default::default()
        };

        let viewport_state = vk::PipelineViewportStateCreateInfo {
            s_type: vk::StructureType::PIPELINE_VIEWPORT_STATE_CREATE_INFO,
            viewport_count: 1,
            scissor_count: 1,
            ..Default::default()
        };

        let rasterizer = vk::PipelineRasterizationStateCreateInfo {
            s_type: vk::StructureType::PIPELINE_RASTERIZATION_STATE_CREATE_INFO,
            depth_clamp_enable: vk::FALSE,
            rasterizer_discard_enable: vk::FALSE,
            polygon_mode: vk::PolygonMode::FILL,
            line_width: 1.0,
            cull_mode: vk::CullModeFlags::BACK,
            front_face: vk::FrontFace::CLOCKWISE,
            depth_bias_enable: vk::FALSE,
            depth_bias_constant_factor: 0.0,
            depth_bias_clamp: 0.0,
            depth_bias_slope_factor: 0.0,
            ..Default::default()
        };

        let multisampling = vk::PipelineMultisampleStateCreateInfo {
            s_type: vk::StructureType::PIPELINE_MULTISAMPLE_STATE_CREATE_INFO,
            sample_shading_enable: vk::FALSE,
            rasterization_samples: vk::SampleCountFlags::TYPE_1,
            min_sample_shading: 1.0,
            p_sample_mask: null(),
            alpha_to_coverage_enable: vk::FALSE,
            alpha_to_one_enable: vk::FALSE,
            ..Default::default()
        };

        let color_blend_attachment = vk::PipelineColorBlendAttachmentState {
            color_write_mask: vk::ColorComponentFlags::R
                | vk::ColorComponentFlags::G
                | vk::ColorComponentFlags::B
                | vk::ColorComponentFlags::A,
            blend_enable: vk::FALSE,
            src_color_blend_factor: vk::BlendFactor::ONE,
            dst_color_blend_factor: vk::BlendFactor::ZERO,
            color_blend_op: vk::BlendOp::ADD,
            src_alpha_blend_factor: vk::BlendFactor::ONE,
            dst_alpha_blend_factor: vk::BlendFactor::ZERO,
            alpha_blend_op: vk::BlendOp::ADD,
        };

        let color_blending = vk::PipelineColorBlendStateCreateInfo {
            s_type: vk::StructureType::PIPELINE_COLOR_BLEND_STATE_CREATE_INFO,
            logic_op_enable: vk::FALSE,
            logic_op: vk::LogicOp::COPY,
            attachment_count: 1,
            p_attachments: &color_blend_attachment,
            blend_constants: [0.0, 0.0, 0.0, 0.0],
            ..Default::default()
        };

        let dynamic_states = vec![vk::DynamicState::VIEWPORT, vk::DynamicState::SCISSOR];

        let dynamic_state = vk::PipelineDynamicStateCreateInfo {
            s_type: vk::StructureType::PIPELINE_DYNAMIC_STATE_CREATE_INFO,
            dynamic_state_count: dynamic_states.len() as u32,
            p_dynamic_states: dynamic_states.as_ptr(),
            ..Default::default()
        };

        let pipeline_layout_info = vk::PipelineLayoutCreateInfo {
            s_type: vk::StructureType::PIPELINE_LAYOUT_CREATE_INFO,
            set_layout_count: 0,
            p_set_layouts: null(),
            push_constant_range_count: 0,
            p_push_constant_ranges: null(),
            ..Default::default()
        };

        let pipeline_layout = unsafe {
            device
                .create_pipeline_layout(&pipeline_layout_info, None)
                .unwrap()
        };

        let pipeline_info = vk::GraphicsPipelineCreateInfo {
            s_type: vk::StructureType::GRAPHICS_PIPELINE_CREATE_INFO,
            stage_count: 2,
            p_stages: shader_stages.as_ptr(),
            p_vertex_input_state: &vertex_input_info,
            p_input_assembly_state: &input_assembly,
            p_viewport_state: &viewport_state,
            p_rasterization_state: &rasterizer,
            p_multisample_state: &multisampling,
            p_depth_stencil_state: null(),
            p_color_blend_state: &color_blending,
            p_dynamic_state: &dynamic_state,
            layout: pipeline_layout,
            render_pass: *render_pass,
            subpass: 0,
            base_pipeline_handle: vk::Pipeline::null(),
            base_pipeline_index: -1,
            ..Default::default()
        };

        let graphics_pipeline = unsafe {
            device
                .create_graphics_pipelines(vk::PipelineCache::null(), &[pipeline_info], None)
                .unwrap()[0]
        };

        unsafe {
            device.destroy_shader_module(frag_shader_module, None);
            device.destroy_shader_module(vert_shader_module, None);
        }
        (pipeline_layout, graphics_pipeline)
    }
```

It runs, everything plays nice, nothing starts screaming other than the sheer volume of validation layers confirming their existence. We have a graphics pipeline! Next, we will start arranging things so that we can actually make use of it.