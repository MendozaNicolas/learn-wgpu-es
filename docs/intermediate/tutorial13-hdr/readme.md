# Renderizado de Alto Rango Dinámico

Hasta este punto, hemos estado usando el espacio de color sRGB para renderizar nuestra escena. Si bien esto está bien, limita lo que podemos hacer con nuestro sistema de iluminación. Estamos usando `TextureFormat::Bgra8UnormSrgb` (en la mayoría de los sistemas) para nuestra textura de superficie. Esto significa que tenemos 8 bits para cada canal de rojo, verde, azul y alfa. Si bien los canales se almacenan como números enteros entre 0 y 255 inclusive, se convierten a y desde valores de punto flotante entre 0.0 y 1.0. El resumen es que usando texturas de 8 bits, solo obtenemos 256 valores posibles en cada canal.

El problema con esto es que la mayoría de la precisión se utiliza para representar valores más oscuros de la escena. Esto significa que objetos brillantes como bombillas tienen el mismo valor que objetos extremadamente brillantes como el sol. Esta inexactitud hace que la iluminación realista sea difícil de hacer correctamente. Por esta razón, vamos a cambiar nuestro sistema de renderizado para usar alto rango dinámico para dar a nuestra escena más flexibilidad y permitirnos aprovechar técnicas más avanzadas como Renderizado Basado en Física.

## ¿Qué es el Alto Rango Dinámico?

En términos simples, una textura de Alto Rango Dinámico es una textura con más bits por píxel. Además de esto, las texturas HDR se almacenan como valores de punto flotante en lugar de valores enteros. Esto significa que la textura puede tener valores de brillo mayores a 1.0, lo que significa que puedes tener un rango dinámico de objetos más brillantes.

## Cambio a HDR

En el momento de escribir esto, wgpu no nos permite usar un formato de punto flotante como `TextureFormat::Rgba16Float` como formato de textura de superficie (no todos los monitores lo soportan de todas formas), así que tendremos que renderizar nuestra escena en un formato HDR y luego convertir los valores a un formato soportado, como `TextureFormat::Bgra8UnormSrgb` usando una técnica llamada mapeo de tonos.

<div class="note">

Hay algunas conversaciones sobre la implementación de soporte de textura de superficie HDR en wgpu. Aquí hay un problema de GitHub si deseas contribuir a ese esfuerzo: https://github.com/gfx-rs/wgpu/issues/2920

</div>

Antes de hacer eso, sin embargo, necesitamos cambiar a usar una textura HDR para renderizar.

Para empezar, crearemos un archivo llamado `hdr.rs` y colocaremos algo de código en él:

```rust
use wgpu::Operations;

use crate::{create_render_pipeline, texture};

/// Owns the render texture and controls tonemapping
pub struct HdrPipeline {
    pipeline: wgpu::RenderPipeline,
    bind_group: wgpu::BindGroup,
    texture: texture::Texture,
    width: u32,
    height: u32,
    format: wgpu::TextureFormat,
    layout: wgpu::BindGroupLayout,
}

impl HdrPipeline {
    pub fn new(device: &wgpu::Device, config: &wgpu::SurfaceConfiguration) -> Self {
        let width = config.width;
        let height = config.height;

        // We could use `Rgba32Float`, but that requires some extra
        // features to be enabled for rendering.
        let format = wgpu::TextureFormat::Rgba16Float;

        let texture = texture::Texture::create_2d_texture(
            device,
            width,
            height,
            format,
            wgpu::TextureUsages::TEXTURE_BINDING | wgpu::TextureUsages::RENDER_ATTACHMENT,
            wgpu::FilterMode::Nearest,
            Some("Hdr::texture"),
        );

        let layout = device.create_bind_group_layout(&wgpu::BindGroupLayoutDescriptor {
            label: Some("Hdr::layout"),
            entries: &[
                // This is the HDR texture
                wgpu::BindGroupLayoutEntry {
                    binding: 0,
                    visibility: wgpu::ShaderStages::FRAGMENT,
                    ty: wgpu::BindingType::Texture {
                        sample_type: wgpu::TextureSampleType::Float { filterable: true },
                        view_dimension: wgpu::TextureViewDimension::D2,
                        multisampled: false,
                    },
                    count: None,
                },
                wgpu::BindGroupLayoutEntry {
                    binding: 1,
                    visibility: wgpu::ShaderStages::FRAGMENT,
                    ty: wgpu::BindingType::Sampler(wgpu::SamplerBindingType::Filtering),
                    count: None,
                },
            ],
        });
        let bind_group = device.create_bind_group(&wgpu::BindGroupDescriptor {
            label: Some("Hdr::bind_group"),
            layout: &layout,
            entries: &[
                wgpu::BindGroupEntry {
                    binding: 0,
                    resource: wgpu::BindingResource::TextureView(&texture.view),
                },
                wgpu::BindGroupEntry {
                    binding: 1,
                    resource: wgpu::BindingResource::Sampler(&texture.sampler),
                },
            ],
        });

        // We'll cover the shader next
        let shader = wgpu::include_wgsl!("hdr.wgsl");
        let pipeline_layout = device.create_pipeline_layout(&wgpu::PipelineLayoutDescriptor {
            label: None,
            bind_group_layouts: &[&layout],
            push_constant_ranges: &[],
        });

        let pipeline = create_render_pipeline(
            device,
            &pipeline_layout,
            config.format.add_srgb_suffix(),
            None,
            // We'll use some math to generate the vertex data in
            // the shader, so we don't need any vertex buffers
            &[],
            wgpu::PrimitiveTopology::TriangleList,
            shader,
        );

        Self {
            pipeline,
            bind_group,
            layout,
            texture,
            width,
            height,
            format,
        }
    }

    /// Resize the HDR texture
    pub fn resize(&mut self, device: &wgpu::Device, width: u32, height: u32) {
        self.texture = texture::Texture::create_2d_texture(
            device,
            width,
            height,
            wgpu::TextureFormat::Rgba16Float,
            wgpu::TextureUsages::TEXTURE_BINDING | wgpu::TextureUsages::RENDER_ATTACHMENT,
            wgpu::FilterMode::Nearest,
            Some("Hdr::texture"),
        );
        self.bind_group = device.create_bind_group(&wgpu::BindGroupDescriptor {
            label: Some("Hdr::bind_group"),
            layout: &self.layout,
            entries: &[
                wgpu::BindGroupEntry {
                    binding: 0,
                    resource: wgpu::BindingResource::TextureView(&self.texture.view),
                },
                wgpu::BindGroupEntry {
                    binding: 1,
                    resource: wgpu::BindingResource::Sampler(&self.texture.sampler),
                },
            ],
        });
        self.width = width;
        self.height = height;
    }

    /// Exposes the HDR texture
    pub fn view(&self) -> &wgpu::TextureView {
        &self.texture.view
    }

    /// The format of the HDR texture
    pub fn format(&self) -> wgpu::TextureFormat {
        self.format
    }

    /// This renders the internal HDR texture to the [TextureView]
    /// supplied as parameter.
    pub fn process(&self, encoder: &mut wgpu::CommandEncoder, output: &wgpu::TextureView) {
        let mut pass = encoder.begin_render_pass(&wgpu::RenderPassDescriptor {
            label: Some("Hdr::process"),
            color_attachments: &[Some(wgpu::RenderPassColorAttachment {
                view: &output,
                resolve_target: None,
                ops: Operations {
                    load: wgpu::LoadOp::Load,
                    store: wgpu::StoreOp::Store,
                },
            })],
            depth_stencil_attachment: None,
        });
        pass.set_pipeline(&self.pipeline);
        pass.set_bind_group(0, &self.bind_group, &[]);
        pass.draw(0..3, 0..1);
    }
}
```

