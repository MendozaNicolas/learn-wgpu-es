# Texturas y grupos de enlace

Hasta ahora, hemos estado dibujando formas muy simples. Si bien podemos hacer un juego solo con triángulos, intentar dibujar objetos altamente detallados limitaría masivamente qué dispositivos podrían ejecutar nuestro juego. Sin embargo, podemos evitar este problema con **texturas**.

Las texturas son imágenes superpuestas en una malla de triángulos para que parezca más detallada. Hay varios tipos de texturas, como mapas normales, mapas de relieve, mapas especulares y mapas difusos. Vamos a hablar de mapas difusos o, más simplemente, la textura de color.

## Cargando una imagen desde un archivo

Si queremos mapear una imagen a nuestra malla, primero necesitamos una imagen. Usemos este pequeño árbol feliz:

![a happy tree](./happy-tree.png)

Usaremos el [crate de image](https://docs.rs/image) para cargar nuestro árbol. Agreguémoslo a nuestras dependencias:

```toml
[dependencies.image]
version = "0.24"
default-features = false
features = ["png", "jpeg"]
```

El decodificador de jpeg que incluye `image` usa [rayon](https://docs.rs/rayon) para acelerar la decodificación con hilos. WASM no admite hilos actualmente, así que necesitamos deshabilitarlo para que nuestro código no se bloquee cuando intentemos cargar un jpeg en la web.

<div class="note">

Decodificar jpgs en WASM no es muy eficiente. Si deseas acelerar la carga de imágenes en general en WASM, podrías optar por usar los decodificadores integrados del navegador en lugar de `image` al compilar con `wasm-bindgen`. Esto implicará crear una etiqueta `<img>` en Rust para obtener la imagen y luego un `<canvas>` para obtener los datos de píxeles, pero dejaré esto como un ejercicio para el lector.

</div>

En el método `new()` de `State`, agrega lo siguiente justo después de declarar `config`:

```rust
let config = wgpu::SurfaceConfiguration {
    usage: wgpu::TextureUsages::RENDER_ATTACHMENT,
    format: surface_format,
    width: size.width,
    height: size.height,
    present_mode: surface_caps.present_modes[0],
    alpha_mode: surface_caps.alpha_modes[0],
    view_formats: vec![],
    desired_maximum_frame_latency: 2,
};
// NEW!

let diffuse_bytes = include_bytes!("happy-tree.png");
let diffuse_image = image::load_from_memory(diffuse_bytes).unwrap();
let diffuse_rgba = diffuse_image.to_rgba8();

use image::GenericImageView;
let dimensions = diffuse_image.dimensions();
```

Aquí, obtenemos los bytes del archivo de imagen y los cargamos en una imagen, que luego se convierte en un `Vec` de bytes RGBA. También guardamos las dimensiones de la imagen para cuando creemos la `Texture` real.

Ahora, creemos la `Texture`:

```rust
let texture_size = wgpu::Extent3d {
    width: dimensions.0,
    height: dimensions.1,
    // All textures are stored as 3D, we represent our 2D texture
    // by setting depth to 1.
    depth_or_array_layers: 1,
};
let diffuse_texture = device.create_texture(
    &wgpu::TextureDescriptor {
        size: texture_size,
        mip_level_count: 1, // We'll talk about this a little later
        sample_count: 1,
        dimension: wgpu::TextureDimension::D2,
        // Most images are stored using sRGB, so we need to reflect that here.
        format: wgpu::TextureFormat::Rgba8UnormSrgb,
        // TEXTURE_BINDING tells wgpu that we want to use this texture in shaders
        // COPY_DST means that we want to copy data to this texture
        usage: wgpu::TextureUsages::TEXTURE_BINDING | wgpu::TextureUsages::COPY_DST,
        label: Some("diffuse_texture"),
        // This is the same as with the SurfaceConfig. It
        // specifies what texture formats can be used to
        // create TextureViews for this texture. The base
        // texture format (Rgba8UnormSrgb in this case) is
        // always supported. Note that using a different
        // texture format is not supported on the WebGL2
        // backend.
        view_formats: &[],
    }
);
```

## Cargando datos en una Textura

La estructura `Texture` no tiene métodos para interactuar con los datos directamente. Sin embargo, podemos usar un método en la `queue` que creamos anteriormente llamado `write_texture` para cargar la textura. Veamos cómo lo hacemos:

```rust
queue.write_texture(
    // Tells wgpu where to copy the pixel data
    wgpu::TexelCopyTextureInfo {
        texture: &diffuse_texture,
        mip_level: 0,
        origin: wgpu::Origin3d::ZERO,
        aspect: wgpu::TextureAspect::All,
    },
    // The actual pixel data
    &diffuse_rgba,
    // The layout of the texture
    wgpu::TexelCopyBufferLayout {
        offset: 0,
        bytes_per_row: Some(4 * dimensions.0),
        rows_per_image: Some(dimensions.1),
    },
    texture_size,
);
```

<div class="note">

La forma antigua de escribir datos en una textura era copiar los datos de píxeles a un búfer y luego copiarlo a la textura. Usar `write_texture` es un poco más eficiente ya que usa un búfer menos - Lo dejaré aquí de todas formas, en caso de que lo necesites.

```rust
let buffer = device.create_buffer_init(
    &wgpu::util::BufferInitDescriptor {
        label: Some("Temp Buffer"),
        contents: &diffuse_rgba,
        usage: wgpu::BufferUsages::COPY_SRC,
    }
);

let mut encoder = device.create_command_encoder(&wgpu::CommandEncoderDescriptor {
    label: Some("texture_buffer_copy_encoder"),
});

encoder.copy_buffer_to_texture(
    wgpu::TexelCopyBufferInfo {
        buffer: &buffer,
        offset: 0,
        bytes_per_row: 4 * dimensions.0,
        rows_per_image: dimensions.1,
    },
    wgpu::TexelCopyTextureInfo {
        texture: &diffuse_texture,
        mip_level: 0,
        array_layer: 0,
        origin: wgpu::Origin3d::ZERO,
    },
    is_surface_configured: false,
);

queue.submit(std::iter::once(encoder.finish()));
```

El campo `bytes_per_row` necesita consideración. Este valor debe ser un múltiplo de 256. Echa un vistazo a [el tutorial de gif](../../showcase/gifs/#how-do-we-make-the-frames) para más detalles.

</div>

## Vistas de Textura y Muestreadores

Ahora que nuestra textura tiene datos en ella, necesitamos una forma de usarla. Aquí es donde entra en juego un `TextureView` y un `Sampler`. Un `TextureView` nos ofrece una *vista* en nuestra textura. Un `Sampler` controla cómo se *muestrea* la `Texture`. El muestreo funciona similar a la herramienta cuentagotas en GIMP/Photoshop. Nuestro programa proporciona una coordenada en la textura (conocida como *coordenada de textura*), y luego el muestreador devuelve el color correspondiente basado en la textura y algunos parámetros internos.

Definamos ahora nuestro `diffuse_texture_view` y `diffuse_sampler`:

```rust
// We don't need to configure the texture view much, so let's
// let wgpu define it.
let diffuse_texture_view = diffuse_texture.create_view(&wgpu::TextureViewDescriptor::default());
let diffuse_sampler = device.create_sampler(&wgpu::SamplerDescriptor {
    address_mode_u: wgpu::AddressMode::ClampToEdge,
    address_mode_v: wgpu::AddressMode::ClampToEdge,
    address_mode_w: wgpu::AddressMode::ClampToEdge,
    mag_filter: wgpu::FilterMode::Linear,
    min_filter: wgpu::FilterMode::Nearest,
    mipmap_filter: wgpu::FilterMode::Nearest,
    ..Default::default()
});
```

Los parámetros `address_mode_*` determinan qué hacer si el muestreador obtiene una coordenada de textura fuera de la textura misma. Tenemos algunas opciones para elegir:

* `ClampToEdge`: Cualquier coordenada de textura fuera de la textura devolverá el color del píxel más cercano en los bordes de la textura.
* `Repeat`: La textura se repetirá cuando las coordenadas de textura excedan las dimensiones de la textura.
* `MirrorRepeat`: Similar a `Repeat`, pero la imagen se volteará cuando cruze límites.

![address_mode.png](./address_mode.png)

Los campos `mag_filter` y `min_filter` describen qué hacer cuando la huella de muestra es menor o mayor que un texel. Estos dos campos generalmente funcionan cuando el mapeo en la escena está lejos o cerca de la cámara.

Hay dos opciones:
* `Linear`: Selecciona dos texels en cada dimensión y devuelve una interpolación lineal entre sus valores.
* `Nearest`: Devuelve el valor del texel más cercano a las coordenadas de textura. Esto crea una imagen que se ve más nítida desde lejos pero pixelada de cerca. Sin embargo, esto puede ser deseable si tus texturas están diseñadas para ser pixeladas, como en juegos de arte de píxeles o juegos de vóxeles como Minecraft.

Los mipmaps son un tema complejo y requerirán su propia sección en el futuro. Por ahora, podemos decir que las funciones `mipmap_filter` son similares a `(mag/min)_filter` ya que indica al muestreador cómo mezclar entre mipmaps.

Estoy usando algunos valores predeterminados para los otros campos. Si deseas ver cuáles son, consulta [la documentación de wgpu](https://docs.rs/wgpu/latest/wgpu/type.SamplerDescriptor.html).

Todos estos recursos diferentes están bien, pero no nos sirven de mucho si no podemos conectarlos en ningún lado. Aquí es donde entran en juego los `BindGroup`s y los `PipelineLayout`s.

## El BindGroup

Un `BindGroup` describe un conjunto de recursos y cómo pueden ser accedidos por un sombreador. Creamos un `BindGroup` usando un `BindGroupLayout`. Primero, creemos uno de esos.

```rust
let texture_bind_group_layout =
            device.create_bind_group_layout(&wgpu::BindGroupLayoutDescriptor {
                entries: &[
                    wgpu::BindGroupLayoutEntry {
                        binding: 0,
                        visibility: wgpu::ShaderStages::FRAGMENT,
                        ty: wgpu::BindingType::Texture {
                            multisampled: false,
                            view_dimension: wgpu::TextureViewDimension::D2,
                            sample_type: wgpu::TextureSampleType::Float { filterable: true },
                        },
                        count: None,
                    },
                    wgpu::BindGroupLayoutEntry {
                        binding: 1,
                        visibility: wgpu::ShaderStages::FRAGMENT,
                        // This should match the filterable field of the
                        // corresponding Texture entry above.
                        ty: wgpu::BindingType::Sampler(wgpu::SamplerBindingType::Filtering),
                        count: None,
                    },
                ],
                label: Some("texture_bind_group_layout"),
            });
```

Nuestro `texture_bind_group_layout` tiene dos entradas: una para una textura muestreada en enlace 0 y una para un muestreador en enlace 1. Ambos enlaces son visibles solo para el sombreador de fragmentos como se especifica en `FRAGMENT`. Los valores posibles para este campo son cualquier combinación bit a bit de `NONE`, `VERTEX`, `FRAGMENT` o `COMPUTE`. La mayoría de las veces, solo usaremos `FRAGMENT` para texturas y muestreadores, pero es bueno saber qué más está disponible.

Con `texture_bind_group_layout`, ahora podemos crear nuestro `BindGroup`:

```rust
let diffuse_bind_group = device.create_bind_group(
    &wgpu::BindGroupDescriptor {
        layout: &texture_bind_group_layout,
        entries: &[
            wgpu::BindGroupEntry {
                binding: 0,
                resource: wgpu::BindingResource::TextureView(&diffuse_texture_view),
            },
            wgpu::BindGroupEntry {
                binding: 1,
                resource: wgpu::BindingResource::Sampler(&diffuse_sampler),
            }
        ],
        label: Some("diffuse_bind_group"),
    }
);
```

Mirando esto, ¡podrías tener un poco de déjà vu! Eso es porque un `BindGroup` es una declaración más específica del `BindGroupLayout`. La razón por la que están separados es que nos permite cambiar los `BindGroup`s sobre la marcha, siempre que todos compartan el mismo `BindGroupLayout`. Cada textura y muestreador que creamos deberá ser agregado a un `BindGroup`. Para nuestros propósitos, crearemos un nuevo grupo de enlace para cada textura.

Ahora que tenemos nuestro `diffuse_bind_group`, agreguémoslo a nuestra estructura `State`:

```rust
pub struct State {
    surface: wgpu::Surface<'static>,
    device: wgpu::Device,
    queue: wgpu::Queue,
    config: wgpu::SurfaceConfiguration,
    is_surface_configured: bool,
    window: &'a wgpu::Window,
    render_pipeline: wgpu::RenderPipeline,
    vertex_buffer: wgpu::Buffer,
    index_buffer: wgpu::Buffer,
    num_indices: u32,
    diffuse_bind_group: wgpu::BindGroup, // NEW!
}
```

Asegúrate de devolver estos campos en el método `new`:

```rust
impl State {
    async fn new() -> Self {
        // ...
        Self {
            surface,
            device,
            queue,
            config,
            is_surface_configured: false,
            window,
            render_pipeline,
            vertex_buffer,
            index_buffer,
            num_indices,
            // NEW!
            diffuse_bind_group,
        }
    }
}
```

Ahora que tenemos nuestro `BindGroup`, podemos usarlo en nuestra función `render()`.

```rust
// render()
// ...
render_pass.set_pipeline(&self.render_pipeline);
render_pass.set_bind_group(0, &self.diffuse_bind_group, &[]); // NEW!
render_pass.set_vertex_buffer(0, self.vertex_buffer.slice(..));
render_pass.set_index_buffer(self.index_buffer.slice(..), wgpu::IndexFormat::Uint16);

render_pass.draw_indexed(0..self.num_indices, 0, 0..1);
```

## PipelineLayout

¿Recuerdas el `PipelineLayout` que creamos hace tiempo en [la sección de tubería](/learn-wgpu/beginner/tutorial3-pipeline#how-do-we-use-the-shaders)? ¡Ahora finalmente llegamos a usarlo! El `PipelineLayout` contiene una lista de `BindGroupLayout`s que la tubería puede usar. Modifica `render_pipeline_layout` para usar nuestro `texture_bind_group_layout`.

```rust
async fn new(...) {
    // ...
    let render_pipeline_layout = device.create_pipeline_layout(
        &wgpu::PipelineLayoutDescriptor {
            label: Some("Render Pipeline Layout"),
            bind_group_layouts: &[&texture_bind_group_layout], // NEW!
            push_constant_ranges: &[],
        }
    );
    // ...
}
```

## Un cambio en los VÉRTICES
Hay algunas cosas que necesitamos cambiar sobre nuestra definición de `Vertex`. Hasta ahora, hemos estado usando un atributo `color` para establecer el color de nuestra malla. Ahora que estamos usando una textura, queremos reemplazar nuestro `color` con `tex_coords`. Estas coordenadas luego serán pasadas al `Sampler` para recuperar el color apropiado.

Como nuestras `tex_coords` son bidimensionales, cambiaremos el campo para tomar dos flotantes en lugar de tres.

Primero, cambiaremos la estructura `Vertex`:

```rust
#[repr(C)]
#[derive(Copy, Clone, Debug, bytemuck::Pod, bytemuck::Zeroable)]
struct Vertex {
    position: [f32; 3],
    tex_coords: [f32; 2], // NEW!
}
```

Y luego refleja estos cambios en el `VertexBufferLayout`:

```rust
impl Vertex {
    fn desc() -> wgpu::VertexBufferLayout<'static> {
        use std::mem;
        wgpu::VertexBufferLayout {
            array_stride: mem::size_of::<Vertex>() as wgpu::BufferAddress,
            step_mode: wgpu::VertexStepMode::Vertex,
            attributes: &[
                wgpu::VertexAttribute {
                    offset: 0,
                    shader_location: 0,
                    format: wgpu::VertexFormat::Float32x3,
                },
                wgpu::VertexAttribute {
                    offset: mem::size_of::<[f32; 3]>() as wgpu::BufferAddress,
                    shader_location: 1,
                    format: wgpu::VertexFormat::Float32x2, // NEW!
                },
            ]
        }
    }
}
```

Por último, necesitamos cambiar `VERTICES` en sí. Reemplaza la definición existente con la siguiente:

```rust
// Changed
const VERTICES: &[Vertex] = &[
    Vertex { position: [-0.0868241, 0.49240386, 0.0], tex_coords: [0.4131759, 0.99240386], }, // A
    Vertex { position: [-0.49513406, 0.06958647, 0.0], tex_coords: [0.0048659444, 0.56958647], }, // B
    Vertex { position: [-0.21918549, -0.44939706, 0.0], tex_coords: [0.28081453, 0.05060294], }, // C
    Vertex { position: [0.35966998, -0.3473291, 0.0], tex_coords: [0.85967, 0.1526709], }, // D
    Vertex { position: [0.44147372, 0.2347359, 0.0], tex_coords: [0.9414737, 0.7347359], }, // E
];
```

## Hora del Sombreador

Con nuestra nueva estructura `Vertex` en su lugar, es hora de actualizar nuestros sombreadores. Primero necesitaremos pasar nuestras `tex_coords` al sombreador de vértices y luego usarlas en nuestro sombreador de fragmentos para obtener el color final del `Sampler`. Empecemos con el sombreador de vértices:

```wgsl
// Vertex shader

struct VertexInput {
    @location(0) position: vec3<f32>,
    @location(1) tex_coords: vec2<f32>,
}

struct VertexOutput {
    @builtin(position) clip_position: vec4<f32>,
    @location(0) tex_coords: vec2<f32>,
}

@vertex
fn vs_main(
    model: VertexInput,
) -> VertexOutput {
    var out: VertexOutput;
    out.tex_coords = model.tex_coords;
    out.clip_position = vec4<f32>(model.position, 1.0);
    return out;
}
```

Ahora que tenemos nuestro sombreador de vértices generando nuestras `tex_coords`, necesitamos cambiar el sombreador de fragmentos para tomarlas. Con estas coordenadas, finalmente podremos usar nuestro muestreador para obtener un color de nuestra textura.

```wgsl
// Fragment shader

@group(0) @binding(0)
var t_diffuse: texture_2d<f32>;
@group(0) @binding(1)
var s_diffuse: sampler;

@fragment
fn fs_main(in: VertexOutput) -> @location(0) vec4<f32> {
    return textureSample(t_diffuse, s_diffuse, in.tex_coords);
}
```

Las variables `t_diffuse` y `s_diffuse` son lo que se conoce como uniformes. Hablaremos más sobre uniformes en la [sección de cámaras](/beginner/tutorial6-uniforms/). Por ahora, todo lo que necesitamos saber es que `group()` corresponde al 1er parámetro en `set_bind_group()` y `binding()` se relaciona con el `binding` especificado cuando creamos el `BindGroupLayout` y `BindGroup`.

## Los resultados

Si ejecutamos nuestro programa ahora, deberíamos obtener el siguiente resultado:

![an upside down tree on a pentagon](./upside-down.png)

¡Qué raro! ¡Nuestro árbol está al revés! Esto es porque las coordenadas del mundo de wgpu tienen el eje y apuntando hacia arriba, mientras que las coordenadas de textura tienen el eje y apuntando hacia abajo. En otras palabras, (0, 0) en coordenadas de textura corresponde a la esquina superior izquierda de la imagen, mientras que (1, 1) es la esquina inferior derecha.

![happy-tree-uv-coords.png](./happy-tree-uv-coords.png)

Podemos voltear nuestro triángulo correctamente reemplazando la coordenada y `y` de cada coordenada de textura con `1 - y`:

```rust
const VERTICES: &[Vertex] = &[
    // Changed
    Vertex { position: [-0.0868241, 0.49240386, 0.0], tex_coords: [0.4131759, 0.00759614], }, // A
    Vertex { position: [-0.49513406, 0.06958647, 0.0], tex_coords: [0.0048659444, 0.43041354], }, // B
    Vertex { position: [-0.21918549, -0.44939706, 0.0], tex_coords: [0.28081453, 0.949397], }, // C
    Vertex { position: [0.35966998, -0.3473291, 0.0], tex_coords: [0.85967, 0.84732914], }, // D
    Vertex { position: [0.44147372, 0.2347359, 0.0], tex_coords: [0.9414737, 0.2652641], }, // E
];
```

Con eso en su lugar, ahora tenemos nuestro árbol en la posición correcta en nuestro pentágono:

![our happy tree as it should be](./rightside-up.png)

## Limpiando las cosas

Para mayor comodidad, extraigamos nuestro código de textura en su propio módulo. Primero necesitamos agregar el [crate de anyhow](https://docs.rs/anyhow/) a nuestro archivo `Cargo.toml` para simplificar el manejo de errores;

```toml
[dependencies]
image = "0.23"
cgmath = "0.18"
winit = { version = "0.30", features = ["android-native-activity"] }
env_logger = "0.10"
log = "0.4"
pollster = "0.3"
wgpu = "27.0.0"
bytemuck = { version = "1.24", features = [ "derive" ] } # NEW!
```

Luego, en un nuevo archivo llamado `src/texture.rs`, agrega lo siguiente:

```rust
use image::GenericImageView;
use anyhow::*;

pub struct Texture {
    #[allow(unused)]
    pub texture: wgpu::Texture,
    pub view: wgpu::TextureView,
    pub sampler: wgpu::Sampler,
}

impl Texture {
    pub fn from_bytes(
        device: &wgpu::Device,
        queue: &wgpu::Queue,
        bytes: &[u8], 
        label: &str
    ) -> Result<Self> {
        let img = image::load_from_memory(bytes)?;
        Self::from_image(device, queue, &img, Some(label))
    }

    pub fn from_image(
        device: &wgpu::Device,
        queue: &wgpu::Queue,
        img: &image::DynamicImage,
        label: Option<&str>
    ) -> Result<Self> {
        let rgba = img.to_rgba8();
        let dimensions = img.dimensions();

        let size = wgpu::Extent3d {
            width: dimensions.0,
            height: dimensions.1,
            depth_or_array_layers: 1,
        };
        let texture = device.create_texture(
            &wgpu::TextureDescriptor {
                label,
                size,
                mip_level_count: 1,
                sample_count: 1,
                dimension: wgpu::TextureDimension::D2,
                format: wgpu::TextureFormat::Rgba8UnormSrgb,
                usage: wgpu::TextureUsages::TEXTURE_BINDING | wgpu::TextureUsages::COPY_DST,
                view_formats: &[],
            }
        );

        queue.write_texture(
            wgpu::TexelCopyTextureInfo {
                aspect: wgpu::TextureAspect::All,
                texture: &texture,
                mip_level: 0,
                origin: wgpu::Origin3d::ZERO,
            },
            &rgba,
            wgpu::TexelCopyBufferLayout {
                offset: 0,
                bytes_per_row: Some(4 * dimensions.0),
                rows_per_image: Some(dimensions.1),
            },
            size,
        );

        let view = texture.create_view(&wgpu::TextureViewDescriptor::default());
        let sampler = device.create_sampler(
            &wgpu::SamplerDescriptor {
                address_mode_u: wgpu::AddressMode::ClampToEdge,
                address_mode_v: wgpu::AddressMode::ClampToEdge,
                address_mode_w: wgpu::AddressMode::ClampToEdge,
                mag_filter: wgpu::FilterMode::Linear,
                min_filter: wgpu::FilterMode::Nearest,
                mipmap_filter: wgpu::FilterMode::Nearest,
                ..Default::default()
            }
        );

        Ok(Self { texture, view, sampler })
    }
}
```

<div class="note">

Observa que estamos usando `to_rgba8()` en lugar de `as_rgba8()`. Los PNGs funcionan bien con `as_rgba8()`, ya que tienen un canal alfa. Pero los JPEGs no tienen un canal alfa, y el código entraría en pánico si intentamos llamar a `as_rgba8()` en la imagen de textura JPEG que vamos a usar. En su lugar, podemos usar `to_rgba8()` para manejar tal imagen, que generará un nuevo búfer de imagen con un canal alfa incluso si la imagen original no tiene uno.

</div>

Necesitamos importar `texture.rs` como módulo, así que en la parte superior de `lib.rs` agrega lo siguiente.

```rust
mod texture;
```

El código de creación de textura en `new()` ahora se simplifica mucho:

```rust
let config = wgpu::SurfaceConfiguration {
    usage: wgpu::TextureUsages::RENDER_ATTACHMENT,
    format: surface_format,
    width: size.width,
    height: size.height,
    present_mode: surface_caps.present_modes[0],
    alpha_mode: surface_caps.alpha_modes[0],
    view_formats: vec![],
    desired_maximum_frame_latency: 2,
};

let diffuse_bytes = include_bytes!("happy-tree.png"); // CHANGED!
let diffuse_texture = texture::Texture::from_bytes(&device, &queue, diffuse_bytes, "happy-tree.png").unwrap(); // CHANGED!

// Everything up until `let texture_bind_group_layout = ...` can now be removed.
```

Aún necesitamos almacenar el grupo de enlace por separado para que `Texture` no necesite saber cómo se distribuye el `BindGroup`. La creación de `diffuse_bind_group` cambia ligeramente para usar los campos `view` y `sampler` de `diffuse_texture`:

```rust
let diffuse_bind_group = device.create_bind_group(
    &wgpu::BindGroupDescriptor {
        layout: &texture_bind_group_layout,
        entries: &[
            wgpu::BindGroupEntry {
                binding: 0,
                resource: wgpu::BindingResource::TextureView(&diffuse_texture.view), // CHANGED!
            },
            wgpu::BindGroupEntry {
                binding: 1,
                resource: wgpu::BindingResource::Sampler(&diffuse_texture.sampler), // CHANGED!
            }
        ],
        label: Some("diffuse_bind_group"),
    }
);
```

Finalmente, actualicemos nuestro campo `State` para usar nuestra nueva estructura `Texture` reluciente, ya que la necesitaremos en futuros tutoriales.

```rust
pub struct State {
    // ...
    diffuse_bind_group: wgpu::BindGroup,
    diffuse_texture: texture::Texture, // NEW
}
```

```rust
impl State {
    async fn new() -> Self {
        // ...
        Self {
            // ...
            num_indices,
            diffuse_bind_group,
            diffuse_texture, // NEW
        }
    }
}
```

¡Uf!

Con estos cambios en su lugar, el código debería funcionar igual que antes, pero ahora tenemos una forma mucho más fácil de crear texturas.

## Demostración

<WasmExample example="tutorial5_textures"></WasmExample>

<AutoGithubLink/>

## Desafío

Crea otra textura e intercámbiala cuando presiones la tecla de espacio.
