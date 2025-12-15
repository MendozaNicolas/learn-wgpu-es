# Instanciación

Nuestra escena en este momento es muy simple: tenemos un objeto centrado en (0,0,0). ¿Qué pasa si queremos más objetos? Aquí es donde entra en juego la instanciación.

La instanciación nos permite dibujar el mismo objeto varias veces con propiedades diferentes (posición, orientación, tamaño, color, etc.). Hay múltiples formas de hacer instanciación. Una forma sería modificar el búfer uniforme para incluir estas propiedades y luego actualizarlo antes de dibujar cada instancia de nuestro objeto.

No queremos usar este método por razones de rendimiento. Actualizar el búfer uniforme para cada instancia requeriría múltiples copias de búfer por fotograma. Además, nuestro método actual para actualizar el búfer uniforme nos requiere crear un nuevo búfer para almacenar los datos actualizados. Eso es mucho tiempo desperdiciado entre llamadas de dibujo.

Si miramos los parámetros de la función `draw_indexed` [en la documentación de wgpu](https://docs.rs/wgpu/latest/wgpu/struct.RenderPass.html#method.draw_indexed), podemos ver una solución a nuestro problema.

```rust
pub fn draw_indexed(
    &mut self,
    indices: Range<u32>,
    base_vertex: i32,
    instances: Range<u32> // <-- This right here
)
```

El parámetro `instances` toma un `Range<u32>`. Este parámetro le dice a la GPU cuántas copias, o instancias, del modelo queremos dibujar. Actualmente, estamos especificando `0..1`, lo que instruye a la GPU a dibujar nuestro modelo una vez y luego detener. Si usáramos `0..5`, nuestro código dibujaría cinco instancias.

El hecho de que `instances` sea un `Range<u32>` puede parecer raro, ya que usar `1..2` para instancias seguiría dibujando una instancia de nuestro objeto. Parece que sería más simple simplemente usar un `u32`, ¿verdad? La razón por la que es un rango es que a veces no queremos dibujar **todos** nuestros objetos. A veces queremos dibujar una selección de ellos porque otros no están en el marco, o estamos depurando y queremos mirar un conjunto particular de instancias.

Bien, ahora sabemos cómo dibujar múltiples instancias de un objeto. ¿Cómo le decimos a wgpu qué instancia particular dibujar? Vamos a usar algo conocido como búfer de instancias.

## El Búfer de Instancias

Crearemos un búfer de instancias de manera similar a cómo creamos un búfer uniforme. Primero, crearemos una estructura llamada `Instance`.

```rust
// lib.rs
// ...

// NEW!
struct Instance {
    position: cgmath::Vector3<f32>,
    rotation: cgmath::Quaternion<f32>,
}
```

<div class="note">

Un `Quaternion` es una estructura matemática que se usa frecuentemente para representar rotaciones. Las matemáticas detrás de ellas están fuera de mi alcance (involucran números imaginarios y espacio 4D), así que no las cubriré aquí. Si realmente quieres profundizar en ellas [aquí hay un artículo de Wolfram Alpha](https://mathworld.wolfram.com/Quaternion.html).

</div>

Usar estos valores directamente en el sombreador sería incómodo, ya que los cuaterniones no tienen un análogo en WGSL. No me siento como escribir las matemáticas en el sombreador, así que convertiremos los datos de `Instance` en una matriz y los almacenaremos en una estructura llamada `InstanceRaw`.

```rust
// NEW!
#[repr(C)]
#[derive(Copy, Clone, bytemuck::Pod, bytemuck::Zeroable)]
struct InstanceRaw {
    model: [[f32; 4]; 4],
}
```

Estos son los datos que irán en `wgpu::Buffer`. Los mantenemos separados para que podamos actualizar `Instance` tanto como queramos sin necesidad de lidiar con matrices. Solo necesitamos actualizar los datos sin procesar antes de dibujar.

Crearemos un método en `Instance` para convertir a `InstanceRaw`.

```rust
// NEW!
impl Instance {
    fn to_raw(&self) -> InstanceRaw {
        InstanceRaw {
            model: (cgmath::Matrix4::from_translation(self.position) * cgmath::Matrix4::from(self.rotation)).into(),
        }
    }
}
```

Ahora necesitamos añadir dos campos a `State`: `instances` e `instance_buffer`.

```rust
pub struct State {
    instances: Vec<Instance>,
    instance_buffer: wgpu::Buffer,
}
```

El crate `cgmath` usa características (traits) para proporcionar métodos matemáticos comunes en sus estructuras, como `Vector3`, que deben importarse antes de que se puedan llamar estos métodos. Por conveniencia, el módulo `prelude` dentro del crate proporciona los más comunes de estos crates de extensión cuando se importa.

Para importar este módulo prelude, pon esta línea cerca de la parte superior de `lib.rs`.

```rust
use cgmath::prelude::*;
```

Crearemos las instancias en `new()`. Usaremos algunas constantes para simplificar las cosas. Mostraremos nuestras instancias en 10 filas de 10, y estarán espaciadas uniformemente.

```rust
const NUM_INSTANCES_PER_ROW: u32 = 10;
const INSTANCE_DISPLACEMENT: cgmath::Vector3<f32> = cgmath::Vector3::new(NUM_INSTANCES_PER_ROW as f32 * 0.5, 0.0, NUM_INSTANCES_PER_ROW as f32 * 0.5);
```

Ahora podemos crear las instancias reales.

```rust
impl State {
    async fn new(window: Arc<Window>) -> anyhow::Result<State> {
        // ...
        let instances = (0..NUM_INSTANCES_PER_ROW).flat_map(|z| {
            (0..NUM_INSTANCES_PER_ROW).map(move |x| {
                let position = cgmath::Vector3 { x: x as f32, y: 0.0, z: z as f32 } - INSTANCE_DISPLACEMENT;

                let rotation = if position.is_zero() {
                    // this is needed so an object at (0, 0, 0) won't get scaled to zero
                    // as Quaternions can affect scale if they're not created correctly
                    cgmath::Quaternion::from_axis_angle(cgmath::Vector3::unit_z(), cgmath::Deg(0.0))
                } else {
                    cgmath::Quaternion::from_axis_angle(position.normalize(), cgmath::Deg(45.0))
                };

                Instance {
                    position, rotation,
                }
            })
        }).collect::<Vec<_>>();
        // ...
    }
}
```

Ahora que tenemos nuestros datos, podemos crear el `instance_buffer` real.

```rust
let instance_data = instances.iter().map(Instance::to_raw).collect::<Vec<_>>();
let instance_buffer = device.create_buffer_init(
    &wgpu::util::BufferInitDescriptor {
        label: Some("Instance Buffer"),
        contents: bytemuck::cast_slice(&instance_data),
        usage: wgpu::BufferUsages::VERTEX,
    }
);
```

Vamos a necesitar crear un nuevo `VertexBufferLayout` para `InstanceRaw`.

```rust
impl InstanceRaw {
    fn desc() -> wgpu::VertexBufferLayout<'static> {
        use std::mem;
        wgpu::VertexBufferLayout {
            array_stride: mem::size_of::<InstanceRaw>() as wgpu::BufferAddress,
            // We need to switch from using a step mode of Vertex to Instance
            // This means that our shaders will only change to use the next
            // instance when the shader starts processing a new instance
            step_mode: wgpu::VertexStepMode::Instance,
            attributes: &[
                // A mat4 takes up 4 vertex slots as it is technically 4 vec4s. We need to define a slot
                // for each vec4. We'll have to reassemble the mat4 in the shader.
                wgpu::VertexAttribute {
                    offset: 0,
                    // While our vertex shader only uses locations 0, and 1 now, in later tutorials, we'll
                    // be using 2, 3, and 4, for Vertex. We'll start at slot 5, not conflict with them later
                    shader_location: 5,
                    format: wgpu::VertexFormat::Float32x4,
                },
                wgpu::VertexAttribute {
                    offset: mem::size_of::<[f32; 4]>() as wgpu::BufferAddress,
                    shader_location: 6,
                    format: wgpu::VertexFormat::Float32x4,
                },
                wgpu::VertexAttribute {
                    offset: mem::size_of::<[f32; 8]>() as wgpu::BufferAddress,
                    shader_location: 7,
                    format: wgpu::VertexFormat::Float32x4,
                },
                wgpu::VertexAttribute {
                    offset: mem::size_of::<[f32; 12]>() as wgpu::BufferAddress,
                    shader_location: 8,
                    format: wgpu::VertexFormat::Float32x4,
                },
            ],
        }
    }
}
```

Necesitamos añadir este descriptor al pipeline de renderizado para que podamos usarlo cuando renderizamos.

```rust
let render_pipeline = device.create_render_pipeline(&wgpu::RenderPipelineDescriptor {
    // ...
    vertex: wgpu::VertexState {
        // ...
        // UPDATED!
        buffers: &[Vertex::desc(), InstanceRaw::desc()],
    },
    // ...
});
```

¡No olvides retornar nuestras nuevas variables!

```rust
Self {
    // ...
    // NEW!
    instances,
    instance_buffer,
}
```

El último cambio que necesitamos hacer es en el método `render()`. Necesitamos vincular nuestro `instance_buffer` y cambiar el rango que usamos en `draw_indexed()` para incluir el número de instancias.

```rust
render_pass.set_pipeline(&self.render_pipeline);
render_pass.set_bind_group(0, &self.diffuse_bind_group, &[]);
render_pass.set_bind_group(1, &self.camera_bind_group, &[]);
render_pass.set_vertex_buffer(0, self.vertex_buffer.slice(..));
// NEW!
render_pass.set_vertex_buffer(1, self.instance_buffer.slice(..));
render_pass.set_index_buffer(self.index_buffer.slice(..), wgpu::IndexFormat::Uint16);

// UPDATED!
render_pass.draw_indexed(0..self.num_indices, 0, 0..self.instances.len() as _);
```

<div class="warning">

Asegúrate de que si añades nuevas instancias al `Vec`, también recrees el `instance_buffer` y `camera_bind_group`. De lo contrario, tus nuevas instancias no aparecerán correctamente.

</div>

Necesitamos referenciar las partes de nuestra nueva matriz en `shader.wgsl` para que podamos usarla para nuestras instancias. Añade lo siguiente en la parte superior de `shader.wgsl`.

```wgsl
struct InstanceInput {
    @location(5) model_matrix_0: vec4<f32>,
    @location(6) model_matrix_1: vec4<f32>,
    @location(7) model_matrix_2: vec4<f32>,
    @location(8) model_matrix_3: vec4<f32>,
};
```

Necesitamos rearmar la matriz antes de poder usarla.

```wgsl
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
    // Continued...
}
```

Aplicaremos `model_matrix` antes de aplicar `camera_uniform.view_proj`. Hacemos esto porque `camera_uniform.view_proj` cambia el sistema de coordenadas del `world space` al `camera space`. Nuestro `model_matrix` es una transformación del `world space`, así que no queremos estar en `camera space` cuando lo usamos.

```wgsl
@vertex
fn vs_main(
    model: VertexInput,
    instance: InstanceInput,
) -> VertexOutput {
    // ...
    var out: VertexOutput;
    out.tex_coords = model.tex_coords;
    out.clip_position = camera.view_proj * model_matrix * vec4<f32>(model.position, 1.0);
    return out;
}
```

¡Con todo eso hecho, deberíamos tener un bosque de árboles!

![./forest.png](./forest.png)

## Demostración

<WasmExample example="tutorial7_instancing"></WasmExample>

<AutoGithubLink/>

## Desafío

Modifica la posición y/o rotación de las instancias en cada fotograma.
