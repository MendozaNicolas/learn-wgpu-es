# Búferes e Índices

## ¡Finalmente vamos a hablar de ellos!

Probablemente ya estabas cansado de que dijera cosas como "Hablaremos de eso cuando hablemos de búferes". Bueno, ahora es el momento de finalmente hablar sobre búferes, pero primero...

## ¿Qué es un búfer?

Un búfer es un blob de datos en la GPU. Se garantiza que un búfer es contiguo, lo que significa que todos los datos se almacenan secuencialmente en la memoria. Los búferes se utilizan generalmente para almacenar cosas simples como estructuras o matrices, pero pueden almacenar cosas más complejas como estructuras de gráficos como árboles (siempre que todos los nodos se almacenen juntos y no hagan referencia a nada fuera del búfer). Vamos a usar búferes mucho, así que empecemos con dos de los más importantes: el búfer de vértices y el búfer de índices.

## El búfer de vértices

Anteriormente, hemos almacenado datos de vértices directamente en el sombreador de vértices. Aunque eso funcionó bien para arrancar, simplemente no funcionará a largo plazo. Los tipos de objetos que necesitamos dibujar variarán en tamaño, y recompilar el sombreador cada vez que necesitemos actualizar el modelo ralentizaría enormemente nuestro programa. En su lugar, vamos a usar búferes para almacenar los datos de vértices que queremos dibujar. Antes de hacer eso, sin embargo, necesitamos describir cómo se ve un vértice. Haremos esto creando una nueva estructura.

```rust
// lib.rs
#[repr(C)]
#[derive(Copy, Clone, Debug)]
struct Vertex {
    position: [f32; 3],
    color: [f32; 3],
}
```

Nuestros vértices tendrán una posición y un color. La posición representa las coordenadas x, y y z del vértice en el espacio 3D. El color son los valores de rojo, verde y azul para el vértice. Necesitamos que `Vertex` sea `Copy` para poder crear un búfer con él.

A continuación, necesitamos los datos reales que formarán nuestro triángulo. Debajo de `Vertex`, añade lo siguiente.

```rust
// lib.rs
const VERTICES: &[Vertex] = &[
    Vertex { position: [0.0, 0.5, 0.0], color: [1.0, 0.0, 0.0] },
    Vertex { position: [-0.5, -0.5, 0.0], color: [0.0, 1.0, 0.0] },
    Vertex { position: [0.5, -0.5, 0.0], color: [0.0, 0.0, 1.0] },
];
```

Disponemos los vértices en orden contrario al reloj: arriba, abajo a la izquierda, abajo a la derecha. Hacemos esto parcialmente por tradición, pero principalmente porque especificamos en el `primitive` del `render_pipeline` que queremos que la `front_face` de nuestro triángulo sea `wgpu::FrontFace::Ccw` para descartar la cara trasera. Esto significa que cualquier triángulo que deba estar de frente a nosotros debe tener sus vértices en orden contrario al reloj.

Ahora que tenemos nuestros datos de vértices, necesitamos almacenarlos en un búfer. Añadamos un campo `vertex_buffer` a `State`.

```rust
// lib.rs
pub struct State {
    // ...
    render_pipeline: wgpu::RenderPipeline,

    // NEW!
    vertex_buffer: wgpu::Buffer,

    // ...
}
```

Ahora vamos a crear el búfer en `new()`.

```rust
// new()
let vertex_buffer = device.create_buffer_init(
    &wgpu::util::BufferInitDescriptor {
        label: Some("Vertex Buffer"),
        contents: bytemuck::cast_slice(VERTICES),
        usage: wgpu::BufferUsages::VERTEX,
    }
);
```