Quizás hayas notado que agregamos un nuevo parámetro a `create_render_pipeline`. Aquí están los cambios a esa función:

```rust
fn create_render_pipeline(
    device: &wgpu::Device,
    layout: &wgpu::PipelineLayout,
    color_format: wgpu::TextureFormat,
    depth_format: Option<wgpu::TextureFormat>,
    vertex_layouts: &[wgpu::VertexBufferLayout],
    topology: wgpu::PrimitiveTopology, // NEW!
    shader: wgpu::ShaderModuleDescriptor,
) -> wgpu::RenderPipeline {
    let shader = device.create_shader_module(shader);

    device.create_render_pipeline(&wgpu::RenderPipelineDescriptor {
        // ...
        primitive: wgpu::PrimitiveState {
            topology, // NEW!
            // ...
        },
        // ...
    })
}
```

## Mapeo de Tonos

El proceso de mapeo de tonos es tomar una imagen HDR y convertirla a un Rango Dinámico Estándar (SDR), que generalmente es sRGB. La curva exacta de mapeo de tonos que uses depende en última instancia de tus necesidades artísticas, pero para este tutorial, usaremos una popular conocida como el Sistema de Codificación de Color de la Academia o ACES utilizado en toda la industria de videojuegos y la industria cinematográfica.

Con eso, saltemos al shader. Crea un archivo llamado `hdr.wgsl` y agrega el siguiente código:

```wgsl
// Maps HDR values to linear values
// Based on http://www.oscars.org/science-technology/sci-tech-projects/aces
fn aces_tone_map(hdr: vec3<f32>) -> vec3<f32> {
    let m1 = mat3x3(
        0.59719, 0.07600, 0.02840,
        0.35458, 0.90834, 0.13383,
        0.04823, 0.01566, 0.83777,
    );
    let m2 = mat3x3(
        1.60475, -0.10208, -0.00327,
        -0.53108,  1.10813, -0.07276,
        -0.07367, -0.00605,  1.07602,
    );
    let v = m1 * hdr;
    let a = v * (v + 0.0245786) - 0.000090537;
    let b = v * (0.983729 * v + 0.4329510) + 0.238081;
    return clamp(m2 * (a / b), vec3(0.0), vec3(1.0));
}

struct VertexOutput {
    @location(0) uv: vec2<f32>,
    @builtin(position) clip_position: vec4<f32>,
};

@vertex
fn vs_main(
    @builtin(vertex_index) vi: u32,
) -> VertexOutput {
    var out: VertexOutput;
    // Generate a triangle that covers the whole screen
    out.uv = vec2<f32>(
        f32((vi << 1u) & 2u),
        f32(vi & 2u),
    );
    out.clip_position = vec4<f32>(out.uv * 2.0 - 1.0, 0.0, 1.0);
    // We need to invert the y coordinate so the image
    // is not upside down
    out.uv.y = 1.0 - out.uv.y;
    return out;
}

@group(0)
@binding(0)
var hdr_image: texture_2d<f32>;

@group(0)
@binding(1)
var hdr_sampler: sampler;

@fragment
fn fs_main(vs: VertexOutput) -> @location(0) vec4<f32> {
    let hdr = textureSample(hdr_image, hdr_sampler, vs.uv);
    let sdr = aces_tone_map(hdr.rgb);
    return vec4(sdr, hdr.a);
}
```

Con eso en su lugar, podemos empezar a usar nuestra textura HDR en nuestra tubería de renderizado principal. Primero, necesitamos agregar el nuevo `HdrPipeline` a `State`:

```rust
// lib.rs

mod hdr; // NUEVO!

// ...

pub struct State {
    // ...
    // NUEVO!
    hdr: hdr::HdrPipeline,
}

impl State {
    pub fn new(window: Window) -> anyhow::Result<Self> {
        // ...
        // NUEVO!
        let hdr = hdr::HdrPipeline::new(&device, &config);

        // ...

        Self {
            // ...
            hdr, // NUEVO!
        }
    }
}
```

Luego, cuando cambiamos el tamaño de la ventana, necesitamos llamar a `resize()` en nuestro `HdrPipeline`:

