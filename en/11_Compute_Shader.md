## Introduction

In this bonus chapter we'll take a look at compute shaders. Up until now all previous chapters dealt with the traditional graphics part of the Vulkan pipeline. But unlike older APIs like OpenGL, compute shader support in Vulkan is mandatory. This means that you can use compute shaders on every Vulkan implementation available, no matter if it's a high-end desktop GPU or a low-powered embedded device.

This opens up the world of general purpose computing on graphics processor units (GPGPU), no matter where your application is running. GPGPU means that you can do general computations on your GPU, something that has traditionally been a domain of CPUs. But with GPUs having become more and more powerful and more flexible, many workloads that would require the general purpose capabilities of a CPU can now be done on the GPU in realtime.

A few examples of where the compute capabilities of a GPU can be used are image manipulation and physics, e.g. for a particle system. And it's even possible to use only compute for doing computational only work that does not require any graphics output, e.g. number crunching or AI related things. This is called "headless compute".

## Advantages

Doing computational expensive calculations on the GPU has several advantages. The most obvious one is offloading work from the CPU. Another one is not requiring a round-trip to the CPU and it's main memory, so all data can stay on the GPU without having to resolve to slow main memory reads.

Aside from these, GPUs are heavily parallelized with some of them having tens of thousands of small compute units. This often makes them a better fit for highly parallel workflows than a CPU with a few large units.

## The Vulkan pipeline

It's important to know that compute is completely separated from the graphics part of the pipeline. This is visible in the following block diagram of the Vulkan pipeline from the official specification:

![](/images/vulkan_pipeline_block_diagram.png)

In this diagram we can see the traditional graphics part of the pipeline on the left, and several stages on the right that are not part of this graphics pipeline, including the compute shader (stage). With the compute shader stage being detached from the graphics pipeline we'll be able to use it anywhere where we see fit. This is very different from e.g. the fragment shader which is always applied to the transformed output of the vertex shader.

## An example

An easy to understand example that we implement in this chapter is a GPU based particle particle system. Such systems are used in many games and often consist of thousands of particles that need to be updated at interactive frame rates. And sometimes even with complex physics applied, e.g. when testing for collisions. So rendering such a system requires vertices (passed as vertex buffers) and a way to update them based on some equation.

A "classical" CPU based particle system would store particles in the system's main memory and then use the CPU to update them. And after the update, the vertices need to be transferred to the GPU's memory again, so it'll display the updated particles in the next frame. The most straight-forward way would be recreating the vertex buffer with the new dta each frame. This is obviously very costly. Depending on your implementation, there are other options like mapping GPU memory so it can be written by the CPU (called "resizable BAR" on desktop systems, or unified memory on integrated GPUs) or just using a host local buffer (which would be the slowest method due to PCI-E bandwidth). But no matter what buffer update you'd choose, you always require a "round-trip" to the CPU to update the particles.

With a GPU based particle system, this round-trip is no longer required. Vertices are only uploaded to the GPU once and all updates are done in the GPU's memory by using compute shaders. One of the main reasons why this is faster is the much higher bandwidth between the GPU and it's local memory. In a CPU based scenario, you'd be limited by main memory and PCI-express bandwidth, which is often just a fraction of the GPU's memory bandwidth.

And doing this on a GPU with a dedicated compute queue, you can update particles in parallel to the rendering part of the graphics pipeline. This is called "async compute", and is an advanced topic not covered in this tutorial.

A screenshot from this chapter's code. The particles shown here are updated by a compute shader directly on the GPU, without any CPU interaction:

![](/images/compute_shader_particles.png)

## Data manipulation

In this tutorial we already learned about different buffer types like vertex and index buffer for passing primitives and uniform buffers for passing data to a shader. And we also used images do to texture mapping. But up until know we always wrote data using the CPU and only did reads on the GPU.

An important concept introduced with compute shaders is the possibility to arbitrarily read from **and write** to buffers. For this, Vulkan offers two dedicated storage types.

### Shader storage buffer objects (SSBO)

A shader storage buffer (SSBO) allows you to read from and write to a buffer. Using these is similar to using uniform buffer objects. The biggest difference is that you can alias other buffer types to SSBOs and that they can be arbitrarily large.

Going back to the GPU based particle system you might now wonder how to deal with vertices being updated (written) by the compute shader and read (drawn) by the vertex shader, as both usages would seemingly require different buffer types.

