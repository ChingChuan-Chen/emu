> The old version of Emu (which used macros and stuff) is [here](https://github.com/calebwin/emu/tree/master/em).

[![Discord Chat](https://img.shields.io/discord/308323056592486420.svg)](https://discord.gg/sKf6KCs)

<p align="center">
<!-- <img width="250px" src="https://i.imgur.com/kTap42K.png"/> -->
    <img width="250px" src="https://i.imgur.com/CZEkdK1.png"/>
</p>

Emu is a GPGPU library with a focus on portability, modularity, and performance. 

It's a CUDA-esque compute-specific abstraction over [WebGPU](https://github.com/gfx-rs/wgpu-rs) providing specific functionality to make WebGPU feel more like CUDA. Here's a quick run-down of highlight features...

- **Emu can run anywhere** - Emu uses WebGPU to support DirectX, Metal, Vulkan (and also OpenGL and browser eventually) as compile targets. This allows Emu to run on pretty much any user interface including desktop, mobile, and browser. By moving 
heavy computations to the user's device, you can reduce system latency and improve privacy.

- **Emu makes compute easier** - Emu makes WebGPU feel like CUDA. It does this by providing...
    - `DeviceBox<T>` as a wrapper for data that lives on the GPU (thereby ensuring type-safe data movement)
    - `DevicePool` as a no-config auto-managed pool of devices (similar to CUDA)
    - `trait Cache` - a no-setup-required LRU cache of JITed compute kernels.
    
- **Emu is transparent** - Emu is a fully transparent abstraction. This means, at any point, you can decide to remove the abstraction and work directly with WebGPU constructs with zero overhead. For example, if you want to mix Emu with WebGPU-based graphics, you can do that with zero overhead. You can also swap out the JIT compiler artifact cache with your own cache, manage the device pool if you wish, and define your own compile-to-SPIR-V compiler that interops with Emu.

Here's a quick example of Emu. You can find more in `emu_core/examples`.

First, we just import a bunch of stuff
```rust
use emu_glsl::*;
use emu_core::prelude::*;
use zerocopy::*;
```
We can define types of structures so that they can be safely serialized and deserialized to/from the GPU.
```rust
#[repr(C)]
#[derive(AsBytes, FromBytes, Copy, Clone, Default, Debug)]
struct Rectangle {
    x: u32,
    y: u32,
    w: i32,
    h: i32,
}
```
For this example, we make this entire function async but in reality you will only want small blocks of code to be async (like a bunch of asynchronous memory transfers and computation) and these blocks will be sent off to an executor to execute. You definitely don't want to do something like this where you are blocking (by doing an entire compilation step) in your async code.
```rust
fn main() -> Result<(), Box<dyn std::error::Error>> {
    futures::executor::block_on(assert_device_pool_initialized());

    // first, we move a bunch of rectangles to the GPU
    let mut x: DeviceBox<[Rectangle]> = vec![Default::default(); 128].as_device_boxed()?;
    
    // then we compile some GLSL code using the GlslCompile compiler and
    // the GlobalCache for caching compiler artifacts
    let c = compile::<String, GlslCompile, _, GlobalCache>(
        GlslBuilder::new()
            .set_entry_point_name("main")
            .add_param_mut()
            .set_code_with_glsl(
            r#"
#version 450
layout(local_size_x = 1) in; // our thread block size is 1, that is we only have 1 thread per block

struct Rectangle {
    uint x;
    uint y;
    int w;
    int h;
};

// make sure to use only a single set and keep all your n parameters in n storage buffers in bindings 0 to n-1
// you shouldn't use push constants or anything OTHER than storage buffers for passing stuff into the kernel
// just use buffers with one buffer per binding
layout(set = 0, binding = 0) buffer Rectangles {
    Rectangle[] rectangles;
}; // this is used as both input and output for convenience

Rectangle flip(Rectangle r) {
    r.x = r.x + r.w;
    r.y = r.y + r.h;
    r.w *= -1;
    r.h *= -1;
    return r;
}

// there should be only one entry point and it should be named "main"
// ultimately, Emu has to kind of restrict how you use GLSL because it is compute focused
void main() {
    uint index = gl_GlobalInvocationID.x; // this gives us the index in the x dimension of the thread space
    rectangles[index] = flip(rectangles[index]);
}
            "#,
        )
    )?.finish()?;
    
    // we spawn 128 threads (really 128 thread blocks)
    unsafe {
        spawn(128).launch(call!(c, &mut x));
    }

    // this is the Future we need to block on to get stuff to happen
    // everything else is non-blocking in the API (except stuff like compilation)
    println!("{:?}", futures::executor::block_on(x.get())?);

    Ok(())
}
```
And last but certainly not least, we use an executor to execute.
```rust
fn main() {
    futures::executor::block_on(do_some_stuff()).expect("failed to do stuff on GPU");
}
```

The latest version is not yet published (just waiting for `wgpu-rs` to get its 0.5.0 version released) but you can definitely clone the repository if you want to check things out for now.

If you have any questions, please [ask in the Discord](https://discord.gg/sKf6KCs).