```rust
fn resize(&mut self, width: u32, height: u32) {
    // ACTUALIZADO!
    if width > 0 && height > 0 {
        // ...
        self.hdr
            .resize(&self.device, new_size.width, new_size.height);
        // ...
    }
}
```

A continuación, en `render()`, necesitamos cambiar el `RenderPass` para usar nuestra textura HDR en lugar de la textura de superficie:

```rust
// render()
let mut render_pass = encoder.begin_render_pass(&wgpu::RenderPassDescriptor {
    label: Some("Render Pass"),
    color_attachments: &[Some(wgpu::RenderPassColorAttachment {
        view: self.hdr.view(), // ACTUALIZADO!
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
    })],
    depth_stencil_attachment: Some(
        // ...
    ),
});
```

Finalmente, después de dibujar todos los objetos en el fotograma, podemos ejecutar nuestro mapeador de tonos con la textura de superficie como salida:

```rust
// NUEVO!
// Aplicar mapeo de tonos
self.hdr.process(&mut encoder, &view);
```

Es un cambio bastante fácil. Aquí está la imagen antes de usar HDR:

![before hdr](./before-hdr.png)

Así es como se ve después de implementar HDR:

![after hdr](./after-hdr.png)

## Cargando texturas HDR

Ahora que tenemos un búfer de renderizado HDR, podemos comenzar a aprovechar las texturas HDR al máximo. Uno de los usos principales de las texturas HDR es almacenar información de iluminación en forma de mapa de ambiente.

Este mapa se puede usar para iluminar objetos, mostrar reflejos y también para hacer un cielo. Vamos a crear un cielo usando textura HDR, pero primero, necesitamos hablar sobre cómo se almacenan los mapas de ambiente.

## Texturas Equirectangulares

Una textura equirectangular es una textura donde una esfera se estira en una superficie rectangular usando lo que se conoce como proyección equirectangular. Este mapa de la Tierra es un ejemplo de esta proyección.