But that's not the case. In Vulkan you can specify multiple usages for buffers and images. So for the particle vertex buffer to be used as a vertex buffer (in the graphics pass) and as a storage buffer (in the compute pass) you simply create the buffer with those two usage flags:

```c++
VkBufferCreateInfo bufferInfo{};
...
bufferInfo.usage = VK_BUFFER_USAGE_VERTEX_BUFFER_BIT | VK_BUFFER_USAGE_STORAGE_BUFFER_BIT | VK_BUFFER_USAGE_TRANSFER_DST_BIT;
...

if (vkCreateBuffer(device, &bufferInfo, nullptr, &vertexBuffer) != VK_SUCCESS) {
    throw std::runtime_error("failed to create vertex buffer!");
}
```

Note that we also added the `VK_BUFFER_USAGE_TRANSFER_DST_BIT` flag in here, as we want the shader storage buffer to stay in GPU memory only which requires us to transfer data from the host to this buffer.

@todo: also show simplified code with createBuffer helper?

The GLSL shader declaration for a SSBO looks like this:

```glsl
struct Particle {
	vec2 position;
	vec2 velocity;
    vec4 color;
};

layout(std140, binding = 0) buffer ParticleSSBO {
   Particle particles[ ];
};
```

In this example we have a typed SSBO with each particle having a position and velocity value (see the `Particle` struct). The SSBO then contains an unbound number of particles as marked by the `[]`. Not having to specify the number of elements in an SSBO is one of the advantages over e.g. uniform buffers.

The two flags `VK_BUFFER_USAGE_VERTEX_BUFFER_BIT` and `VK_BUFFER_USAGE_STORAGE_BUFFER_BIT` set with `bufferInfo.usage` tell the implementation that we want to use this buffer for two different scenarios: as a vertex buffer in the vertex shader and as a store buffer.

Writing to such a storage buffer object in the compute shader is straight-forward and similar to how you'd write to the buffer on the C++ side:

```glsl
particles[index].position += deltaTime * particles[index].velocity;
```

### Storage images

A storage image allows you read from and write to an image. Typical use cases are applying image effects to textures or doing post processing (which in turn is very similar).

This is similar for images:

```c++
VkImageCreateInfo imageInfo {};
...
imageInfo.usage = VK_IMAGE_USAGE_SAMPLED_BIT | VK_IMAGE_USAGE_STORAGE_BIT;
...

if (vkCreateImage(device, &imageInfo, nullptr, &textureImage) != VK_SUCCESS) {
    throw std::runtime_error("failed to create image!");
}
```

The two flags `VK_IMAGE_USAGE_SAMPLED_BIT` and `VK_IMAGE_USAGE_STORAGE_BIT` set with `bufferInfo.usage` tell the implementation that we want to use this image for two different scenarios: as an image sampled in the fragment shader and as a storage image in the computer shader;

The GLSL shader declaration for storage image looks similar to sampled images used e.g. in the fragment shader:

```glsl
layout (binding = 0, rgba8) uniform readonly image2D inputImage;
layout (binding = 1, rgba8) uniform writeonly image2D outputImage;
```

A few differences here are additional attributes like `rgba8` for the format of the image, the `readonly` and `writeonly` qualifiers, telling the implementation that we'll only read from the input and write to the output image. And last but not least we need to use the `image2D` type do declare a storage image.

Reading from and writing to storage images in the compute shader is then done using `imageLoad` and `imageStore`: 

```glsl
vec3 pixel = imageLoad(inputImage, ivec2(gl_GlobalInvocationID.xy)).rgb;
imageStore(outputImage, ivec2(gl_GlobalInvocationID.xy), pixel);
```

## Compute queue families

