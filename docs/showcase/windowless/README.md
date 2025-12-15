# Wgpu sin ventana

A veces simplemente queremos aprovechar la gpu. Tal vez queramos procesar un gran conjunto de números en paralelo. Tal vez estamos trabajando en una película 3D y necesitamos crear una escena de apariencia realista con path tracing. Tal vez estamos minando una criptomoneda. En todas estas situaciones, no necesariamente *necesitamos* ver qué está sucediendo.

## Entonces, ¿qué necesitamos hacer?

Es realmente bastante simple. No *necesitamos* una ventana para crear una `Instance`, no *necesitamos* una ventana para seleccionar un `Adapter`, ni *necesitamos* una ventana para crear un `Device`. Solo necesitábamos la ventana para crear una `Surface` que necesitábamos para crear el `SwapChain`. Una vez que tenemos un `Device`, tenemos todo lo que necesitamos para comenzar a enviar comandos a la gpu.

```rust
let adapter = instance
    .request_adapter(&wgpu::RequestAdapterOptions {
        power_preference: wgpu::PowerPreference::default(),
        compatible_surface: None,
    })
    .await
    .unwrap();
let (device, queue) = adapter
    .request_device(&Default::default(), None)
    .await
    .unwrap();
```

## Un triángulo sin ventana

