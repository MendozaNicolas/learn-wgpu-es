# Creando GIFs

A veces has creado una simulación/animación interesante y quieres exhibirla. Si bien puedes grabar un video, eso podría ser excesivo si solo quieres algo para publicar en Twitter. Para eso es para lo que sirven los [GIF](https://en.wikipedia.org/wiki/GIF)s.

Además, GIF se pronuncia GHIF, no JIF ya que JIF no solo es [mantequilla de cacahuete](https://en.wikipedia.org/wiki/Jif_%28peanut_butter%29), sino que también es un [formato de imagen diferente](https://filext.com/file-extension/JIF).

## ¿Cómo estamos creando el GIF?

Vamos a crear una función usando el [gif crate](https://docs.rs/gif/) para codificar la imagen actual.

```rust
fn save_gif(path: &str, frames: &mut Vec<Vec<u8>>, speed: i32, size: u16) -> Result<(), failure::Error> {
    use gif::{Frame, Encoder, Repeat, SetParameter};
    
    let mut image = std::fs::File::create(path)?;
    let mut encoder = Encoder::new(&mut image, size, size, &[])?;
    encoder.set(Repeat::Infinite)?;

    for mut frame in frames {
        encoder.write_frame(&Frame::from_rgba_speed(size, size, &mut frame, speed))?;
    }

    Ok(())
}
```

<!-- image-rs doesn't currently support looping, so I switched to gif -->
<!-- A GIF is a type of image, and fortunately, the [image crate](https://docs.rs/image/) supports GIFs natively. It's pretty simple to use. -->

<!-- ```rust
fn save_gif(path: &str, frames: &mut Vec<Vec<u8>>, speed: i32, size: u16) -> Result<(), failure::Error> {
    let output = std::fs::File::create(path)?;
    let mut encoder = image::gif::Encoder::new(output);

    for mut data in frames {
        let frame = image::gif::Frame::from_rgba_speed(size, size, &mut data, speed);
        encoder.encode(&frame)?;
    }

    Ok(())
}
``` -->

Todo lo que necesitamos para usar este código son los fotogramas del GIF, qué tan rápido debería ejecutarse y el tamaño del GIF (podrías usar ancho y alto por separado, pero no lo hice).

## ¿Cómo hacemos los fotogramas?

Si revisaste el [showcase sin ventana](../windowless/#a-triangle-without-a-window), sabrás que renderizamos directamente a una `wgpu::Texture`. Crearemos una textura para renderizar y un búfer para copiar la salida.

```rust
// create a texture to render to
let texture_size = 256u32;
let rt_desc = wgpu::TextureDescriptor {
    size: wgpu::Extent3d {
        width: texture_size,
        height: texture_size,
        depth_or_array_layers: 1,
    },
    mip_level_count: 1,
    sample_count: 1,
    dimension: wgpu::TextureDimension::D2,
    format: wgpu::TextureFormat::Rgba8UnormSrgb,
    usage: wgpu::TextureUsages::COPY_SRC
        | wgpu::TextureUsages::RENDER_ATTACHMENT,
    label: None,
};
let render_target = framework::Texture::from_descriptor(&device, rt_desc);

// wgpu requires texture -> buffer copies to be aligned using
// wgpu::COPY_BYTES_PER_ROW_ALIGNMENT. Because of this we'll
// need to save both the padded_bytes_per_row as well as the
// unpadded_bytes_per_row
let pixel_size = mem::size_of::<[u8;4]>() as u32;
let align = wgpu::COPY_BYTES_PER_ROW_ALIGNMENT;
let unpadded_bytes_per_row = pixel_size * texture_size;
let padding = (align - unpadded_bytes_per_row % align) % align;
let padded_bytes_per_row = unpadded_bytes_per_row + padding;

// create a buffer to copy the texture to so we can get the data
let buffer_size = (padded_bytes_per_row * texture_size) as wgpu::BufferAddress;
let buffer_desc = wgpu::BufferDescriptor {
    size: buffer_size,
    usage: wgpu::BufferUsages::COPY_DST | wgpu::BufferUsages::MAP_READ,
    label: Some("Output Buffer"),
    mapped_at_creation: false,
};
let output_buffer = device.create_buffer(&buffer_desc);
```

Con eso, podemos renderizar un fotograma y luego copiar ese fotograma a un `Vec<u8>`.

```rust
let mut frames = Vec::new();

for c in &colors {
    let mut encoder = device.create_command_encoder(&wgpu::CommandEncoderDescriptor {
        label: None,
    });

    let mut rpass = encoder.begin_render_pass(&wgpu::RenderPassDescriptor {
        label: Some("GIF Pass"),
        color_attachments: &[
            wgpu::RenderPassColorAttachment {
                view: &render_target.view,
                resolve_target: None,
                ops: wgpu::Operations {
                    load: wgpu::LoadOp::Clear(
                        wgpu::Color {
                            r: c[0],
                            g: c[1],
                            b: c[2],
                            a: 1.0,
                        }
                    ),
                    store: wgpu::StoreOp::Store,
                },
            }
        ],
        depth_stencil_attachment: None,
    });

    rpass.set_pipeline(&render_pipeline);
    rpass.draw(0..3, 0..1);

    drop(rpass);

    encoder.copy_texture_to_buffer(
        wgpu::TexelCopyTextureInfo {
            texture: &render_target.texture,
            mip_level: 0,
            origin: wgpu::Origin3d::ZERO,
        }, 
        wgpu::TexelCopyBufferInfo {
            buffer: &output_buffer,
            layout: wgpu::TexelCopyBufferLayout {
                offset: 0,
                bytes_per_row: padded_bytes_per_row,
                rows_per_image: texture_size,
            }
        },
        render_target.desc.size
    );

    queue.submit(std::iter::once(encoder.finish()));
    
    // Create the map request
    let buffer_slice = output_buffer.slice(..);
    let request = buffer_slice.map_async(wgpu::MapMode::Read);
    // wait for the GPU to finish
    device.poll(wgpu::PollType::wait_indefinitely())?;
    let result = request.await;
    
    match result {
        Ok(()) => {
            let padded_data = buffer_slice.get_mapped_range();
            let data = padded_data
                .chunks(padded_bytes_per_row as _)
                .map(|chunk| { &chunk[..unpadded_bytes_per_row as _]})
                .flatten()
                .map(|x| { *x })
                .collect::<Vec<_>>();
            drop(padded_data);
            output_buffer.unmap();
            frames.push(data);
        }
        _ => { eprintln!("Something went wrong") }
    }

}
```

Una vez hecho eso, podemos pasar nuestros fotogramas a `save_gif()`.

```rust
save_gif("output.gif", &mut frames, 1, texture_size as u16).unwrap();
```

Esa es la esencia. Podemos mejorar las cosas usando un arreglo de texturas y enviando todos los comandos de dibujo a la vez, pero esto comunica la idea. Con el shader que escribí obtenemos el siguiente GIF.


![./output.gif](./output.gif)

<AutoGithubLink/>