Para acceder al método `create_buffer_init` en `wgpu::Device`, tendremos que importar la característica de extensión [DeviceExt](https://docs.rs/wgpu/latest/wgpu/util/trait.DeviceExt.html#tymethod.create_buffer_init). Para obtener más información sobre características de extensión, consulta [este artículo](http://xion.io/post/code/rust-extension-traits.html).

Para importar la característica de extensión, coloca esta línea en algún lugar cerca de la parte superior de `lib.rs`.

```rust
use wgpu::util::DeviceExt;
```

Observa que estamos usando [bytemuck](https://docs.rs/bytemuck/latest/bytemuck/) para convertir nuestras `VERTICES` como `&[u8]`. El método `create_buffer_init()` espera `&[u8]`, y `bytemuck::cast_slice` hace eso por nosotros. Añade lo siguiente a tu `Cargo.toml`.

```toml
bytemuck = { version = "1.24", features = [ "derive" ] }
```

También necesitaremos implementar dos características para que `bytemuck` funcione. Estas son [bytemuck::Pod](https://docs.rs/bytemuck/latest/bytemuck/trait.Pod.html) y [bytemuck::Zeroable](https://docs.rs/bytemuck/latest/bytemuck/trait.Zeroable.html). `Pod` indica que nuestro `Vertex` es "Plain Old Data" (Datos Antiguos Simples), y por lo tanto puede interpretarse como `&[u8]`. `Zeroable` indica que podemos usar `std::mem::zeroed()`. Podemos modificar nuestra estructura `Vertex` para derivar estos métodos.

```rust
#[repr(C)]
#[derive(Copy, Clone, Debug, bytemuck::Pod, bytemuck::Zeroable)]
struct Vertex {
    position: [f32; 3],
    color: [f32; 3],
}
```

<div class="note">

Si tu estructura incluye tipos que no implementan `Pod` y `Zeroable`, tendrás que implementar estas características manualmente. Estas características no requieren que implementemos ningún método, así que solo necesitamos usar lo siguiente para que nuestro código funcione.

```rust
unsafe impl bytemuck::Pod for Vertex {}
unsafe impl bytemuck::Zeroable for Vertex {}
```

</div>

Finalmente, podemos añadir nuestro `vertex_buffer` a nuestra estructura `State`.

```rust
Ok(Self {
    surface,
    device,
    queue,
    config,
    is_surface_configured: false,
    window,
    render_pipeline,
    vertex_buffer,
})
```

## Entonces, ¿qué hago con él?

Necesitamos decirle al `render_pipeline` que use este búfer cuando estemos dibujando, pero primero, necesitamos decirle al `render_pipeline` cómo leer el búfer. Hacemos esto usando `VertexBufferLayout` y el campo `vertex_buffers` que prometí que hablaríamos cuando creamos el `render_pipeline`.

Un `VertexBufferLayout` define cómo se representa un búfer en la memoria. Sin esto, el render_pipeline no tiene idea de cómo mapear el búfer en el sombreador. Así es como se vería el descriptor para un búfer lleno de `Vertex`.

```rust
wgpu::VertexBufferLayout {
    array_stride: std::mem::size_of::<Vertex>() as wgpu::BufferAddress, // 1.
    step_mode: wgpu::VertexStepMode::Vertex, // 2.
    attributes: &[ // 3.
        wgpu::VertexAttribute {
            offset: 0, // 4.
            shader_location: 0, // 5.
            format: wgpu::VertexFormat::Float32x3, // 6.
        },
        wgpu::VertexAttribute {
            offset: std::mem::size_of::<[f32; 3]>() as wgpu::BufferAddress,
            shader_location: 1,
            format: wgpu::VertexFormat::Float32x3,
        }
    ]
}
```

1. El `array_stride` define cuán ancho es un vértice. Cuando el sombreador va a leer el siguiente vértice, saltará sobre el número de bytes `array_stride`. En nuestro caso, array_stride será probablemente 24 bytes.
2. `step_mode` le dice al pipeline si cada elemento de la matriz en este búfer representa datos por vértice o datos por instancia. Podemos especificar `wgpu::VertexStepMode::Instance` si solo queremos cambiar vértices cuando comenzamos a dibujar una nueva instancia. Cubriremos la instanciación en un tutorial posterior.
3. Los atributos de vértice describen las partes individuales del vértice. Generalmente, esto es un mapeo 1:1 con los campos de una estructura, que es el caso en nuestro.
4. Esto define el `offset` en bytes hasta que comienza el atributo. Para el primer atributo, el offset suele ser cero. Para cualquier atributo posterior, el offset es la suma sobre `size_of` de los datos de los atributos anteriores.
5. Esto le dice al sombreador en qué ubicación almacenar este atributo. Por ejemplo, `@location(0) x: vec3<f32>` en el sombreador de vértices correspondería al campo `position` de la estructura `Vertex`, mientras que `@location(1) x: vec3<f32>` sería el campo `color`.
6. `format` le dice al sombreador la forma del atributo. `Float32x3` corresponde a `vec3<f32>` en el código del sombreador. El valor máximo que podemos almacenar en un atributo es `Float32x4` (`Uint32x4` y `Sint32x4` también funcionan). Tendremos esto en cuenta para cuando tengamos que almacenar cosas que son más grandes que `Float32x4`.

Para los aprendices visuales, nuestro búfer de vértices se ve así.

![A figure of the VertexBufferLayout](./vb_desc.png)

Vamos a crear un método estático en `Vertex` que devuelva este descriptor.

```rust
// lib.rs
impl Vertex {
    fn desc() -> wgpu::VertexBufferLayout<'static> {
        wgpu::VertexBufferLayout {
            array_stride: std::mem::size_of::<Vertex>() as wgpu::BufferAddress,
            step_mode: wgpu::VertexStepMode::Vertex,
            attributes: &[
                wgpu::VertexAttribute {
                    offset: 0,
                    shader_location: 0,
                    format: wgpu::VertexFormat::Float32x3,
                },
                wgpu::VertexAttribute {
                    offset: std::mem::size_of::<[f32; 3]>() as wgpu::BufferAddress,
                    shader_location: 1,
                    format: wgpu::VertexFormat::Float32x3,
                }
            ]
        }
    }
}
```

<div class="note">

Especificar los atributos como lo hicimos ahora es bastante detallado. Podríamos usar la macro `vertex_attr_array` proporcionada por wgpu para limpiar un poco las cosas. Con ella, nuestro `VertexBufferLayout` se convierte en

```rust
wgpu::VertexBufferLayout {
    array_stride: std::mem::size_of::<Vertex>() as wgpu::BufferAddress,
    step_mode: wgpu::VertexStepMode::Vertex,
    attributes: &wgpu::vertex_attr_array![0 => Float32x3, 1 => Float32x3],
}
```

Aunque esto es definitivamente agradable, Rust ve el resultado de `vertex_attr_array` como un valor temporal, por lo que se requiere un ajuste para devolverlo desde una función. Podríamos [hacerlo `const`](https://github.com/gfx-rs/wgpu/discussions/1790#discussioncomment-1160378), como en el ejemplo a continuación:

```rust
impl Vertex {
    const ATTRIBS: [wgpu::VertexAttribute; 2] =
        wgpu::vertex_attr_array![0 => Float32x3, 1 => Float32x3];

    fn desc() -> wgpu::VertexBufferLayout<'static> {
        use std::mem;

        wgpu::VertexBufferLayout {
            array_stride: mem::size_of::<Self>() as wgpu::BufferAddress,
            step_mode: wgpu::VertexStepMode::Vertex,
            attributes: &Self::ATTRIBS,
        }
    }
}
```

De cualquier manera, creo que es bueno mostrar cómo se asignan los datos, así que omitré el uso de esta macro por ahora.

</div>

Ahora, podemos usarlo cuando creemos el `render_pipeline`.

```rust
let render_pipeline = device.create_render_pipeline(&wgpu::RenderPipelineDescriptor {
    // ...
    vertex: wgpu::VertexState {
        // ...
        buffers: &[
            Vertex::desc(),
        ],
    },
    // ...
});
```

Una cosa más: necesitamos establecer el búfer de vértices en el método render. De lo contrario, nuestro programa se bloqueará.

```rust
// render()
render_pass.set_pipeline(&self.render_pipeline);
// NEW!
render_pass.set_vertex_buffer(0, self.vertex_buffer.slice(..));
render_pass.draw(0..3, 0..1);
```

`set_vertex_buffer` toma dos parámetros. El primero es qué ranura de búfer usar para este búfer de vértices. Puedes tener múltiples búferes de vértices establecidos al mismo tiempo.

El segundo parámetro es la porción del búfer a usar. Puedes almacenar tantos objetos en un búfer como tu hardware permita, por lo que `slice` nos permite especificar qué porción del búfer usar. Usamos `..` para especificar el búfer completo.

Antes de continuar, debemos cambiar la llamada `render_pass.draw()` para usar el número de vértices especificado por `VERTICES`. Añade un `num_vertices` a `State`, y establécelo igual a `VERTICES.len()`.

```rust
// lib.rs

pub struct State {
    // ...
    num_vertices: u32,
}

impl State {
    // ...
    fn new(...) -> Self {
        // ...
        let num_vertices = VERTICES.len() as u32;

        Self {
            surface,
            device,
            queue,
            config,
            is_surface_configured: false,
            window,
            render_pipeline,
            vertex_buffer,
            num_vertices,
        }
    }
}
```

Luego, úsalo en la llamada de dibujo.

```rust
// render
render_pass.draw(0..self.num_vertices, 0..1);
```

Antes de que nuestros cambios tengan efecto, necesitamos actualizar nuestro sombreador de vértices para obtener sus datos del búfer de vértices. También haremos que incluya el color del vértice.

```wgsl
// Vertex shader

struct VertexInput {
    @location(0) position: vec3<f32>,
    @location(1) color: vec3<f32>,
};

struct VertexOutput {
    @builtin(position) clip_position: vec4<f32>,
    @location(0) color: vec3<f32>,
};

@vertex
fn vs_main(
    model: VertexInput,
) -> VertexOutput {
    var out: VertexOutput;
    out.color = model.color;
    out.clip_position = vec4<f32>(model.position, 1.0);
    return out;
}

// Fragment shader

@fragment
fn fs_main(in: VertexOutput) -> @location(0) vec4<f32> {
    return vec4<f32>(in.color, 1.0);
}
```

Si has hecho las cosas correctamente, deberías ver un triángulo que se vea algo así.

![A colorful triangle](./triangle.png)

## El búfer de índices
Técnicamente, no *necesitamos* un búfer de índices, pero aún son muy útiles. Un búfer de índices entra en juego cuando comenzamos a usar modelos con muchos triángulos. Considera este pentágono.

![A pentagon made of 3 triangles](./pentagon.png)

Tiene un total de 5 vértices y 3 triángulos. Ahora, si quisiéramos mostrar algo así usando solo vértices, necesitaríamos algo como lo siguiente.

```rust
const VERTICES: &[Vertex] = &[
    Vertex { position: [-0.0868241, 0.49240386, 0.0], color: [0.5, 0.0, 0.5] }, // A
    Vertex { position: [-0.49513406, 0.06958647, 0.0], color: [0.5, 0.0, 0.5] }, // B
    Vertex { position: [0.44147372, 0.2347359, 0.0], color: [0.5, 0.0, 0.5] }, // E

    Vertex { position: [-0.49513406, 0.06958647, 0.0], color: [0.5, 0.0, 0.5] }, // B
    Vertex { position: [-0.21918549, -0.44939706, 0.0], color: [0.5, 0.0, 0.5] }, // C
    Vertex { position: [0.44147372, 0.2347359, 0.0], color: [0.5, 0.0, 0.5] }, // E

    Vertex { position: [-0.21918549, -0.44939706, 0.0], color: [0.5, 0.0, 0.5] }, // C
    Vertex { position: [0.35966998, -0.3473291, 0.0], color: [0.5, 0.0, 0.5] }, // D
    Vertex { position: [0.44147372, 0.2347359, 0.0], color: [0.5, 0.0, 0.5] }, // E
];
```

Observa, sin embargo, que algunos de los vértices se utilizan más de una vez. C y B se utilizan dos veces, y E se repite tres veces. Asumiendo que cada flotante tiene 4 bytes, eso significa que de los 216 bytes que usamos para `VERTICES`, 96 de ellos son datos duplicados. ¿No sería agradable si pudiéramos enumerar estos vértices una sola vez? ¡Bueno, podemos! Ahí es donde entra en juego un búfer de índices.

Básicamente, almacenamos todos los vértices únicos en `VERTICES`, y creamos otro búfer que almacena índices a elementos en `VERTICES` para crear los triángulos. Aquí hay un ejemplo de eso con nuestro pentágono.

```rust
// lib.rs
const VERTICES: &[Vertex] = &[
    Vertex { position: [-0.0868241, 0.49240386, 0.0], color: [0.5, 0.0, 0.5] }, // A
    Vertex { position: [-0.49513406, 0.06958647, 0.0], color: [0.5, 0.0, 0.5] }, // B
    Vertex { position: [-0.21918549, -0.44939706, 0.0], color: [0.5, 0.0, 0.5] }, // C
    Vertex { position: [0.35966998, -0.3473291, 0.0], color: [0.5, 0.0, 0.5] }, // D
    Vertex { position: [0.44147372, 0.2347359, 0.0], color: [0.5, 0.0, 0.5] }, // E
];

const INDICES: &[u16] = &[
    0, 1, 4,
    1, 2, 4,
    2, 3, 4,
];
```

Ahora, con esta configuración, nuestras `VERTICES` ocupan aproximadamente 120 bytes e `INDICES` es solo 18 bytes, dado que `u16` tiene 2 bytes de ancho. En este caso, wgpu añade automáticamente 2 bytes adicionales de relleno para asegurar que el búfer esté alineado a 4 bytes, pero sigue siendo solo 20 bytes. En total, nuestro pentágono tiene 140 bytes. ¡Eso significa que ahorramos 76 bytes! Podría no parecer mucho, pero cuando se trata de conteos de triángulos en cientos de miles, la indexación ahorra mucha memoria. Ten en cuenta que el orden de los índices importa. En el ejemplo anterior, los triángulos se crean en sentido contrario a las agujas del reloj. Si quieres cambiarlo a sentido de las agujas del reloj, ve a tu render pipeline y cambia el `front_face` a `Cw`.

Hay un par de cosas que necesitamos cambiar para usar la indexación. La primera es que necesitamos crear un búfer para almacenar los índices. En el método `new()` de `State`, crea el `index_buffer` después de crear el `vertex_buffer`. Además, cambia `num_vertices` a `num_indices` e establécelo igual a `INDICES.len()`.

```rust
let vertex_buffer = device.create_buffer_init(
    &wgpu::util::BufferInitDescriptor {
        label: Some("Vertex Buffer"),
        contents: bytemuck::cast_slice(VERTICES),
        usage: wgpu::BufferUsages::VERTEX,
    }
);
// NEW!
let index_buffer = device.create_buffer_init(
    &wgpu::util::BufferInitDescriptor {
        label: Some("Index Buffer"),
        contents: bytemuck::cast_slice(INDICES),
        usage: wgpu::BufferUsages::INDEX,
    }
);
let num_indices = INDICES.len() as u32;
```

No necesitamos implementar `Pod` y `Zeroable` para nuestros índices porque `bytemuck` ya los ha implementado para tipos básicos como `u16`. Eso significa que podemos simplemente añadir `index_buffer` y `num_indices` a la estructura `State`.

```rust
pub struct State {
    surface: wgpu::Surface<'static>,
    device: wgpu::Device,
    queue: wgpu::Queue,
    config: wgpu::SurfaceConfiguration,
    is_surface_configured: bool,
    window: Arc<Window>,
    render_pipeline: wgpu::RenderPipeline,
    vertex_buffer: wgpu::Buffer,
    // NEW!
    index_buffer: wgpu::Buffer, 
    num_indices: u32,
}
```

Y luego rellena estos campos en el constructor:

```rust
Ok(Self {
    surface,
    device,
    queue,
    config,
    is_surface_configured: false,
    window,
    render_pipeline,
    vertex_buffer,
    // NEW!
    index_buffer,
    num_indices,
})
```

Todo lo que tenemos que hacer ahora es actualizar el método `render()` para usar el `index_buffer`.

```rust
// render()
render_pass.set_pipeline(&self.render_pipeline);
render_pass.set_vertex_buffer(0, self.vertex_buffer.slice(..));
render_pass.set_index_buffer(self.index_buffer.slice(..), wgpu::IndexFormat::Uint16); // 1.
render_pass.draw_indexed(0..self.num_indices, 0, 0..1); // 2.
```

Un par de cosas a tener en cuenta:

1. El nombre del método es `set_index_buffer`, no `set_index_buffers`. Solo puedes tener un búfer de índices configurado a la vez.
2. Cuando uses un búfer de índices, necesitas usar `draw_indexed`. El método `draw` ignora el búfer de índices. Además, asegúrate de usar el número de índices (`num_indices`), no vértices, ya que tu modelo dibujará de manera incorrecta o el método entrará en `panic` porque no hay suficientes índices.

Con todo eso, deberías tener un pentágono magenta chillón en tu ventana.

![Magenta pentagon in window](./indexed-pentagon.png)

## Corrección de Color

Si usas un selector de color en el pentágono magenta, obtendrás un valor hexadecimal de #BC00BC. Si conviertes esto a valores RGB, obtendrás (188, 0, 188). Dividiendo estos valores por 255 para obtenerlos en el rango [0, 1], obtenemos aproximadamente (0.737254902, 0, 0.737254902). Esto no es lo mismo que lo que estamos usando para nuestros colores de vértice, que es (0.5, 0.0, 0.5). La razón de esto tiene que ver con los espacios de color.

La mayoría de los monitores utilizan un espacio de color conocido como sRGB. Nuestra superficie es (muy probablemente dependiendo de lo que se devuelva de `surface.get_preferred_format()`) usando un formato de textura sRGB. El formato sRGB almacena colores de acuerdo a su brillo relativo en lugar de su brillo real. La razón de esto es que nuestros ojos no perciben la luz linealmente. Notamos más diferencias en colores más oscuros que en colores más claros.

La mayoría del software que usa colores los almacena en formato sRGB (o uno similar patentado). Wgpu espera valores en espacio de color lineal, así que tenemos que convertir los valores.

Obtén el color correcto usando la siguiente fórmula: `rgb_color = ((srgb_color / 255 + 0.055) / 1.055) ^ 2.4`. Haciendo esto con un valor sRGB de (188, 0, 188) nos dará (0.5028864580325687, 0.0, 0.5028864580325687). Un poco diferente de nuestro (0.5, 0.0, 0.5). En lugar de hacer una conversión de color manual, probablemente ahorrarás mucho tiempo usando texturas en su lugar, ya que si se almacenan en una textura sRGB, la conversión a lineal sucederá automáticamente. Cubriremos texturas en la siguiente lección.

## Demo

<WasmExample example="tutorial4_buffer"></WasmExample>

<AutoGithubLink/>

## Desafío
Crea una forma más compleja que la que hicimos (es decir, más de tres triángulos) usando un búfer de vértices y un búfer de índices. Alterna entre los dos con la tecla de espacio.
