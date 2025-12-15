# El Búfer de Profundidad

Echemos un vistazo más cercano al último ejemplo desde un ángulo.

![depth_problems.png](./depth_problems.png)

Los modelos que deberían estar atrás se están renderizando antes que los que están al frente. Esto es causado por el orden de dibujo. Por defecto, los datos de píxeles de un objeto nuevo reemplazan los datos de píxeles antiguos.

Hay dos formas de resolver esto: ordenar los datos de atrás hacia adelante o usar lo que se conoce como un búfer de profundidad.

## Ordenar de atrás hacia adelante

Este es el método más común para renderizado 2D ya que es bastante fácil saber qué debe estar al frente de qué. Puedes simplemente usar el orden z. En renderizado 3D, se vuelve un poco más complicado porque el orden de los objetos cambia según el ángulo de la cámara.

Una forma simple de hacer esto es ordenar todos los objetos por su distancia desde la posición de la cámara. Sin embargo, este método tiene defectos, como cuando un objeto grande está detrás de un objeto pequeño, partes del objeto grande que deberían estar al frente del objeto pequeño se renderizarán detrás de él. También nos encontraremos con problemas con objetos que se superponen *a sí mismos*.

Si queremos hacer esto correctamente, necesitamos tener precisión a nivel de píxel. Ahí es donde entra un *búfer de profundidad*.

## Profundidad de un píxel

Un búfer de profundidad es una textura en blanco y negro que almacena la coordenada z de los píxeles renderizados. Wgpu puede usar esto al dibujar nuevos píxeles para determinar si debe reemplazar o mantener los datos. Esta técnica se llama prueba de profundidad. ¡Esto solucionará nuestro problema de orden de dibujo sin necesidad de ordenar nuestros objetos!

Creemos una función para crear la textura de profundidad en `texture.rs`.

```rust
impl Texture {
    pub const DEPTH_FORMAT: wgpu::TextureFormat = wgpu::TextureFormat::Depth32Float; // 1.
    
    pub fn create_depth_texture(device: &wgpu::Device, config: &wgpu::SurfaceConfiguration, label: &str) -> Self {
        let size = wgpu::Extent3d { // 2.
            width: config.width.max(1),
            height: config.height.max(1),
            depth_or_array_layers: 1,
        };
        let desc = wgpu::TextureDescriptor {
            label: Some(label),
            size,
            mip_level_count: 1,
            sample_count: 1,
            dimension: wgpu::TextureDimension::D2,
            format: Self::DEPTH_FORMAT,
            usage: wgpu::TextureUsages::RENDER_ATTACHMENT // 3.
                | wgpu::TextureUsages::TEXTURE_BINDING,
            view_formats: &[],
        };
        let texture = device.create_texture(&desc);

        let view = texture.create_view(&wgpu::TextureViewDescriptor::default());
        let sampler = device.create_sampler(
            &wgpu::SamplerDescriptor { // 4.
                address_mode_u: wgpu::AddressMode::ClampToEdge,
                address_mode_v: wgpu::AddressMode::ClampToEdge,
                address_mode_w: wgpu::AddressMode::ClampToEdge,
                mag_filter: wgpu::FilterMode::Linear,
                min_filter: wgpu::FilterMode::Linear,
                mipmap_filter: wgpu::FilterMode::Nearest,
                compare: Some(wgpu::CompareFunction::LessEqual), // 5.
                lod_min_clamp: 0.0,
                lod_max_clamp: 100.0,
                ..Default::default()
            }
        );

        Self { texture, view, sampler }
    }
}
```

1. Necesitamos el DEPTH_FORMAT para crear la etapa de profundidad de la `render_pipeline` y para crear la textura de profundidad en sí.
2. Nuestra textura de profundidad debe tener el mismo tamaño que nuestra pantalla si queremos que las cosas se rendericen correctamente. Podemos usar nuestra `config` para asegurar que nuestra textura de profundidad tenga el mismo tamaño que nuestras texturas de superficie.
3. Como estamos renderizando a esta textura, necesitamos agregar la bandera `RENDER_ATTACHMENT` a ella.
4. Técnicamente no *necesitamos* un sampler para una textura de profundidad, pero nuestra estructura `Texture` lo requiere, y necesitamos uno si alguna vez queremos muestrearla.
5. Si decidimos renderizar nuestra textura de profundidad, necesitamos usar `CompareFunction::LessEqual`. Esto se debe a cómo `sampler_comparison` y `textureSampleCompare()` interactúan con la función `texture()` en GLSL.