![map of the earth](https://upload.wikimedia.org/wikipedia/commons/thumb/8/83/Equirectangular_projection_SW.jpg/1024px-Equirectangular_projection_SW.jpg)

Esta proyección asigna los valores de longitud de la esfera a las coordenadas horizontales de la textura. Los valores de latitud se asignan a las coordenadas verticales. Esto significa que el medio vertical de la textura es el ecuador (latitud 0°) de la esfera, el medio horizontal es el meridiano principal (longitud 0°) de la esfera, los bordes izquierdo y derecho de la textura son el antimeridiano (longitud +180°/-180°) y los bordes superior e inferior de la textura son el polo norte (latitud 90°) y el polo sur (latitud -90°), respectivamente.

![equirectangular diagram](./equirectangular.svg)

Esta proyección simple es fácil de usar, lo que la hace una de las proyecciones más populares para almacenar texturas esféricas. Puedes ver el mapa de ambiente particular que vamos a usar a continuación.

![equirectangular skybox](./kloofendal_43d_clear_puresky.jpg)

## Mapas de Cubo

Si bien técnicamente podemos usar un mapa equirectangular directamente, siempre que hagamos las matemáticas para determinar las coordenadas correctas, es mucho más conveniente convertir nuestro mapa de ambiente en un mapa de cubo.

<div class="info">

Un mapa de cubo es un tipo especial de textura que tiene seis capas. Cada capa corresponde a una cara diferente de un cubo imaginario alineado con los ejes X, Y y Z. Las capas se almacenan en el siguiente orden: +X, -X, +Y, -Y, +Z, -Z.

</div>

Para prepararnos para almacenar la textura del cubo, vamos a crear una nueva estructura llamada `CubeTexture` en `texture.rs`.

```rust
pub struct CubeTexture {
    texture: wgpu::Texture,
    sampler: wgpu::Sampler,
    view: wgpu::TextureView,
}

impl CubeTexture {
    pub fn create_2d(
        device: &wgpu::Device,
        width: u32,
        height: u32,
        format: wgpu::TextureFormat,
        mip_level_count: u32,
        usage: wgpu::TextureUsages,
        mag_filter: wgpu::FilterMode,
        label: Option<&str>,
    ) -> Self {
        let texture = device.create_texture(&wgpu::TextureDescriptor {
            label,
            size: wgpu::Extent3d {
                width,
                height,
                // A cube has 6 sides, so we need 6 layers
                depth_or_array_layers: 6,
            },
            mip_level_count,
            sample_count: 1,
            dimension: wgpu::TextureDimension::D2,
            format,
            usage,
            view_formats: &[],
        });

        let view = texture.create_view(&wgpu::TextureViewDescriptor {
            label,
            dimension: Some(wgpu::TextureViewDimension::Cube),
            array_layer_count: Some(6),
            ..Default::default()
        });

        let sampler = device.create_sampler(&wgpu::SamplerDescriptor {
            label,
            address_mode_u: wgpu::AddressMode::ClampToEdge,
            address_mode_v: wgpu::AddressMode::ClampToEdge,
            address_mode_w: wgpu::AddressMode::ClampToEdge,
            mag_filter,
            min_filter: wgpu::FilterMode::Nearest,
            mipmap_filter: wgpu::FilterMode::Nearest,
            ..Default::default()
        });

        Self {
            texture,
            sampler,
            view,
        }
    }

    pub fn texture(&self) -> &wgpu::Texture { &self.texture }
    
    pub fn view(&self) -> &wgpu::TextureView { &self.view }

    pub fn sampler(&self) -> &wgpu::Sampler { &self.sampler }

}
```

Con esto, ahora podemos escribir el código para cargar el HDR en una textura de cubo.

## Shaders de Computación

Hasta este punto, hemos estado usando exclusivamente tuberías de renderizado, pero sentí que era un buen momento para introducir las tuberías de computación y, por extensión, los shaders de computación. Las tuberías de computación son mucho más fáciles de configurar. Todo lo que necesitas es decirle a la tubería qué recursos deseas usar, qué código deseas ejecutar y cuántos hilos te gustaría que la GPU use al ejecutar tu código. Vamos a usar un shader de computación para dar a cada píxel en nuestra textura de cubo un color de la imagen HDR.

Antes de que podamos usar shaders de computación, necesitamos habilitarlos en wgpu. Podemos hacerlo cambiando la línea donde especificamos qué características queremos usar. En `lib.rs`, cambia el código donde solicitamos un dispositivo:

```rust
let (device, queue) = adapter
    .request_device(
        &wgpu::DeviceDescriptor {
            label: None,
            // UPDATED!
            features: wgpu::Features::all_webgpu_mask(),
            // UPDATED!
            required_limits: wgpu::Limits::downlevel_defaults(),
        },
        None, // Trace path
    )
    .await
    .unwrap();
```

<div class="warn">

Quizás hayas notado que hemos cambiado de `downlevel_webgl2_defaults()` a `downlevel_defaults()`. Esto significa que estamos eliminando el soporte para WebGL2. La razón es que WebGL2 no admite shaders de computación. WebGPU fue construido teniendo en cuenta los shaders de computación. En el momento de escribir esto, el único navegador que soporta WebGPU es Chrome y algunos navegadores experimentales como Firefox Nightly.

Por consiguiente, vamos a eliminar la característica de WebGL de `Cargo.toml`. Esta línea en particular:

```toml
wgpu = { version = "27.0.0", features = ["webgl"]}
```

</div>

Ahora que le hemos dicho a wgpu que queremos usar shaders de computación, vamos a crear una estructura en `resource.rs` que usaremos para cargar la imagen HDR en nuestro mapa de cubo.

```rust
pub struct HdrLoader {
    texture_format: wgpu::TextureFormat,
    equirect_layout: wgpu::BindGroupLayout,
    equirect_to_cubemap: wgpu::ComputePipeline,
}

impl HdrLoader {
    pub fn new(device: &wgpu::Device) -> Self {
        let module = device.create_shader_module(wgpu::include_wgsl!("equirectangular.wgsl"));
        let texture_format = wgpu::TextureFormat::Rgba32Float;
        let equirect_layout = device.create_bind_group_layout(&wgpu::BindGroupLayoutDescriptor {
            label: Some("HdrLoader::equirect_layout"),
            entries: &[
                wgpu::BindGroupLayoutEntry {
                    binding: 0,
                    visibility: wgpu::ShaderStages::COMPUTE,
                    ty: wgpu::BindingType::Texture {
                        sample_type: wgpu::TextureSampleType::Float { filterable: false },
                        view_dimension: wgpu::TextureViewDimension::D2,
                        multisampled: false,
                    },
                    count: None,
                },
                wgpu::BindGroupLayoutEntry {
                    binding: 1,
                    visibility: wgpu::ShaderStages::COMPUTE,
                    ty: wgpu::BindingType::StorageTexture {
                        access: wgpu::StorageTextureAccess::WriteOnly,
                        format: texture_format,
                        view_dimension: wgpu::TextureViewDimension::D2Array,
                    },
                    count: None,
                },
            ],
        });

        let pipeline_layout = device.create_pipeline_layout(&wgpu::PipelineLayoutDescriptor {
            label: None,
            bind_group_layouts: &[&equirect_layout],
            push_constant_ranges: &[],
        });

        let equirect_to_cubemap =
            device.create_compute_pipeline(&wgpu::ComputePipelineDescriptor {
                label: Some("equirect_to_cubemap"),
                layout: Some(&pipeline_layout),
                module: &module,
                entry_point: Some("compute_equirect_to_cubemap"),
                compilation_options: Default::default(),
                cache: None,
            });

        Self {
            equirect_to_cubemap,
            texture_format,
            equirect_layout,
        }
    }

    pub fn from_equirectangular_bytes(
        &self,
        device: &wgpu::Device,
        queue: &wgpu::Queue,
        data: &[u8],
        dst_size: u32,
        label: Option<&str>,
    ) -> anyhow::Result<texture::CubeTexture> {
        let hdr_decoder = HdrDecoder::new(Cursor::new(data))?;
        let meta = hdr_decoder.metadata();
        
        #[cfg(not(target_arch="wasm32"))]
        let pixels = {
            let mut pixels = vec![[0.0, 0.0, 0.0, 0.0]; meta.width as usize * meta.height as usize];
            hdr_decoder.read_image_transform(
                |pix| {
                    let rgb = pix.to_hdr();
                    [rgb.0[0], rgb.0[1], rgb.0[2], 1.0f32]
                },
                &mut pixels[..],
            )?;
            pixels
        };
        #[cfg(target_arch="wasm32")]
        let pixels = hdr_decoder.read_image_native()?
            .into_iter()
            .map(|pix| {
                let rgb = pix.to_hdr();
                [rgb.0[0], rgb.0[1], rgb.0[2], 1.0f32]
            })
            .collect::<Vec<_>>();

        let src = texture::Texture::create_2d_texture(
            device,
            meta.width,
            meta.height,
            self.texture_format,
            wgpu::TextureUsages::TEXTURE_BINDING | wgpu::TextureUsages::COPY_DST,
            wgpu::FilterMode::Linear,
            None,
        );

        queue.write_texture(
            wgpu::TexelCopyTextureInfo {
                texture: &src.texture,
                mip_level: 0,
                origin: wgpu::Origin3d::ZERO,
                aspect: wgpu::TextureAspect::All,
            },
            &bytemuck::cast_slice(&pixels),
            wgpu::TexelCopyBufferLayout {
                offset: 0,
                bytes_per_row: Some(src.size.width * std::mem::size_of::<[f32; 4]>() as u32),
                rows_per_image: Some(src.size.height),
            },
            src.size,
        );

        let dst = texture::CubeTexture::create_2d(
            device,
            dst_size,
            dst_size,
            self.texture_format,
            1,
            // We are going to write to `dst` texture so we
            // need to use a `STORAGE_BINDING`.
            wgpu::TextureUsages::STORAGE_BINDING
                | wgpu::TextureUsages::TEXTURE_BINDING,
            wgpu::FilterMode::Nearest,
            label,
        );

        let dst_view = dst.texture().create_view(&wgpu::TextureViewDescriptor {
            label,
            // Normally, you'd use `TextureViewDimension::Cube`
            // for a cube texture, but we can't use that
            // view dimension with a `STORAGE_BINDING`.
            // We need to access the cube texture layers
            // directly.
            dimension: Some(wgpu::TextureViewDimension::D2Array),
            ..Default::default()
        });

        let bind_group = device.create_bind_group(&wgpu::BindGroupDescriptor {
            label,
            layout: &self.equirect_layout,
            entries: &[
                wgpu::BindGroupEntry {
                    binding: 0,
                    resource: wgpu::BindingResource::TextureView(&src.view),
                },
                wgpu::BindGroupEntry {
                    binding: 1,
                    resource: wgpu::BindingResource::TextureView(&dst_view),
                },
            ],
        });

        let mut encoder = device.create_command_encoder(&Default::default());
        let mut pass = encoder.begin_compute_pass(&wgpu::ComputePassDescriptor { label });

        let num_workgroups = (dst_size + 15) / 16;
        pass.set_pipeline(&self.equirect_to_cubemap);
        pass.set_bind_group(0, &bind_group, &[]);
        pass.dispatch_workgroups(num_workgroups, num_workgroups, 6);

        drop(pass);

        queue.submit([encoder.finish()]);

        Ok(dst)
    }
}
```

La llamada `dispatch_workgroups` le dice a la GPU que ejecute nuestro código en lotes llamados grupos de trabajo. Cada grupo de trabajo tiene un número de hilos de trabajo llamados invocaciones que ejecutan el código en paralelo. Los grupos de trabajo se organizan como una cuadrícula 3D con las dimensiones que pasamos a `dispatch_workgroups`.

En este ejemplo, tenemos una cuadrícula de grupos de trabajo dividida en fragmentos de 16x16 y almacenando la capa en la dimensión z.

## El shader de computación

Ahora, escribamos un shader de computación que convertirá nuestra textura equirectangular a una textura de cubo. Crea un archivo llamado `equirectangular.wgsl`. Vamos a desglosarlo fragmento por fragmento.

```wgsl
const PI: f32 = 3.1415926535897932384626433832795;

struct Face {
    forward: vec3<f32>,
    up: vec3<f32>,
    right: vec3<f32>,
}
```

Dos cosas aquí:

1. WGSL no tiene una función integrada para PI, así que necesitamos especificarlo nosotros mismos.
2. Cada cara del mapa de cubo tiene una orientación, así que necesitamos almacenarla.

```wgsl
@group(0)
@binding(0)
var src: texture_2d<f32>;

@group(0)
@binding(1)
var dst: texture_storage_2d_array<rgba32float, write>;
```

Aquí, tenemos los únicos dos enlaces que necesitamos. La textura equirectangular `src` y nuestra textura de cubo `dst`. Algunas cosas a tener en cuenta sobre `dst`:

1. Aunque `dst` es una textura de cubo, se almacena como una matriz de texturas 2D.
2. El tipo de enlace que estamos usando aquí es una textura de almacenamiento. Una textura de almacenamiento de matriz, para ser precisos. Este es un enlace único solo disponible para shaders de computación. Nos permite escribir directamente en la textura.
3. Cuando usamos un enlace de textura de almacenamiento, necesitamos especificar el formato de la textura. Si intentas vincular una textura con un formato diferente, wgpu entrará en pánico.

```wgsl
@compute
@workgroup_size(16, 16, 1)
fn compute_equirect_to_cubemap(
    @builtin(global_invocation_id)
    gid: vec3<u32>,
) {
    // If texture size is not divisible by 32, we
    // need to make sure we don't try to write to
    // pixels that don't exist.
    if gid.x >= u32(textureDimensions(dst).x) {
        return;
    }

    var FACES: array<Face, 6> = array(
        // FACES +X
        Face(
            vec3(1.0, 0.0, 0.0),  // forward
            vec3(0.0, 1.0, 0.0),  // up
            vec3(0.0, 0.0, -1.0), // right
        ),
        // FACES -X
        Face (
            vec3(-1.0, 0.0, 0.0),
            vec3(0.0, 1.0, 0.0),
            vec3(0.0, 0.0, 1.0),
        ),
        // FACES +Y
        Face (
            vec3(0.0, -1.0, 0.0),
            vec3(0.0, 0.0, 1.0),
            vec3(1.0, 0.0, 0.0),
        ),
        // FACES -Y
        Face (
            vec3(0.0, 1.0, 0.0),
            vec3(0.0, 0.0, -1.0),
            vec3(1.0, 0.0, 0.0),
        ),
        // FACES +Z
        Face (
            vec3(0.0, 0.0, 1.0),
            vec3(0.0, 1.0, 0.0),
            vec3(1.0, 0.0, 0.0),
        ),
        // FACES -Z
        Face (
            vec3(0.0, 0.0, -1.0),
            vec3(0.0, 1.0, 0.0),
            vec3(-1.0, 0.0, 0.0),
        ),
    );

    // Get texture coords relative to cubemap face
    let dst_dimensions = vec2<f32>(textureDimensions(dst));
    let cube_uv = vec2<f32>(gid.xy) / dst_dimensions * 2.0 - 1.0;

    // Get spherical coordinate from cube_uv
    let face = FACES[gid.z];
    let spherical = normalize(face.forward + face.right * cube_uv.x + face.up * cube_uv.y);

    // Get coordinate on the equirectangular texture
    let inv_atan = vec2(0.1591, 0.3183);
    let eq_uv = vec2(atan2(spherical.z, spherical.x), asin(spherical.y)) * inv_atan + 0.5;
    let eq_pixel = vec2<i32>(eq_uv * vec2<f32>(textureDimensions(src)));

    // We use textureLoad() as textureSample() is not allowed in compute shaders
    var sample = textureLoad(src, eq_pixel, 0);

    textureStore(dst, gid.xy, gid.z, sample);
}
```

Aunque comenté el código anterior, hay algunas cosas que quiero repasar que no se ajustarían bien en un comentario.

El decorador `workgroup_size` indica las dimensiones de la cuadrícula local de invocaciones del grupo de trabajo. Debido a que estamos enviando un grupo de trabajo para cada píxel en la textura, hacemos que cada grupo de trabajo sea una cuadrícula de 16x16x1. Esto significa que cada grupo de trabajo puede tener 256 hilos con los que trabajar.

<div class="warn">

Para WebGPU, cada grupo de trabajo puede tener un máximo de 256 hilos (también llamados invocaciones).

</div>

Con esto, podemos cargar el mapa de ambiente en la función `new()`:

```rust
let hdr_loader = resources::HdrLoader::new(&device);
let sky_bytes = resources::load_binary("pure-sky.hdr").await?;
let sky_texture = hdr_loader.from_equirectangular_bytes(
    &device,
    &queue,
    &sky_bytes,
    1080,
    Some("Sky Texture"),
)?;
```

## Cielo (Skybox)

Ahora que tenemos un mapa de ambiente para renderizar, usémoslo para hacer nuestro cielo. Hay diferentes formas de renderizar un cielo. Una forma estándar es renderizar un cubo y mapear el mapa de ambiente en él. Si bien ese método funciona, puede tener algunos artefactos en las esquinas y bordes donde se encuentran las caras del cubo.

En cambio, vamos a renderizar en toda la pantalla, calcular la dirección de vista de cada píxel y usarla para muestrear la textura. Primero, necesitamos crear un grupo de enlace para el mapa de ambiente para que podamos usarlo para renderizar. Agrega lo siguiente a `new()`:

```rust
let environment_layout =
    device.create_bind_group_layout(&wgpu::BindGroupLayoutDescriptor {
        label: Some("environment_layout"),
        entries: &[
            wgpu::BindGroupLayoutEntry {
                binding: 0,
                visibility: wgpu::ShaderStages::FRAGMENT,
                ty: wgpu::BindingType::Texture {
                    sample_type: wgpu::TextureSampleType::Float { filterable: false },
                    view_dimension: wgpu::TextureViewDimension::Cube,
                    multisampled: false,
                },
                count: None,
            },
            wgpu::BindGroupLayoutEntry {
                binding: 1,
                visibility: wgpu::ShaderStages::FRAGMENT,
                ty: wgpu::BindingType::Sampler(wgpu::SamplerBindingType::NonFiltering),
                count: None,
            },
        ],
    });

let environment_bind_group = device.create_bind_group(&wgpu::BindGroupDescriptor {
    label: Some("environment_bind_group"),
    layout: &environment_layout,
    entries: &[
        wgpu::BindGroupEntry {
            binding: 0,
            resource: wgpu::BindingResource::TextureView(&sky_texture.view()),
        },
        wgpu::BindGroupEntry {
            binding: 1,
            resource: wgpu::BindingResource::Sampler(sky_texture.sampler()),
        },
    ],
});
```

Ahora que tenemos el grupo de enlace, necesitamos una tubería de renderizado para renderizar el cielo.

```rust
// NUEVO!
let sky_pipeline = {
    let layout = device.create_pipeline_layout(&wgpu::PipelineLayoutDescriptor {
        label: Some("Sky Pipeline Layout"),
        bind_group_layouts: &[&camera_bind_group_layout, &environment_layout],
        push_constant_ranges: &[],
    });
    let shader = wgpu::include_wgsl!("sky.wgsl");
    create_render_pipeline(
        &device,
        &layout,
        hdr.format(),
        Some(texture::Texture::DEPTH_FORMAT),
        &[],
        wgpu::PrimitiveTopology::TriangleList,
        shader,
    )
};
```

Una cosa a tener en cuenta aquí. Agregamos el formato primitivo a `create_render_pipeline()`. Además, cambiamos la función de comparación de profundidad a `CompareFunction::LessEqual` (discutiremos por qué cuando repasemos el shader del cielo). Aquí están los cambios en eso:

```rust
fn create_render_pipeline(
    device: &wgpu::Device,
    layout: &wgpu::PipelineLayout,
    color_format: wgpu::TextureFormat,
    depth_format: Option<wgpu::TextureFormat>,
    vertex_layouts: &[wgpu::VertexBufferLayout],
    topology: wgpu::PrimitiveTopology, // NEW!
    shader: wgpu::ShaderModuleDescriptor,
) -> wgpu::RenderPipeline {
    let shader = device.create_shader_module(shader);

    device.create_render_pipeline(&wgpu::RenderPipelineDescriptor {
        // ...
        primitive: wgpu::PrimitiveState {
            topology, // NEW!
            // ...
        },
        depth_stencil: depth_format.map(|format| wgpu::DepthStencilState {
            format,
            depth_write_enabled: true,
            depth_compare: wgpu::CompareFunction::LessEqual, // UDPATED!
            stencil: wgpu::StencilState::default(),
            bias: wgpu::DepthBiasState::default(),
        }),
        // ...
    })
}
```

No olvides agregar el nuevo grupo de enlace y tubería a `State`.

```rust
pub struct State {
    // ...
    // NUEVO!
    hdr: hdr::HdrPipeline,
    environment_bind_group: wgpu::BindGroup,
    sky_pipeline: wgpu::RenderPipeline,
}

// ...
impl State {
    async fn new(window: Arc<Window>) -> anyhow::Result<State> {
        // ...
        Ok(Self {
            // ...
            // NUEVO!
            hdr,
            environment_bind_group,
            sky_pipeline,
            debug,
        })
    }
}
```

Ahora cubramos `sky.wgsl`.

```wgsl
struct Camera {
    view_pos: vec4<f32>,
    view: mat4x4<f32>,
    view_proj: mat4x4<f32>,
    inv_proj: mat4x4<f32>,
    inv_view: mat4x4<f32>,
}
@group(0) @binding(0)
var<uniform> camera: Camera;

@group(1)
@binding(0)
var env_map: texture_cube<f32>;
@group(1)
@binding(1)
var env_sampler: sampler;

struct VertexOutput {
    @builtin(position) frag_position: vec4<f32>,
    @location(0) clip_position: vec4<f32>,
}

@vertex
fn vs_main(
    @builtin(vertex_index) id: u32,
) -> VertexOutput {
    let uv = vec2<f32>(vec2<u32>(
        id & 1u,
        (id >> 1u) & 1u,
    ));
    var out: VertexOutput;
    // out.clip_position = vec4(uv * vec2(4.0, -4.0) + vec2(-1.0, 1.0), 0.0, 1.0);
    out.clip_position = vec4(uv * 4.0 - 1.0, 1.0, 1.0);
    out.frag_position = vec4(uv * 4.0 - 1.0, 1.0, 1.0);
    return out;
}

@fragment
fn fs_main(in: VertexOutput) -> @location(0) vec4<f32> {
    let view_pos_homogeneous = camera.inv_proj * in.clip_position;
    let view_ray_direction = view_pos_homogeneous.xyz / view_pos_homogeneous.w;
    var ray_direction = normalize((camera.inv_view * vec4(view_ray_direction, 0.0)).xyz);

    let sample = textureSample(env_map, env_sampler, ray_direction);
    return sample;
}
```

Desglosemos esto:

1. Creamos un triángulo del doble del tamaño de la pantalla.
2. En el shader de fragmento, obtenemos la dirección de vista de la posición de clip. Usamos la matriz de proyección inversa para convertir las coordenadas de clip a dirección de vista. Luego, usamos la matriz de vista inversa para obtener la dirección en el espacio mundial, ya que eso es lo que necesitamos para muestrear el cielo correctamente.
3. Luego muestreamos la textura del cielo con la dirección de vista.

<!-- ![debugging skybox](./debugging-skybox.png) -->

Para que esto funcione, necesitamos cambiar un poco nuestros uniformes de cámara. Necesitamos agregar la matriz de vista inversa y la matriz de proyección inversa a la estructura `CameraUniform`.

```rust
#[repr(C)]
#[derive(Copy, Clone, bytemuck::Pod, bytemuck::Zeroable)]
struct CameraUniform {
    view_position: [f32; 4],
    view: [[f32; 4]; 4], // NEW!
    view_proj: [[f32; 4]; 4],
    inv_proj: [[f32; 4]; 4], // NEW!
    inv_view: [[f32; 4]; 4], // NEW!
}

impl CameraUniform {
    fn new() -> Self {
        Self {
            view_position: [0.0; 4],
            view: cgmath::Matrix4::identity().into(),
            view_proj: cgmath::Matrix4::identity().into(),
            inv_proj: cgmath::Matrix4::identity().into(), // NEW!
            inv_view: cgmath::Matrix4::identity().into(), // NEW!
        }
    }

    // UPDATED!
    fn update_view_proj(&mut self, camera: &camera::Camera, projection: &camera::Projection) {
        self.view_position = camera.position.to_homogeneous().into();
        let proj = projection.calc_matrix();
        let view = camera.calc_matrix();
        let view_proj = proj * view;
        self.view = view.into();
        self.view_proj = view_proj.into();
        self.inv_proj = proj.invert().unwrap().into();
        self.inv_view = view.transpose().into();
    }
}
```

Asegúrate de cambiar la definición de `Camera` en `shader.wgsl` y `light.wgsl`. Como recordatorio, se ve así:

```wgsl
struct Camera {
    view_pos: vec4<f32>,
    view: mat4x4<f32>,
    view_proj: mat4x4<f32>,
    inv_proj: mat4x4<f32>,
    inv_view: mat4x4<f32>,
}
var<uniform> camera: Camera;
```

<div class="info">

Quizás hayas notado que quitamos la `OPENGL_TO_WGPU_MATRIX`. La razón es que estaba interfiriendo con la proyección del cielo.

![projection error](./project-error.png)

Técnicamente, no era necesaria, así que me pareció bien eliminarla.

</div>

## Reflejos

Ahora que tenemos un cielo, podemos experimentar usándolo para iluminación. Esto no será físicamente preciso (lo veremos más adelante). Dicho esto, tenemos el mapa de ambiente, así que también podemos usarlo.

Para hacer eso, sin embargo, necesitamos cambiar nuestro shader para hacer iluminación en el espacio mundial en lugar de espacio tangente porque nuestro mapa de ambiente está en el espacio mundial. Debido a que hay muchos cambios, publicaré el shader completo aquí:

```wgsl
// Vertex shader

struct Camera {
    view_pos: vec4<f32>,
    view: mat4x4<f32>,
    view_proj: mat4x4<f32>,
    inv_proj: mat4x4<f32>,
    inv_view: mat4x4<f32>,
}
@group(0) @binding(0)
var<uniform> camera: Camera;

struct Light {
    position: vec3<f32>,
    color: vec3<f32>,
}
@group(2) @binding(0)
var<uniform> light: Light;

struct VertexInput {
    @location(0) position: vec3<f32>,
    @location(1) tex_coords: vec2<f32>,
    @location(2) normal: vec3<f32>,
    @location(3) tangent: vec3<f32>,
    @location(4) bitangent: vec3<f32>,
}
struct InstanceInput {
    @location(5) model_matrix_0: vec4<f32>,
    @location(6) model_matrix_1: vec4<f32>,
    @location(7) model_matrix_2: vec4<f32>,
    @location(8) model_matrix_3: vec4<f32>,
    @location(9) normal_matrix_0: vec3<f32>,
    @location(10) normal_matrix_1: vec3<f32>,
    @location(11) normal_matrix_2: vec3<f32>,
}

struct VertexOutput {
    @builtin(position) clip_position: vec4<f32>,
    @location(0) tex_coords: vec2<f32>,
    // Updated!
    @location(1) world_position: vec3<f32>,
    @location(2) world_view_position: vec3<f32>,
    @location(3) world_light_position: vec3<f32>,
    @location(4) world_normal: vec3<f32>,
    @location(5) world_tangent: vec3<f32>,
    @location(6) world_bitangent: vec3<f32>,
}

@vertex
fn vs_main(
    model: VertexInput,
    instance: InstanceInput,
) -> VertexOutput {
    let model_matrix = mat4x4<f32>(
        instance.model_matrix_0,
        instance.model_matrix_1,
        instance.model_matrix_2,
        instance.model_matrix_3,
    );
    let normal_matrix = mat3x3<f32>(
        instance.normal_matrix_0,
        instance.normal_matrix_1,
        instance.normal_matrix_2,
    );

    // UPDATED!
    let world_position = model_matrix * vec4<f32>(model.position, 1.0);

    var out: VertexOutput;
    out.clip_position = camera.view_proj * world_position;
    out.tex_coords = model.tex_coords;
    out.world_normal = normalize(normal_matrix * model.normal);
    out.world_tangent = normalize(normal_matrix * model.tangent);
    out.world_bitangent = normalize(normal_matrix * model.bitangent);
    out.world_position = world_position.xyz;
    out.world_view_position = camera.view_pos.xyz;
    return out;
}

// Fragment shader

@group(0) @binding(0)
var t_diffuse: texture_2d<f32>;
@group(0)@binding(1)
var s_diffuse: sampler;
@group(0)@binding(2)
var t_normal: texture_2d<f32>;
@group(0) @binding(3)
var s_normal: sampler;

@group(3)
@binding(0)
var env_map: texture_cube<f32>;
@group(3)
@binding(1)
var env_sampler: sampler;

@fragment
fn fs_main(in: VertexOutput) -> @location(0) vec4<f32> {
    let object_color: vec4<f32> = textureSample(t_diffuse, s_diffuse, in.tex_coords);
    let object_normal: vec4<f32> = textureSample(t_normal, s_normal, in.tex_coords);

    // NEW!
    // Adjust the tangent and bitangent using the Gramm-Schmidt process
    // This makes sure that they are perpendicular to each other and the
    // normal of the surface.
    let world_tangent = normalize(in.world_tangent - dot(in.world_tangent, in.world_normal) * in.world_normal);
    let world_bitangent = cross(world_tangent, in.world_normal);

    // Convert the normal sample to world space
    let TBN = mat3x3(
        world_tangent,
        world_bitangent,
        in.world_normal,
    );
    let tangent_normal = object_normal.xyz * 2.0 - 1.0;
    let world_normal = TBN * tangent_normal;

    // Create the lighting vectors
    let light_dir = normalize(light.position - in.world_position);
    let view_dir = normalize(in.world_view_position - in.world_position);
    let half_dir = normalize(view_dir + light_dir);

    let diffuse_strength = max(dot(world_normal, light_dir), 0.0);
    let diffuse_color = light.color * diffuse_strength;

    let specular_strength = pow(max(dot(world_normal, half_dir), 0.0), 32.0);
    let specular_color = specular_strength * light.color;

    // NEW!
    // Calculate reflections
    let world_reflect = reflect(-view_dir, world_normal);
    let reflection = textureSample(env_map, env_sampler, world_reflect).rgb;
    let shininess = 0.1;

    let result = (diffuse_color + specular_color) * object_color.xyz + reflection * shininess;

    return vec4<f32>(result, object_color.a);
}
```

Una pequeña nota sobre la matemática de la reflexión. El `view_dir` nos da la dirección hacia la cámara desde la superficie. La matemática de reflexión necesita la dirección desde la cámara hacia la superficie, así que negamos `view_dir`. Luego usamos la función integrada `reflect` de `wgsl` para reflejar el `view_dir` invertido sobre `world_normal`. Esto nos da una dirección que podemos usar para muestrear el mapa de ambiente y obtener el color del cielo en esa dirección. Solo mirando el componente de reflexión nos da lo siguiente:

![just-reflections](./just-reflections.png)

Aquí está la escena terminada:

![with-reflections](./with-reflections.png)

## ¿La salida demasiado oscura en WebGPU?

WebGPU no admite el uso de formatos de textura sRGB como
la salida de una superficie. Podemos contornear esto haciendo
que la vista de textura utilizada para renderizar use la versión
sRGB del formato. Para hacer esto, necesitamos cambiar la
configuración de superficie que usamos para permitir formatos
de vista con sRGB.

```rust
let config = wgpu::SurfaceConfiguration {
    usage: wgpu::TextureUsages::RENDER_ATTACHMENT,
    format: surface_format,
    width: size.width,
    height: size.height,
    present_mode: surface_caps.present_modes[0],
    alpha_mode: surface_caps.alpha_modes[0],
    // NUEVO!
    view_formats: vec![surface_format.add_srgb_suffix()],
    desired_maximum_frame_latency: 2,
};
```

Luego, necesitamos crear una vista con sRGB habilitado en
`State::render()`.

```rust
let view = output
    .texture
    .create_view(&wgpu::TextureViewDescriptor {
        format: Some(self.config.format.add_srgb_suffix()),
        ..Default::default()
    });
```

Quizás también hayas notado que en `HdrPipeline::new()`
usamos `config.format.add_srgb_suffix()` cuando creamos
la tubería de renderizado. Esto es obligatorio porque si no lo
hacemos, la `TextureView` habilitada para sRGB no funcionará
con la tubería de renderizado.

Con eso deberías obtener la salida sRGB como se esperaba.

## Demo

<div class="warn">

Si tu navegador no soporta WebGPU, este ejemplo no funcionará para ti.

</div>

<WasmExample example="tutorial13_hdr"></WasmExample>

<AutoGithubLink/>
