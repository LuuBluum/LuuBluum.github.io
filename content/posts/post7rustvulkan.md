---
categories: ["Rust", "Vulkan"]
title: "Learning Vulkan with Rust, Part 7: Drawing"
date: 2022-08-14T21:20:32-07:00
draft: false
url: ""
---

The moment of truth is here! After handling the [graphics pipeline](https://vulkan-tutorial.com/Drawing_a_triangle/Graphics_pipeline_basics/Introduction), we're on our way to [drawing our first triangle](https://vulkan-tutorial.com/Drawing_a_triangle/Drawing/Framebuffers)! All of the code can be found at [this repo](https://github.com/LuuBluum/Learning-Vulkan-with-Rust).

<!--more-->

This is the big moment. By the end of all of this, we will have an actual triangle appear. But, we still have a ways to go.

## Framebuffers

We lead with [framebuffers](https://vulkan-tutorial.com/Drawing_a_triangle/Drawing/Framebuffers), the thing that actually holds the data on the next frame to draw. So, we need to get that set up real quick:

```rust

pub struct VulkanDetails {
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
    swap_chain_frame_buffers: Vec<vk::Framebuffer>,
}
```

Our function for creating the framebuffer is a mirror of the equivalent C++:

```rust
    fn create_framebuffers(
        device: &ash::Device,
        swap_chain_image_views: &Vec<vk::ImageView>,
        swap_chain_extent: &vk::Extent2D,
        render_pass: &vk::RenderPass,
    ) -> Vec<vk::Framebuffer> {
        let mut framebuffers = Vec::new();

        for image_view in swap_chain_image_views {
            let framebuffer_info = vk::FramebufferCreateInfo {
                s_type: vk::StructureType::FRAMEBUFFER_CREATE_INFO,
                render_pass: *render_pass,
                attachment_count: 1,
                p_attachments: image_view,
                width: swap_chain_extent.width,
                height: swap_chain_extent.height,
                layers: 1,
                ..Default::default()
            };
            framebuffers
                .push(unsafe { device.create_framebuffer(&framebuffer_info, None).unwrap() });
        }
        framebuffers
    }
    fn cleanup(&mut self) {
        unsafe {
            for framebuffer in &self.swap_chain_framebuffers {
                self.device.destroy_framebuffer(*framebuffer, None);
            }
            self.device.destroy_pipeline(self.graphics_pipeline, None);
            self.device
                .destroy_pipeline_layout(self.pipeline_layout, None);
            self.device.destroy_render_pass(self.render_pass, None);
            for image_view in &self.swap_chain_image_views {
                self.device.destroy_image_view(*image_view, None);
            }
            Swapchain::new(&self.instance, &self.device).destroy_swapchain(self.swap_chain, None);
            self.device.destroy_device(None);
            DebugUtils::new(&self.entry, &self.instance)
                .destroy_debug_utils_messenger(self.debug_messenger, None);
            Surface::new(&self.entry, &self.instance).destroy_surface(self.surface, None);
            self.instance.destroy_instance(None);
        }
    }
```

Nothing really else to say about framebuffers. We create a total amount for our given number of image views and configure their size to match our swap chain, with the render pass being the one we created. Simple enough.

## Command Buffers

Now, the fun part. We're going to use [command buffers](https://vulkan-tutorial.com/en/Drawing_a_triangle/Drawing/Command_buffers) to actually draw. But first, a command pool, to hold the memory for our command buffer:

```rust
    fn create_command_pool(
        entry: &ash::Entry,
        instance: &ash::Instance,
        physical_device: &vk::PhysicalDevice,
        device: &ash::Device,
        surface: &vk::SurfaceKHR,
    ) -> vk::CommandPool {
        let (graphics_queue_family_index, _) =
            VulkanDetails::find_queue_familes(entry, instance, physical_device, surface);
        let pool_info = vk::CommandPoolCreateInfo {
            s_type: vk::StructureType::COMMAND_POOL_CREATE_INFO,
            flags: vk::CommandPoolCreateFlags::RESET_COMMAND_BUFFER,
            queue_family_index: graphics_queue_family_index.unwrap() as u32,
            ..Default::default()
        };
        unsafe { device.create_command_pool(&pool_info, None).unwrap() }
    }
    fn cleanup(&mut self) {
        unsafe {
            self.device.destroy_command_pool(self.command_pool, None);
            for framebuffer in &self.swap_chain_framebuffers {
                self.device.destroy_framebuffer(*framebuffer, None);
            }
            self.device.destroy_pipeline(self.graphics_pipeline, None);
            self.device
                .destroy_pipeline_layout(self.pipeline_layout, None);
            self.device.destroy_render_pass(self.render_pass, None);
            for image_view in &self.swap_chain_image_views {
                self.device.destroy_image_view(*image_view, None);
            }
            Swapchain::new(&self.instance, &self.device).destroy_swapchain(self.swap_chain, None);
            self.device.destroy_device(None);
            DebugUtils::new(&self.entry, &self.instance)
                .destroy_debug_utils_messenger(self.debug_messenger, None);
            Surface::new(&self.entry, &self.instance).destroy_surface(self.surface, None);
            self.instance.destroy_instance(None);
        }
    }
```

Nothing particularly special there. Create the necessary info struct, populate it with whatever we need, and then use our logical device to create whatever we're creating. Same flow as always. The command buffer follows similarly:

```rust
    fn create_command_buffer(
        device: &ash::Device,
        command_pool: &vk::CommandPool,
    ) -> vk::CommandBuffer {
        let alloc_info = vk::CommandBufferAllocateInfo {
            s_type: vk::StructureType::COMMAND_BUFFER_ALLOCATE_INFO,
            command_pool: *command_pool,
            level: vk::CommandBufferLevel::PRIMARY,
            command_buffer_count: 1,
            ..Default::default()
        };
        unsafe { device.allocate_command_buffers(&alloc_info).unwrap()[0] }
    }
```

With that, we can actually write a drawing command. For that, again our code mirros the C++ almost identically, though rather than passing in the command buffer we just grab a reference to self (why they don't do this in the C++ code is beyond me):

```rust
    fn record_command_buffer(&self, image_index: usize) {
        let begin_info = vk::CommandBufferBeginInfo {
            s_type: vk::StructureType::COMMAND_BUFFER_BEGIN_INFO,
            ..Default::default()
        };
        unsafe {
            self.device
                .begin_command_buffer(self.command_buffer, &begin_info)
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
                self.command_buffer,
                &render_pass_info,
                vk::SubpassContents::INLINE,
            );
            self.device.cmd_bind_pipeline(
                self.command_buffer,
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
                .cmd_set_viewport(self.command_buffer, 0, &[viewport]);
        }
        let scissor = vk::Rect2D {
            offset: vk::Offset2D { x: 0, y: 0 },
            extent: self.swap_chain_extent,
        };
        unsafe {
            self.device
                .cmd_set_scissor(self.command_buffer, 0, &[scissor]);
            self.device.cmd_draw(self.command_buffer, 3, 1, 0, 0);
            self.device.cmd_end_render_pass(self.command_buffer);
            self.device.end_command_buffer(self.command_buffer).unwrap();
        }
    }
```

That gives us drawing. Now the actual drawing loop.

## Rendering

Now we get to the meat of things. Our draw command was simple enough, but now we actually have to set up how we're going to draw. First, we modify `run`:

```rust
    pub fn run(mut self) -> ! {
        self.event_loop.run(move |event, _, control_flow| {
            *control_flow = ControlFlow::Poll;

            match event {
                Event::WindowEvent {
                    event: WindowEvent::CloseRequested,
                    window_id,
                } if window_id == self.window.id() => {
                    self.vulkan_details.cleanup();
                    *control_flow = ControlFlow::Exit
                }
                _ => {
                    self.vulkan_details.draw_frame();
                }
            }
        });
    }
```

Then we need to... actually implement drawing a frame, which means dealing with synchronization. For which we get to coordinate with Vulkan synchronization objects to do. Joy. That means more to our struct:

```rust

pub struct VulkanDetails {
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
    command_buffer: vk::CommandBuffer,
    image_available_semaphore: vk::Semaphore,
    render_finished_semaphore: vk::Semaphore,
    in_flight_fence: vk::Fence,
}
```

Our struct is looking rather sizable at this point. The creation of these objects requires yet more structure creation, though in this case they're all empty:

```rust
    fn create_sync_objects(device: &ash::Device) -> (vk::Semaphore, vk::Semaphore, vk::Fence) {
        let semaphore_info = vk::SemaphoreCreateInfo {
            s_type: vk::StructureType::SEMAPHORE_CREATE_INFO,
            ..Default::default()
        };
        let fence_info = vk::FenceCreateInfo {
            s_type: vk::StructureType::FENCE_CREATE_INFO,
            flags: vk::FenceCreateFlags::SIGNALED,
            ..Default::default()
        };
        unsafe {
            (
                device.create_semaphore(&semaphore_info, None).unwrap(),
                device.create_semaphore(&semaphore_info, None).unwrap(),
                device.create_fence(&fence_info, None).unwrap(),
            )
        }
    }
```

How exciting. These get destroyed at the end like everything else:

```rust
    fn cleanup(&mut self) {
        unsafe {
            self.device
                .destroy_semaphore(self.image_available_semaphore, None);
            self.device
                .destroy_semaphore(self.render_finished_semaphore, None);
            self.device.destroy_fence(self.in_flight_fence, None);
            self.device.destroy_command_pool(self.command_pool, None);
            for framebuffer in &self.swap_chain_framebuffers {
                self.device.destroy_framebuffer(*framebuffer, None);
            }
            self.device.destroy_pipeline(self.graphics_pipeline, None);
            self.device
                .destroy_pipeline_layout(self.pipeline_layout, None);
            self.device.destroy_render_pass(self.render_pass, None);
            for image_view in &self.swap_chain_image_views {
                self.device.destroy_image_view(*image_view, None);
            }
            Swapchain::new(&self.instance, &self.device).destroy_swapchain(self.swap_chain, None);
            self.device.destroy_device(None);
            DebugUtils::new(&self.entry, &self.instance)
                .destroy_debug_utils_messenger(self.debug_messenger, None);
            Surface::new(&self.entry, &self.instance).destroy_surface(self.surface, None);
            self.instance.destroy_instance(None);
        }
    }
```

Now, on to drawing a frame. This also mirrors the C++ (since we're really just deferring to Vulkan for everything at this point). Honestly it's rather remarkable just how much of this is the same across different systems. I guess that goes to show just how much we're deferring to Vulkan for everything! Just like the C++, we need to update our `create_render_pass` function:

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

        let dependency = vk::SubpassDependency {
            src_subpass: vk::SUBPASS_EXTERNAL,
            dst_subpass: 0,
            src_stage_mask: vk::PipelineStageFlags::COLOR_ATTACHMENT_OUTPUT,
            src_access_mask: vk::AccessFlags::empty(),
            dst_stage_mask: vk::PipelineStageFlags::COLOR_ATTACHMENT_OUTPUT,
            dst_access_mask: vk::AccessFlags::COLOR_ATTACHMENT_WRITE,
            ..Default::default()
        };

        let render_pass_info = vk::RenderPassCreateInfo {
            s_type: vk::StructureType::RENDER_PASS_CREATE_INFO,
            attachment_count: 1,
            p_attachments: &color_attachment,
            subpass_count: 1,
            p_subpasses: &subpass,
            dependency_count: 1,
            p_dependencies: &dependency,
            ..Default::default()
        };
        unsafe { device.create_render_pass(&render_pass_info, None).unwrap() }
    }
```

Then we have our function for drawing a frame:

```rust
    fn draw_frame(&self) {
        unsafe {
            self.device
                .wait_for_fences(&[self.in_flight_fence], true, u64::MAX)
                .unwrap();
            self.device.reset_fences(&[self.in_flight_fence]).unwrap();
            let swap_chain_handle = Swapchain::new(&self.instance, &self.device);
            let (image_index, _) = swap_chain_handle
                .acquire_next_image(
                    self.swap_chain,
                    u64::MAX,
                    self.image_available_semaphore,
                    vk::Fence::null(),
                )
                .unwrap();
            self.device
                .reset_command_buffer(self.command_buffer, vk::CommandBufferResetFlags::empty())
                .unwrap();
            self.record_command_buffer(image_index as usize);
            let submit_info = vk::SubmitInfo {
                s_type: vk::StructureType::SUBMIT_INFO,
                wait_semaphore_count: 1,
                p_wait_semaphores: [self.image_available_semaphore].as_ptr(),
                p_wait_dst_stage_mask: [vk::PipelineStageFlags::COLOR_ATTACHMENT_OUTPUT].as_ptr(),
                command_buffer_count: 1,
                p_command_buffers: [self.command_buffer].as_ptr(),
                signal_semaphore_count: 1,
                p_signal_semaphores: [self.render_finished_semaphore].as_ptr(),
                ..Default::default()
            };
            self.device
                .queue_submit(self.graphics_queue, &[submit_info], self.in_flight_fence)
                .unwrap();
            let present_info = vk::PresentInfoKHR {
                s_type: vk::StructureType::PRESENT_INFO_KHR,
                wait_semaphore_count: 1,
                p_wait_semaphores: [self.render_finished_semaphore].as_ptr(),
                swapchain_count: 1,
                p_swapchains: [self.swap_chain].as_ptr(),
                p_image_indices: &image_index,
                ..Default::default()
            };
            swap_chain_handle
                .queue_present(self.present_queue, &present_info)
                .unwrap();
        }
    }
```

That leaves us with a pickle: we want to wait on destroying all of our Vulkan details until they're not in-use, but our current approach doesn't wait for the window to be destroyed. We can adjust that, though:

```rust
    pub fn run(mut self) -> ! {
        self.event_loop.run(move |event, _, control_flow| {
            *control_flow = ControlFlow::Poll;

            match event {
                Event::WindowEvent {
                    event: WindowEvent::CloseRequested,
                    window_id,
                } if window_id == self.window.id() => *control_flow = ControlFlow::Exit,
                Event::LoopDestroyed => {
                    unsafe { self.vulkan_details.device.device_wait_idle().unwrap() };
                    self.vulkan_details.cleanup();
                }
                _ => {
                    self.vulkan_details.draw_frame();
                }
            }
        });
    }
```

Now we actually handle our cleanup on the actual destruction of the event loop, which happens after the window closes. Much better. If we put in that wait where it was before, it would never actually let the window close. We're not done yet, though!

## Handling Multiple Frames

Right now we just leave one [frame in flight](https://vulkan-tutorial.com/en/Drawing_a_triangle/Drawing/Frames_in_flight), which is poor performance-wise because the CPU idles while the GPU renders causing everybody to wait on each other and generally slow things down. If we have multiple frames moving between the CPU and GPU at once, then this issue goes away. Except each frame is going to need its own command buffer and synchronization objects.

Which means everything is vectors now.

Joy.

```rust
pub struct VulkanDetails {
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
    command_buffers: Vec<vk::CommandBuffer>,
    image_available_semaphores: Vec<vk::Semaphore>,
    render_finished_semaphores: Vec<vk::Semaphore>,
    in_flight_fences: Vec<vk::Fence>,
}
```

Naturally this means changing our functions for creating all of these, too:

```rust
    fn create_command_buffers(
        device: &ash::Device,
        command_pool: &vk::CommandPool,
    ) -> Vec<vk::CommandBuffer> {
        let alloc_info = vk::CommandBufferAllocateInfo {
            s_type: vk::StructureType::COMMAND_BUFFER_ALLOCATE_INFO,
            command_pool: *command_pool,
            level: vk::CommandBufferLevel::PRIMARY,
            command_buffer_count: MAX_FRAMES_IN_FLIGHT as u32,
            ..Default::default()
        };
        unsafe { device.allocate_command_buffers(&alloc_info).unwrap() }
    }
    fn create_sync_objects(
        device: &ash::Device,
    ) -> (Vec<vk::Semaphore>, Vec<vk::Semaphore>, Vec<vk::Fence>) {
        let semaphore_info = vk::SemaphoreCreateInfo {
            s_type: vk::StructureType::SEMAPHORE_CREATE_INFO,
            ..Default::default()
        };
        let fence_info = vk::FenceCreateInfo {
            s_type: vk::StructureType::FENCE_CREATE_INFO,
            flags: vk::FenceCreateFlags::SIGNALED,
            ..Default::default()
        };
        let mut image_available_semaphores = Vec::new();
        let mut render_finished_semaphores = Vec::new();
        let mut in_flight_fences = Vec::new();
        unsafe {
            for _ in 0..MAX_FRAMES_IN_FLIGHT {
                image_available_semaphores
                    .push(device.create_semaphore(&semaphore_info, None).unwrap());
                render_finished_semaphores
                    .push(device.create_semaphore(&semaphore_info, None).unwrap());
                in_flight_fences.push(device.create_fence(&fence_info, None).unwrap());
            }
        }
        (
            image_available_semaphores,
            render_finished_semaphores,
            in_flight_fences,
        )
    }
```

And that means we also get to change our `draw_frame` and `record_command_buffer` functions:

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
        unsafe {
            self.device
                .cmd_set_scissor(self.command_buffers[self.current_frame], 0, &[scissor]);
            self.device
                .cmd_draw(self.command_buffers[self.current_frame], 3, 1, 0, 0);
            self.device
                .cmd_end_render_pass(self.command_buffers[self.current_frame]);
            self.device
                .end_command_buffer(self.command_buffers[self.current_frame])
                .unwrap();
        }
    }
    fn draw_frame(&mut self) {
        unsafe {
            self.device
                .wait_for_fences(&[self.in_flight_fences[self.current_frame]], true, u64::MAX)
                .unwrap();
            self.device
                .reset_fences(&[self.in_flight_fences[self.current_frame]])
                .unwrap();
            let swap_chain_handle = Swapchain::new(&self.instance, &self.device);
            let (image_index, _) = swap_chain_handle
                .acquire_next_image(
                    self.swap_chain,
                    u64::MAX,
                    self.image_available_semaphores[self.current_frame],
                    vk::Fence::null(),
                )
                .unwrap();
            self.device
                .reset_command_buffer(
                    self.command_buffers[self.current_frame],
                    vk::CommandBufferResetFlags::empty(),
                )
                .unwrap();
            self.record_command_buffer(image_index as usize);
            let submit_info = vk::SubmitInfo {
                s_type: vk::StructureType::SUBMIT_INFO,
                wait_semaphore_count: 1,
                p_wait_semaphores: [self.image_available_semaphores[self.current_frame]].as_ptr(),
                p_wait_dst_stage_mask: [vk::PipelineStageFlags::COLOR_ATTACHMENT_OUTPUT].as_ptr(),
                command_buffer_count: 1,
                p_command_buffers: [self.command_buffers[self.current_frame]].as_ptr(),
                signal_semaphore_count: 1,
                p_signal_semaphores: [self.render_finished_semaphores[self.current_frame]].as_ptr(),
                ..Default::default()
            };
            self.device
                .queue_submit(
                    self.graphics_queue,
                    &[submit_info],
                    self.in_flight_fences[self.current_frame],
                )
                .unwrap();
            let present_info = vk::PresentInfoKHR {
                s_type: vk::StructureType::PRESENT_INFO_KHR,
                wait_semaphore_count: 1,
                p_wait_semaphores: [self.render_finished_semaphores[self.current_frame]].as_ptr(),
                swapchain_count: 1,
                p_swapchains: [self.swap_chain].as_ptr(),
                p_image_indices: &image_index,
                ..Default::default()
            };
            swap_chain_handle
                .queue_present(self.present_queue, &present_info)
                .unwrap();
            self.current_frame = (self.current_frame + 1) % MAX_FRAMES_IN_FLIGHT;
        }
    }
```

And yes, we take in `self` mutably for drawing a frame all because we need to update what frame we're drawing. Such as it is.

## Recreating the Swap Chain

One last step: [recreating the swap chain](https://vulkan-tutorial.com/en/Drawing_a_triangle/Swap_chain_recreation) every time we need to, which will be for window resizing, minimization, and whatnot. The basic function structure is... a bit more chunky than the C++:

```rust
    fn recreate_swap_chain(&mut self, window: &winit::window::Window) {
        unsafe { self.device.device_wait_idle().unwrap() };

        self.cleanup_swap_chain();

        (
            self.swap_chain,
            self.swap_chain_images,
            self.swap_chain_image_format,
            self.swap_chain_extent,
        ) = VulkanDetails::create_swap_chain(
            &window,
            &self.entry,
            &self.instance,
            &self.physical_device,
            &self.device,
            &self.surface,
        );

        self.swap_chain_image_views = VulkanDetails::create_image_views(
            &self.device,
            &self.swap_chain_images,
            &self.swap_chain_image_format,
        );

        self.swap_chain_framebuffers = VulkanDetails::create_framebuffers(
            &self.device,
            &self.swap_chain_image_views,
            &self.swap_chain_extent,
            &self.render_pass,
        );
    }
```

Necessary since all of these functions were originally conceived without our struct being populated. The cleanup code for our swapchain is just our cleanup code, moved around:

```rust
    fn cleanup_swap_chain(&mut self) {
        unsafe {
            for framebuffer in &self.swap_chain_framebuffers {
                self.device.destroy_framebuffer(*framebuffer, None);
            }
            for image_view in &self.swap_chain_image_views {
                self.device.destroy_image_view(*image_view, None);
            }
            Swapchain::new(&self.instance, &self.device).destroy_swapchain(self.swap_chain, None);
        }
    }
    fn cleanup(&mut self) {
        unsafe {
            self.cleanup_swap_chain();
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

Bake these into our `draw_frame` function, dealing with some of those results that we just casually unwrapped by actually doing something properly now:

```rust
    fn draw_frame(&mut self, window: &winit::window::Window) {
        unsafe {
            self.device
                .wait_for_fences(&[self.in_flight_fences[self.current_frame]], true, u64::MAX)
                .unwrap();
            self.device
                .reset_fences(&[self.in_flight_fences[self.current_frame]])
                .unwrap();
            let swap_chain_handle = Swapchain::new(&self.instance, &self.device);
            let (image_index, _) = match swap_chain_handle.acquire_next_image(
                self.swap_chain,
                u64::MAX,
                self.image_available_semaphores[self.current_frame],
                vk::Fence::null(),
            ) {
                Ok(value) => value,
                Err(error) => match error {
                    vk::Result::ERROR_OUT_OF_DATE_KHR => {
                        self.recreate_swap_chain(window);
                        return;
                    }
                    _ => panic!("Problem with the surface!"),
                },
            };
            self.device
                .reset_command_buffer(
                    self.command_buffers[self.current_frame],
                    vk::CommandBufferResetFlags::empty(),
                )
                .unwrap();
            self.record_command_buffer(image_index as usize);
            let submit_info = vk::SubmitInfo {
                s_type: vk::StructureType::SUBMIT_INFO,
                wait_semaphore_count: 1,
                p_wait_semaphores: [self.image_available_semaphores[self.current_frame]].as_ptr(),
                p_wait_dst_stage_mask: [vk::PipelineStageFlags::COLOR_ATTACHMENT_OUTPUT].as_ptr(),
                command_buffer_count: 1,
                p_command_buffers: [self.command_buffers[self.current_frame]].as_ptr(),
                signal_semaphore_count: 1,
                p_signal_semaphores: [self.render_finished_semaphores[self.current_frame]].as_ptr(),
                ..Default::default()
            };
            self.device
                .queue_submit(
                    self.graphics_queue,
                    &[submit_info],
                    self.in_flight_fences[self.current_frame],
                )
                .unwrap();
            let present_info = vk::PresentInfoKHR {
                s_type: vk::StructureType::PRESENT_INFO_KHR,
                wait_semaphore_count: 1,
                p_wait_semaphores: [self.render_finished_semaphores[self.current_frame]].as_ptr(),
                swapchain_count: 1,
                p_swapchains: [self.swap_chain].as_ptr(),
                p_image_indices: &image_index,
                ..Default::default()
            };
            match swap_chain_handle.queue_present(self.present_queue, &present_info) {
                Ok(should_recreate) => {
                    if should_recreate {
                        self.recreate_swap_chain(window);
                    }
                }
                Err(error) => match error {
                    vk::Result::ERROR_OUT_OF_DATE_KHR => self.recreate_swap_chain(window),
                    _ => panic!("Unable to present!"),
                },
            };
            self.current_frame = (self.current_frame + 1) % MAX_FRAMES_IN_FLIGHT;
        }
    }
```

Notably, if we ran it as things are now, it will deadlock. Not particularly convenient. This has to do with our current fence approach, which we should take care of real quick:

```rust
    fn draw_frame(&mut self, window: &winit::window::Window) {
        unsafe {
            self.device
                .wait_for_fences(&[self.in_flight_fences[self.current_frame]], true, u64::MAX)
                .unwrap();
            let swap_chain_handle = Swapchain::new(&self.instance, &self.device);
            let (image_index, _) = match swap_chain_handle.acquire_next_image(
                self.swap_chain,
                u64::MAX,
                self.image_available_semaphores[self.current_frame],
                vk::Fence::null(),
            ) {
                Ok(value) => value,
                Err(error) => match error {
                    vk::Result::ERROR_OUT_OF_DATE_KHR => {
                        self.recreate_swap_chain(window);
                        return;
                    }
                    _ => panic!("Problem with the surface!"),
                },
            };
            self.device
                .reset_fences(&[self.in_flight_fences[self.current_frame]])
                .unwrap();
```

Just don't reset the fence until we're past the point where we might return early. Easy peasy. Now we have to take care of resizing, since a resize triggering `VK_ERROR_OUT_OF_DATE_KHR` isn't guaranteed. So, we make a change in our checking for recreating the swap chain:

```rust
            match swap_chain_handle.queue_present(self.present_queue, &present_info) {
                Ok(should_recreate) => {
                    if should_recreate || self.framebuffer_resized {
                        self.framebuffer_resized = false;
                        self.recreate_swap_chain(window);
                    }
                }
                Err(error) => match error {
                    vk::Result::ERROR_OUT_OF_DATE_KHR => self.recreate_swap_chain(window),
                    _ => panic!("Unable to present!"),
                },
            };
```

And we upate our `run` function:

```rust
    pub fn run(mut self) -> ! {
        self.event_loop.run(move |event, _, control_flow| {
            *control_flow = ControlFlow::Poll;

            match event {
                Event::WindowEvent {
                    event: WindowEvent::CloseRequested,
                    window_id,
                } if window_id == self.window.id() => *control_flow = ControlFlow::Exit,
                Event::WindowEvent {
                    event: WindowEvent::Resized(_),
                    window_id,
                } if window_id == self.window.id() => {
                    self.vulkan_details.framebuffer_resized = true
                }
                Event::LoopDestroyed => {
                    unsafe { self.vulkan_details.device.device_wait_idle().unwrap() };
                    self.vulkan_details.cleanup();
                }
                _ => {
                    self.vulkan_details.draw_frame(&self.window);
                }
            }
        });
    }
```

Now, all we have left is to pause whenever there's nothing to draw (such as minimizing or resizing to the point where the framebuffer is gone). However, there's a problem: we can't just stall until we get another event. This needs to be handled in the event loop, not our swap chain recreation. This is actually rather simple; we can just not draw when one of our size dimensions is zero:

```rust
    pub fn run(mut self) -> ! {
        self.event_loop.run(move |event, _, control_flow| {
            *control_flow = ControlFlow::Poll;

            match event {
                Event::WindowEvent {
                    event: WindowEvent::CloseRequested,
                    window_id,
                } if window_id == self.window.id() => *control_flow = ControlFlow::Exit,
                Event::WindowEvent {
                    event: WindowEvent::Resized(size),
                    window_id,
                } if window_id == self.window.id() => {
                    self.vulkan_details.framebuffer_resized = true;
                    if size.width > 0 && size.height > 0 {
                        self.vulkan_details.draw_frame(&self.window);
                    }
                }
                Event::LoopDestroyed => {
                    unsafe { self.vulkan_details.device.device_wait_idle().unwrap() };
                    self.vulkan_details.cleanup();
                }
                _ => {
                    if self.window.inner_size().width > 0 && self.window.inner_size().height > 0 {
                        self.vulkan_details.draw_frame(&self.window);
                    }
                }
            }
        });
    }
```

And there we are! We're drawing a triangle! It doesn't vomit errors at us if we minimize or resize things! The next steps will be quite a bit more complicated, but at least we can visually see the product of our effort now.