Creamos nuestra `depth_texture` en `State::new()`.

```rust
let depth_texture = texture::Texture::create_depth_texture(&device, &config, "depth_texture");
```

Necesitamos modificar nuestra `render_pipeline` para permitir la prueba de profundidad.

```rust
let render_pipeline = device.create_render_pipeline(&wgpu::RenderPipelineDescriptor {
    // ...
    depth_stencil: Some(wgpu::DepthStencilState {
        format: texture::Texture::DEPTH_FORMAT,
        depth_write_enabled: true,
        depth_compare: wgpu::CompareFunction::Less, // 1.
        stencil: wgpu::StencilState::default(), // 2.
        bias: wgpu::DepthBiasState::default(),
    }),
    // ...
});
```

1. La función `depth_compare` nos dice cuándo descartar un píxel nuevo. Usar `LESS` significa que los píxeles se dibujarán de adelante hacia atrás. Aquí están los otros valores posibles para un [CompareFunction](https://docs.rs/wgpu/latest/wgpu/enum.CompareFunction.html) que puedes usar:

```rust
#[repr(C)]
#[derive(Copy, Clone, Debug, Hash, Eq, PartialEq)]
#[cfg_attr(feature = "serde", derive(Serialize, Deserialize))]
pub enum CompareFunction {
    Undefined = 0,
    Never = 1,
    Less = 2,
    Equal = 3,
    LessEqual = 4,
    Greater = 5,
    NotEqual = 6,
    GreaterEqual = 7,
    Always = 8,
}
```

2. Hay otro tipo de búfer llamado búfer de plantilla. Es una práctica común almacenar el búfer de plantilla y el búfer de profundidad en la misma textura. Estos campos controlan los valores para las pruebas de plantilla. Usaremos valores predeterminados ya que no estamos usando un búfer de plantilla. Cubriremos los búferes de plantilla [más adelante](../../todo).

No olvides almacenar la `depth_texture` en `State`.

```rust
pub struct State {
    // ...
    depth_texture: Texture,
    // ...
}

async fn new(window: Window) -> Self {
    // ...
    
    Self {
        // ...
        depth_texture,
        // ...
    }
}
```

Necesitamos recordar cambiar el método `resize()` para crear una nueva `depth_texture` y `depth_texture_view`.

```rust
fn resize(&mut self, width: u32, height: u32) {
    // ...

    self.depth_texture = texture::Texture::create_depth_texture(&self.device, &self.config, "depth_texture");

    // ...
}
```

Asegúrate de actualizar la `depth_texture` *después* de actualizar `config`. Si no lo haces, tu programa se bloqueará porque la `depth_texture` tendrá un tamaño diferente al de la textura `surface`.

El último cambio que necesitamos hacer es en la función `render()`. Hemos creado la `depth_texture`, pero actualmente no la estamos usando. La usamos adjuntándola a la `depth_stencil_attachment` de una pasada de renderizado.

```rust
let mut render_pass = encoder.begin_render_pass(&wgpu::RenderPassDescriptor {
    // ...
    depth_stencil_attachment: Some(wgpu::RenderPassDepthStencilAttachment {
        view: &self.depth_texture.view,
        depth_ops: Some(wgpu::Operations {
            load: wgpu::LoadOp::Clear(1.0),
            store: wgpu::StoreOp::Store,
        }),
        stencil_ops: None,
    }),
});
```

¡Y eso es todo lo que tenemos que hacer! ¡No se necesita código de shader! Si ejecutas la aplicación, los problemas de profundidad se solucionarán.

![forest_fixed.png](./forest_fixed.png)

## Demo

<WasmExample example="tutorial8_depth"></WasmExample>

<AutoGithubLink/>

## Desafío

Como el búfer de profundidad es una textura, podemos muestrearla en el shader. Porque es una textura de profundidad, tendremos que usar el tipo de uniform `sampler_comparison` y la función `textureSampleCompare` en lugar de `sampler` y `sampler2D` respectivamente. Crea un grupo de enlace para la textura de profundidad (o reutiliza uno existente), y renderízala en la pantalla.