In the [physical device and queue families chapter](03_Drawing_a_triangle/00_Setup/03_Physical_devices_and_queue_families.md#page_Queue-families) we already learned about queue families and how to select a graphics queue family. Compute introduces the new queue family type bit `VK_QUEUE_COMPUTE_BIT`. So if we want to do compute, we need to get a queue from a queue family type that supports compute.

Note that Vulkan requires an implementation that supports graphics operations to have at least one queue family that supports both graphics and compute, but it's also possible that implementations offer a dedicated compute queue. This dedicated compute queue (that does not have the graphics bit) hints at an asynchronous compute queue. To keep this tutorial beginner friendly though, we'll use a queue that can do both graphics and compute. This will also save us some advanced synchronization.

So for our compute sample we need to change the device creation code a bit:

```c++
uint32_t queueFamilyCount = 0;
vkGetPhysicalDeviceQueueFamilyProperties(device, &queueFamilyCount, nullptr);

std::vector<VkQueueFamilyProperties> queueFamilies(queueFamilyCount);
vkGetPhysicalDeviceQueueFamilyProperties(device, &queueFamilyCount, queueFamilies.data());

int i = 0;
for (const auto& queueFamily : queueFamilies) {
    if ((queueFamily.queueFlags & VK_QUEUE_GRAPHICS_BIT) && (queueFamily.queueFlags & VK_QUEUE_COMPUTE_BIT)) {
        indices.graphicsAndComputeFamily = i;
    }

    i++;
}
```

The changed queue family index selection code will now try to find a queue family that supports both graphics and compute. 

We can then get a compute queue from this queue family in `createLogicalDevice`:

```c++
vkGetDeviceQueue(device, indices.graphicsAndComputeFamily.value(), 0, &computeQueue);
```

## The compute shader stage

In the graphics samples we have used different pipeline stages to load shaders and access descriptors. Compute shaders are accessed in a similar way by using the `VK_SHADER_STAGE_COMPUTE_BIT` pipeline. So loading a compute shader is just the same as loading a vertex shader, but with a different shader stage. We'll talk about this in detail in the next paragraphs. Compute also introduces a new binding point type for descriptors and pipelines named `VK_PIPELINE_BIND_POINT_COMPUTE` that we'll have to use later on.

## Loading compute shaders

Loading compute shaders in our application is the same as loading any other other shader. The only real difference is that we'll need to use the `VK_SHADER_STAGE_COMPUTE_BIT` mentioned above.

```c++
auto computeShaderCode = readFile("shaders/compute.spv");

VkShaderModule computeShaderModule = createShaderModule(computeShaderCode);

VkPipelineShaderStageCreateInfo computeShaderStageInfo{};
computeShaderStageInfo.sType = VK_STRUCTURE_TYPE_PIPELINE_SHADER_STAGE_CREATE_INFO;
computeShaderStageInfo.stage = VK_SHADER_STAGE_COMPUTE_BIT;
computeShaderStageInfo.module = computeShaderModule;
computeShaderStageInfo.pName = "main";
...
```

## Descriptors

Descriptors that we want to access in a compute shader also need to have this stage flag set:

```c++
VkDescriptorSetLayoutBinding uboLayoutBinding{};
uboLayoutBinding.binding = 0;
uboLayoutBinding.descriptorType = VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER;
uboLayoutBinding.descriptorCount = 1;
uboLayoutBinding.stageFlags = VK_SHADER_STAGE_COMPUTE_BIT;
...
```

Note that you can combine shader stages here, so if you want the descriptor to be accessible from the vertex and compute stage, e.g. for a uniform buffer with parameters shared across them, you simply set the bits for both stages:

```c++
uboLayoutBinding.stageFlags = VK_SHADER_STAGE_VERTEX_BIT | VK_SHADER_STAGE_COMPUTE_BIT;
```

## Compute pipelines

As compute is not a part of the graphics pipeline, we can't use `vkCreateGraphicsPipelines` to attach the compute shader to it. Instead we need to create a dedicated compute pipeline `vkCreateComputePipelines` for running our compute commands. Since a compute pipeline does not touch any of the rasterization state, it has a lot less state than a graphics pipeline:

```c++
VkComputePipelineCreateInfo pipelineInfo{};
pipelineInfo.sType = VK_STRUCTURE_TYPE_COMPUTE_PIPELINE_CREATE_INFO;
pipelineInfo.layout = computePipelineLayout;
pipelineInfo.stage = computeShaderStageInfo;

if (vkCreateComputePipelines(device, VK_NULL_HANDLE, 1, &pipelineInfo, nullptr, &computePipeline) != VK_SUCCESS) {
    throw std::runtime_error("failed to create compute pipeline!");
}
```

The setup is a lot simpler, as we only require one shader stage and a pipeline layout. The pipeline layout works the same as with the graphics pipeline but may use storage types where required:

```c++
std::array<VkWriteDescriptorSet, 2> descriptorWrites{};

descriptorWrites[0].sType = VK_STRUCTURE_TYPE_WRITE_DESCRIPTOR_SET;
descriptorWrites[0].dstSet = computeDescriptorSets[i];
descriptorWrites[0].dstBinding = 0;
descriptorWrites[0].dstArrayElement = 0;
descriptorWrites[0].descriptorType = VK_DESCRIPTOR_TYPE_STORAGE_BUFFER;
descriptorWrites[0].descriptorCount = 1;
descriptorWrites[0].pBufferInfo = &storageBufferInfo;

descriptorWrites[1].sType = VK_STRUCTURE_TYPE_WRITE_DESCRIPTOR_SET;
descriptorWrites[1].dstSet = computeDescriptorSets[i];
descriptorWrites[1].dstBinding = 1;
descriptorWrites[1].dstArrayElement = 0;
descriptorWrites[1].descriptorType = VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER;
descriptorWrites[1].descriptorCount = 1;
descriptorWrites[1].pBufferInfo = &uniformBufferInfo;

vkUpdateDescriptorSets(device, static_cast<uint32_t>(descriptorWrites.size()), descriptorWrites.data(), 0, nullptr);
```

## Compute space

Before we get into how a compute shader works and how we submit compute workloads to the GPU, we need to talk about two important compute concepts: **work groups** and **invocations**. They define an abstract execution model for how compute workloads are processed by the compute hardware of the GPU in three dimensions (x,y and z).

**Work groups** define how the compute workloads are formed and processed by the the compute hardware of the GPU. You can think of them as work items the GPU has to work through. Work group dimensions are set by the application at command buffer time using a dispatch command.

And each work group then is a collection of **invocations** that execute the same compute shader. Invocations can potentially run in parallel and their dimensions are set in the compute shader. Invocations within a single workgroup have access to shared memory.

This image shows the relation between these two in three dimensions:

![](/images/compute_space.svg)

The number of dimensions for work groups and invocations depends on how input data is structured. If you e.g. work on a one-dimensional array, like we do in this chapter, you only have to specify the x dimension for both.

As an example: If we dispatch a work group count of [64, 1, 1] with a compute shader local size of [32, 32, ,1], our compute shader will be invoked 64 x 32 x 32 = 65,536 times.

## Compute shaders

Now that we learned about all the parts required to setup a compute shader pipeline, it's time to take a look at compute shaders. All of the things we learned about using GLSL shaders e.g. for vertex and fragment shaders also applies to compute shaders. The syntax is the same, and many concepts like passing data between the application and the shader is the same. But there are some important differences.

A very basic compute shader for updating a linear array of particles may look like this:

```glsl
#version 450

struct Particle {
	vec2 position;
	vec2 velocity;
    vec4 color;
};

layout(std140, binding = 0) buffer ParticleSSBO {
   Particle particles[ ];
};

layout (binding = 1) uniform ParameterUBO {
	float deltaTime;
} ubo;

layout (local_size_x = 256, local_size_y = 1, local_size_z = 1) in;

void main() 
{
    uint index = gl_GlobalInvocationID.x;  
    particles[index].position.xy += particles[index].velocity.xy * ubo.deltaTime;
}
```

The top part references the shader storage buffer object (SSBO) we learned about earlier and also has a uniform buffer object that contains a delta time as a parameter passed to this shader. The `main` function is also pretty simplistic, but shows how to read and write to the SSBO containing the particles.

The interesting part here, that needs some explanation is this declaration:

```glsl
layout (local_size_x = 256, local_size_y = 1, local_size_z = 1) in;
```
This defines the number invocations of this compute shader in the current work group. As noted earlier this is the local part of the compute space. Hence the `local_` prefix. As we work on a linear 1D array of particles we only need to specify a number for x dimension in `local_size_x`.

Similar to other shader types, compute shaders have their own set of builtin input variables. Built-ins are always prefixed with `gl_`. One such built-in is `gl_GlobalInvocationID`, a variable that uniquely identifies the current compute shader invocation across the current dispatch. As such it's often used to index into e.g. buffers. In our sample we use this to index into our particle array.

## Running compute commands 

### Dispatch

Now it's time to actually tell the GPU to do some compute. This is done by calling `vkCmdDispatch` inside a command buffer. While not perfectly true, a dispatch is for compute what a draw call like `vkCmdDraw` is for graphics. This dispatches a given number of compute work items in at max. three dimensions.

```c++
VkCommandBufferBeginInfo beginInfo{};
beginInfo.sType = VK_STRUCTURE_TYPE_COMMAND_BUFFER_BEGIN_INFO;

if (vkBeginCommandBuffer(commandBuffer, &beginInfo) != VK_SUCCESS) {
    throw std::runtime_error("failed to begin recording command buffer!");
}

...

vkCmdBindPipeline(commandBuffer, VK_PIPELINE_BIND_POINT_COMPUTE, computePipeline);
vkCmdBindDescriptorSets(commandBuffer, VK_PIPELINE_BIND_POINT_COMPUTE, computePipelineLayout, 0, 1, &computeDescriptorSets[i], 0, 0);

vkCmdDispatch(computeCommandBuffer, PARTICLE_COUNT / 256, 1, 1);

...

if (vkEndCommandBuffer(commandBuffer) != VK_SUCCESS) {
    throw std::runtime_error("failed to record command buffer!");
}
```

The `vkCmdDispatch` will dispatch `PARTICLE_COUNT / 256` local work groups in the x dimension. As our particles array is linear, we leave the other two dimensions at one, resulting in a one-dimensional dispatch. But why do we divide the number of particles (in our array) by 256? That's because in the previous paragraph we defined that every compute shader in a work group will do 256 invocations. So if we were to have 4096 particles, we would dispatch 16 work groups, with each work group running 256 compute shader invocations. Getting the two number right usually takes some tinkering and profiling, depending on your workload and the hardware you're running on.

And just as was the case for the compute pipeline, a compute command buffer contains a lot less state then a graphics command buffer. There's no need to start a render pass or set a viewport.

### Submitting work

As our sample does both compute and graphics operations, we'll be doing two submits to both the graphics and compute queue per frame (see the `drawFrame` function):

```c++
...
if (vkQueueSubmit(computeQueue, 1, &submitInfo, nullptr) != VK_SUCCESS) {
    throw std::runtime_error("failed to submit compute command buffer!");
};
...
if (vkQueueSubmit(graphicsQueue, 1, &submitInfo, inFlightFences[currentFrame]) != VK_SUCCESS) {
    throw std::runtime_error("failed to submit draw command buffer!");
}
```

The first submit to the compute queue updates the particle positions using the compute shader, and the second submit will then use that updated data to draw the particle system.

### Synchronizing graphics and compute

Synchronization is an important part of Vulkan, even more so When doing compute in conjunction with graphics. Not doing proper synchronization may result in the vertex starting to read (and draw) particles while the compute shader hasn't finished writing them (read-after-write hazard), or the compute shader could start updating particles that are still in use by the vertex part of the pipeline (write-after-read hazard).

So we must make sure that those cases don't happen by synchronizing the graphics and the compute load. There are different ways of doing so, which also depends on how you submit your compute workload but in our case with two separate submits we'll be using [semaphores](03_Drawing_a_triangle/03_Drawing/02_Rendering_and_presentation.md#page_Semaphores) to ensure that the vertex shader won't start fetching vertices until the compute shader has finished updating them.

This is necessary as even though the two `vkQueueSubmit` are ordered one-after-another, there is no guarantee that they execute on the GPU in this order. Adding in wait and signal semaphores ensures this execution order.

So we first add a new set of semaphores for the compute work:

```c++
std::vector<VkSemaphore> computeFinishedSemaphores;
...
computeFinishedSemaphores.resize(MAX_FRAMES_IN_FLIGHT);

for (size_t i = 0; i < MAX_FRAMES_IN_FLIGHT; i++) {
    if (...
        vkCreateSemaphore(device, &semaphoreInfo, nullptr, &computeFinishedSemaphores[i]) != VK_SUCCESS
        ...) {
        throw std::runtime_error("failed to create synchronization objects for a frame!");
    }
}
```
And then use that to synchronize the compute buffer submission with the graphics submission:

```c++
vkWaitForFences(device, 1, &inFlightFences[currentFrame], VK_TRUE, UINT64_MAX);
...

// Compute submission
submitInfo.waitSemaphoreCount = 0;
submitInfo.pWaitSemaphores = nullptr;
submitInfo.commandBufferCount = 1;
submitInfo.pCommandBuffers = &computeCommandBuffers[currentFrame];
submitInfo.signalSemaphoreCount = 1;
submitInfo.pSignalSemaphores = &computeFinishedSemaphores[currentFrame];

if (vkQueueSubmit(computeQueue, 1, &submitInfo, nullptr) != VK_SUCCESS) {
    throw std::runtime_error("failed to submit compute command buffer!");
};

// Graphics submission
VkSemaphore waitSemaphores[] = { computeFinishedSemaphores[currentFrame], imageAvailableSemaphores[currentFrame] };
VkPipelineStageFlags waitStages[] = { VK_PIPELINE_STAGE_VERTEX_INPUT_BIT, VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT };
submitInfo = {};
submitInfo.sType = VK_STRUCTURE_TYPE_SUBMIT_INFO;

submitInfo.waitSemaphoreCount = 2;
submitInfo.pWaitSemaphores = waitSemaphores;
submitInfo.pWaitDstStageMask = waitStages;
submitInfo.commandBufferCount = 1;
submitInfo.pCommandBuffers = &commandBuffers[currentFrame];
submitInfo.signalSemaphoreCount = 1;
submitInfo.pSignalSemaphores = &renderFinishedSemaphores[currentFrame];

if (vkQueueSubmit(graphicsQueue, 1, &submitInfo, inFlightFences[currentFrame]) != VK_SUCCESS) {
    throw std::runtime_error("failed to submit draw command buffer!");
}
```

If we remember the sample in the [semaphores chapter](03_Drawing_a_triangle/03_Drawing/02_Rendering_and_presentation.md#page_Semaphores), this setup will immediately run the compute shader as we haven't specified any wait semaphores. This is fine, as we are waiting for the graphics command buffer to finish execution before the compute submission with the `vkWaitForFences` command.

The graphics submission on the other hand needs to wait for the compute work to finish so it doesn't start fetching vertices while the compute buffer is still updating them. So we wait on the `computeFinishedSemaphores` for the current frame and have the graphics submission wait on the `VK_PIPELINE_STAGE_VERTEX_INPUT_BIT` stage, where vertex buffer is consumed.

But it also needs to wait for presentation so the fragment shader won't output to the color attachments until the image has been presented. So we wait also wait on the `imageAvailableSemaphores` on the current frame at the `VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT` stage.

## Drawing the particle system

Earlier on we learned that buffers in Vulkan can have multiple use-cases and so we created the shader storage buffer that contains our particles with both the shader storage buffer bit and the vertex buffer bit. This means that we can use the shader storage buffer for drawing just as we used "pure" vertex buffers in the previous chapters.

We first setup the vertex input state to match our particle structure:

```c++
struct Particle {
    ...

    static std::array<VkVertexInputAttributeDescription, 2> getAttributeDescriptions() {
        std::array<VkVertexInputAttributeDescription, 2> attributeDescriptions{};

        attributeDescriptions[0].binding = 0;
        attributeDescriptions[0].location = 0;
        attributeDescriptions[0].format = VK_FORMAT_R32G32_SFLOAT;
        attributeDescriptions[0].offset = offsetof(Particle, position);

        attributeDescriptions[1].binding = 0;
        attributeDescriptions[1].location = 1;
        attributeDescriptions[1].format = VK_FORMAT_R32G32B32A32_SFLOAT;
        attributeDescriptions[1].offset = offsetof(Particle, color);

        return attributeDescriptions;
    }
};
```

Note that we don't add `velocity` to the vertex input attributes, as this is only used by the compute shader.

And then bind and draw it like we would with any vertex buffer:

```c++
vkCmdBindVertexBuffers(commandBuffer, 0, 1, &shaderStorageBuffer, offsets);

vkCmdDraw(commandBuffer, PARTICLE_COUNT, 1, 0, 0);
```

## Conclusion

@todo: descriptor pool
@todo: talk about limits
@todo: add author to bonus chapter?

Once you have learned how to use compute shaders you may take a look at some advanced topics like:

- Shared memory
- Asynchronous compute
- Atomic operations
- Subgroups

[C++ code](/code/31_compute_shader.cpp) /
[Vertex shader](/code/31_shader_compute.vert) /
[Fragment shader](/code/31_shader_compute.frag) /
[Compute shader](/code/31_shader_compute.comp)