Ahora hemos hablado sobre no necesitar ver qué está haciendo la gpu, pero necesitamos ver los resultados en algún momento. Si miramos atrás la discusión sobre la [surface](/beginner/tutorial2-surface/#render) vemos que usamos `surface.get_current_texture()` para obtener una textura para dibujar. Saltaremos ese paso creando la textura nosotros mismos. Una cosa a tener en cuenta aquí es que necesitamos especificar `wgpu::TextureFormat::Rgba8UnormSrgb` al `format` en lugar de `surface.get_preferred_format(&adapter)` ya que PNG usa RGBA, no BGRA.

```rust
let texture_size = 256u32;

let texture_desc = wgpu::TextureDescriptor {
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
        | wgpu::TextureUsages::RENDER_ATTACHMENT
        ,
    label: None,
};
let texture = device.create_texture(&texture_desc);
let texture_view = texture.create_view(&Default::default());
```

Estamos usando `TextureUsages::RENDER_ATTACHMENT` para que wgpu pueda renderizar a nuestra textura. El `TextureUsages::COPY_SRC` es para que podamos extraer datos de la textura para poder guardarla en un archivo.

Aunque podemos usar esta textura para dibujar nuestro triángulo, necesitamos alguna forma de acceder a los píxeles dentro de ella. En el [tutorial de texturas](/beginner/tutorial5-textures/) usamos un búfer para cargar datos de color desde un archivo que luego copiamos en nuestro búfer. Ahora vamos a hacer lo inverso: copiar datos a un búfer desde nuestra textura para guardar en un archivo. Necesitaremos un búfer lo suficientemente grande para nuestros datos.

```rust
// we need to store this for later
let u32_size = std::mem::size_of::<u32>() as u32;

let output_buffer_size = (u32_size * texture_size * texture_size) as wgpu::BufferAddress;
let output_buffer_desc = wgpu::BufferDescriptor {
    size: output_buffer_size,
    usage: wgpu::BufferUsages::COPY_DST
        // this tells wpgu that we want to read this buffer from the cpu
        | wgpu::BufferUsages::MAP_READ,
    label: None,
    mapped_at_creation: false,
};
let output_buffer = device.create_buffer(&output_buffer_desc);
```

Ahora que tenemos algo en lo que dibujar, hagamos algo para dibujar. Como solo estamos dibujando un triángulo, tomemos el código del shader del [tutorial de pipeline](/beginner/tutorial3-pipeline/#writing-the-shaders).

```glsl
// shader.vert
#version 450

const vec2 positions[3] = vec2[3](
    vec2(0.0, 0.5),
    vec2(-0.5, -0.5),
    vec2(0.5, -0.5)
);

void main() {
    gl_Position = vec4(positions[gl_VertexIndex], 0.0, 1.0);
}
```

```glsl
// shader.frag
#version 450

layout(location=0) out vec4 f_color;

void main() {
    f_color = vec4(0.3, 0.2, 0.1, 1.0);
}
```

Actualiza las dependencias para soportar módulos SPIR-V.

```toml
[dependencies]
image = "0.23"
shaderc = "0.7"
wgpu = { version = "27.0.0", features = ["spirv"] }
pollster = "0.3"
```

Usando eso crearemos un simple `RenderPipeline`.

```rust
let vs_src = include_str!("shader.vert");
let fs_src = include_str!("shader.frag");
let mut compiler = shaderc::Compiler::new().unwrap();
let vs_spirv = compiler
    .compile_into_spirv(
        vs_src,
        shaderc::ShaderKind::Vertex,
        "shader.vert",
        "main",
        None,
    )
    .unwrap();
let fs_spirv = compiler
    .compile_into_spirv(
        fs_src,
        shaderc::ShaderKind::Fragment,
        "shader.frag",
        "main",
        None,
    )
    .unwrap();
let vs_data = wgpu::util::make_spirv(vs_spirv.as_binary_u8());
let fs_data = wgpu::util::make_spirv(fs_spirv.as_binary_u8());
let vs_module = device.create_shader_module(wgpu::ShaderModuleDescriptor {
    label: Some("Vertex Shader"),
    source: vs_data,
    flags: wgpu::ShaderFlags::default(),
});
let fs_module = device.create_shader_module(wgpu::ShaderModuleDescriptor {
    label: Some("Fragment Shader"),
    source: fs_data,
    flags: wgpu::ShaderFlags::default(),
});

let render_pipeline_layout = device.create_pipeline_layout(&wgpu::PipelineLayoutDescriptor {
    label: Some("Render Pipeline Layout"),
    bind_group_layouts: &[],
    push_constant_ranges: &[],
});

let render_pipeline = device.create_render_pipeline(&wgpu::RenderPipelineDescriptor {
    label: Some("Render Pipeline"),
    layout: Some(&render_pipeline_layout),
    vertex: wgpu::VertexState {
        module: &vs_module,
        entry_point: Some("main"),
        buffers: &[],
    },
    fragment: Some(wgpu::FragmentState {
        module: &fs_module,
        entry_point: Some("main"),
        targets: &[Some(wgpu::ColorTargetState {
            format: texture_desc.format,
            alpha_blend: wgpu::BlendState::REPLACE,
            color_blend: wgpu::BlendState::REPLACE,
            write_mask: wgpu::ColorWrites::ALL,
        })],
    }),
    primitive: wgpu::PrimitiveState {
        topology: wgpu::PrimitiveTopology::TriangleList,
        strip_index_format: None,
        front_face: wgpu::FrontFace::Ccw,
        cull_mode: Some(wgpu::Face::Back),
        // Setting this to anything other than Fill requires Features::NON_FILL_POLYGON_MODE
        polygon_mode: wgpu::PolygonMode::Fill,
    },
    depth_stencil: None,
    multisample: wgpu::MultisampleState {
        count: 1,
        mask: !0,
        alpha_to_coverage_enabled: false,
    },
});
```

Vamos a necesitar un encoder, así que hagámoslo.

```rust
let mut encoder = device.create_command_encoder(&wgpu::CommandEncoderDescriptor {
    label: None,
});
```

El `RenderPass` es donde las cosas se ponen interesantes. Un render pass requiere al menos un color attachment. Un color attachment requiere un `TextureView` para adjuntar. Solíamos usar una textura del `SwapChain` para esto, pero cualquier `TextureView` funcionará, incluida nuestra `texture_view`.

```rust
{
    let render_pass_desc = wgpu::RenderPassDescriptor {
        label: Some("Render Pass"),
        color_attachments: &[
            wgpu::RenderPassColorAttachment {
                view: &texture_view,
                resolve_target: None,
                ops: wgpu::Operations {
                    load: wgpu::LoadOp::Clear(wgpu::Color {
                        r: 0.1,
                        g: 0.2,
                        b: 0.3,
                        a: 1.0,
                    }),
                    store: wgpu::StoreOp::Store,
                },
            }
        ],
        depth_stencil_attachment: None,
    };
    let mut render_pass = encoder.begin_render_pass(&render_pass_desc);

    render_pass.set_pipeline(&render_pipeline);
    render_pass.draw(0..3, 0..1);
}
```

No hay mucho que podamos hacer con los datos cuando están atrapados en una `Texture`, así que copiémoslos en nuestro `output_buffer`.

```rust
encoder.copy_texture_to_buffer(
    wgpu::TexelCopyTextureInfo {
        aspect: wgpu::TextureAspect::All,
                texture: &texture,
        mip_level: 0,
        origin: wgpu::Origin3d::ZERO,
    },
    wgpu::TexelCopyBufferInfo {
        buffer: &output_buffer,
        layout: wgpu::TexelCopyBufferLayout {
            offset: 0,
            bytes_per_row: u32_size * texture_size,
            rows_per_image: texture_size,
        },
    },
    texture_desc.size,
);
```

Ahora que hemos hecho todos nuestros comandos, enviémoslos a la gpu.

```rust
queue.submit(Some(encoder.finish()));
```

## Extrayendo datos de un búfer

Para obtener los datos fuera del búfer, primero necesitamos mapearlo, entonces podemos obtener una `BufferView` que podemos tratar como `&[u8]`.

```rust
// We need to scope the mapping variables so that we can
// unmap the buffer
{
    let buffer_slice = output_buffer.slice(..);

    // NOTE: We have to create the mapping THEN device.poll() before await
    // the future. Otherwise the application will freeze.
    let (tx, rx) = futures_intrusive::channel::shared::oneshot_channel();
    buffer_slice.map_async(wgpu::MapMode::Read, move |result| {
        tx.send(result).unwrap();
    });
    device.poll(wgpu::PollType::wait_indefinitely())?;
    rx.receive().await.unwrap().unwrap();

    let data = buffer_slice.get_mapped_range();

    use image::{ImageBuffer, Rgba};
    let buffer =
        ImageBuffer::<Rgba<u8>, _>::from_raw(texture_size, texture_size, data).unwrap();
    buffer.save("image.png").unwrap();

}
output_buffer.unmap();
```

<div class="note">

Usé [futures-intrusive](https://docs.rs/futures-intrusive) ya que es el crate que usan en los [ejemplos en el repositorio de wgpu](https://github.com/gfx-rs/wgpu/tree/master/wgpu/examples/capture).

</div>

## Main no es asincrónico

El método `main()` no puede retornar un future, así que no podemos usar la palabra clave `async`. Lo solucionaremos poniendo nuestro código en una función diferente para que podamos bloquearlo en `main()`. Necesitarás usar un crate que pueda sondear futures como el [crate pollster](https://docs.rs/pollster).

<div class="note">

Hay crates como [async-std](https://docs.rs/async-std) y [tokio](https://docs.rs/tokio) que puedes usar para anotar `main()` para que sea asincrónico. Opté por no hacerlo ya que ambos crates son un poco más pesados para este proyecto. Eres bienvenido a usar cualquier configuración asincrónica que prefieras :slightly_smiling_face:

</div>

```rust
async fn run() {
    // Windowless drawing code...
}

fn main() {
    run().unwrap();
}
```

Con todo eso, deberías tener una imagen como esta.

![a brown triangle](./image-output.png)

<AutoGithubLink/>
