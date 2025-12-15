# El Pipeline

## ¿Qué es un pipeline?

Si estás familiarizado con OpenGL, es posible que recuerdes usar programas de shaders. Puedes pensar en un pipeline como una versión más robusta de eso. Un pipeline describe todas las acciones que la GPU realizará al actuar sobre un conjunto de datos. En esta sección, crearemos específicamente un `RenderPipeline`.

## ¿Espera, shaders?

Los shaders son mini-programas que envías a la GPU para realizar operaciones en tus datos. Hay tres tipos principales de shaders: vertex, fragment y compute. Hay otros, como shaders de geometría o shaders de teselación, pero no son compatibles con WebGL. Deben evitarse en general ([consulta las discusiones](https://community.khronos.org/t/does-the-use-of-geometric-shaders-significantly-reduce-performance/106326)). Por ahora, solo vamos a usar shaders de vertex y fragment.

## Vertex, fragment... ¿qué son esos?

Un vertex es un punto en el espacio 3D (también puede ser 2D). Estos vértices se agrupan en grupos de 2 para formar líneas y/o grupos de 3 para formar triángulos.

<img alt="Vertices Graphic" src="./tutorial3-pipeline-vertices.png" />

La mayoría del renderizado moderno utiliza triángulos para hacer todas las formas, desde formas simples (como cubos) hasta complejas (como personas). Estos triángulos se almacenan como vértices, que son los puntos que forman las esquinas de los triángulos.

<!-- Todo: Find/make an image to put here -->

Usamos un vertex shader para manipular los vértices para transformar la forma como queremos que se vea.

Los vértices se convierten entonces en fragmentos. Cada píxel en la imagen resultante obtiene al menos un fragmento. Cada fragmento tiene un color que se copia en su píxel correspondiente. El fragment shader decide qué color tendrá el fragmento.

## WGSL

[WebGPU Shading Language](https://www.w3.org/TR/WGSL/) (WGSL) es el lenguaje de shader para WebGPU.
El desarrollo de WGSL se enfoca en permitir que se convierta fácilmente al lenguaje de shader correspondiente del backend; por ejemplo, SPIR-V para Vulkan, MSL para Metal, HLSL para DX12 y GLSL para OpenGL.
La conversión se realiza internamente y generalmente no necesitamos preocuparnos por los detalles.
En el caso de wgpu, se realiza mediante la biblioteca llamada [naga](https://github.com/gfx-rs/wgpu/tree/trunk/naga).

Ten en cuenta que, en el momento de escribir esto, algunas implementaciones de WebGPU también admiten SPIR-V, pero es solo una medida temporal durante el período de transición a WGSL y se eliminará (Si tienes curiosidad sobre el drama detrás de SPIR-V y WGSL, consulta [esta entrada de blog](https://kvark.github.io/spirv/2021/05/01/spirv-horrors.html)).

<div class="note">

Si has recorrido este tutorial antes, probablemente notarás que he cambiado de usar GLSL a usar WGSL. Dado que el soporte de GLSL es una preocupación secundaria y que WGSL es el lenguaje de primera clase de WGPU, he decidido convertir todos los tutoriales para usar WGSL. Algunos ejemplos de showcase aún usan GLSL, pero el tutorial principal y todos los ejemplos en adelante usarán WGSL.

</div>

<div class="note">

La especificación de WGSL y su inclusión en WGPU aún están en desarrollo. Si encuentras problemas al usarlo, es posible que quieras que la gente en [https://app.element.io/#/room/#wgpu:matrix.org](https://app.element.io/#/room/#wgpu:matrix.org) revise tu código.

</div>

## Escribiendo los shaders

En la misma carpeta que `main.rs`, crea un archivo `shader.wgsl`. Escribe el siguiente código en `shader.wgsl`.

```wgsl
// Vertex shader

struct VertexOutput {
    @builtin(position) clip_position: vec4<f32>,
};

@vertex
fn vs_main(
    @builtin(vertex_index) in_vertex_index: u32,
) -> VertexOutput {
    var out: VertexOutput;
    let x = f32(1 - i32(in_vertex_index)) * 0.5;
    let y = f32(i32(in_vertex_index & 1u) * 2 - 1) * 0.5;
    out.clip_position = vec4<f32>(x, y, 0.0, 1.0);
    return out;
}
```

Primero, declaramos un `struct` para almacenar la salida de nuestro vertex shader. Actualmente consta de un solo campo, que es la `clip_position` de nuestro vértice. El bit `@builtin(position)` le dice a WGPU que este es el valor que queremos usar como las [coordenadas de clip](https://en.wikipedia.org/wiki/Clip_coordinates) del vértice. Esto es análogo a la variable `gl_Position` de GLSL.

<div class="note">

Los tipos de vectores como `vec4` son genéricos. Actualmente, debes especificar el tipo de valor que contendrá el vector. Por lo tanto, un vector 3D usando floats de 32 bits sería `vec3<f32>`.

</div>

La siguiente parte del código del shader es la función `vs_main`. Usamos `@vertex` para marcar esta función como un punto de entrada válido para un vertex shader. Esperamos un `u32` llamado `in_vertex_index`, que obtiene su valor de `@builtin(vertex_index)`.

Luego declaramos una variable llamada `out` usando nuestro struct `VertexOutput`. Creamos dos variables más para el `x` y `y` de un triángulo.

<div class="note">

Los bits `f32()` e `i32()` son ejemplos de conversiones de tipo.

</div>

<div class="note">

Las variables definidas con `var` se pueden modificar pero deben especificar su tipo. Las variables creadas con `let` pueden tener sus tipos inferidos, pero su valor no se puede cambiar durante el shader.

</div>

Ahora podemos guardar nuestra `clip_position` en `out`. Luego solo devolvemos `out` y hemos terminado con el vertex shader.

<div class="note">

Técnicamente no necesitábamos un struct para este ejemplo y podríamos haber hecho algo como lo siguiente:

```wgsl
@vertex
fn vs_main(
    @builtin(vertex_index) in_vertex_index: u32
) -> @builtin(position) vec4<f32> {
    // Código del vertex shader...
}
```

Agregaremos más campos a `VertexOutput` más adelante, así que bien podemos empezar a usarlo ahora.

</div>

A continuación, el fragment shader. Aún en `shader.wgsl` agrega lo siguiente:

```wgsl
// Fragment shader

@fragment
fn fs_main(in: VertexOutput) -> @location(0) vec4<f32> {
    return vec4<f32>(0.3, 0.2, 0.1, 1.0);
}
```

Esto establece el color del fragmento actual a marrón.

<div class="note">

Ten en cuenta que el punto de entrada para el vertex shader se llamaba `vs_main` y que el punto de entrada para el fragment shader se llama `fs_main`. En versiones anteriores de wgpu, estaba bien que ambas funciones tuvieran el mismo nombre, pero las versiones más nuevas de la [especificación WGSL](https://www.w3.org/TR/WGSL/#declaration-and-scope) requieren que estos nombres sean diferentes. Por lo tanto, el esquema de nombres mencionado anteriormente (que se adopta de los ejemplos de `wgpu`) se usa en todo el tutorial.

</div>

El bit `@location(0)` le dice a WGPU que almacene el valor `vec4` devuelto por esta función en el primer destino de color. Entraremos en más detalle sobre qué es esto más adelante.

<div class="note">

Algo a tener en cuenta sobre `@builtin(position)`, en el fragment shader, este valor está en [espacio de framebuffer](https://gpuweb.github.io/gpuweb/#coordinate-systems). Esto significa que si tu ventana es 800x600, x e y de `clip_position` estarían entre 0-800 y 0-600, respectivamente, siendo y = 0 en la parte superior de la pantalla. Esto puede ser útil si quieres conocer las coordenadas de píxel de un fragmento dado, pero si quieres las coordenadas de posición, tendrás que pasarlas por separado.

```wgsl
struct VertexOutput {
    @builtin(position) clip_position: vec4<f32>,
    @location(0) vert_pos: vec3<f32>,
}

@vertex
fn vs_main(
    @builtin(vertex_index) in_vertex_index: u32,
) -> VertexOutput {
    var out: VertexOutput;
    let x = f32(1 - i32(in_vertex_index)) * 0.5;
    let y = f32(i32(in_vertex_index & 1u) * 2 - 1) * 0.5;
    out.clip_position = vec4<f32>(x, y, 0.0, 1.0);
    out.vert_pos = out.clip_position.xyz;
    return out;
}
```

</div>

## ¿Cómo usamos los shaders?

Esta es la parte donde finalmente hacemos lo que se menciona en el título: el pipeline. Primero, modifiquemos `State` para incluir lo siguiente.

```rust
// lib.rs
pub struct State {
    surface: wgpu::Surface<'static>,
    device: wgpu::Device,
    queue: wgpu::Queue,
    config: wgpu::SurfaceConfiguration,
    is_surface_configured: bool,
    // NEW!
    render_pipeline: wgpu::RenderPipeline,
    window: Arc<Window>,
}
```

Ahora, pasemos al método `new()` y comencemos a hacer el pipeline. Tendremos que cargar los shaders que hicimos anteriormente, ya que el `render_pipeline` los requiere.

```rust
let shader = device.create_shader_module(wgpu::ShaderModuleDescriptor {
    label: Some("Shader"),
    source: wgpu::ShaderSource::Wgsl(include_str!("shader.wgsl").into()),
});
```

<div class="note">

También puedes usar la macro `include_wgsl!` como un pequeño atajo para crear el `ShaderModuleDescriptor`.

```rust
let shader = device.create_shader_module(wgpu::include_wgsl!("shader.wgsl"));
```

</div>

Una cosa más, necesitamos crear un `PipelineLayout`. Entraremos en más detalle sobre esto después de cubrir los `Buffer`s.

```rust
let render_pipeline_layout =
    device.create_pipeline_layout(&wgpu::PipelineLayoutDescriptor {
        label: Some("Render Pipeline Layout"),
        bind_group_layouts: &[],
        push_constant_ranges: &[],
    });
```

Finalmente, tenemos todo lo que necesitamos para crear el `render_pipeline`.

```rust
let render_pipeline = device.create_render_pipeline(&wgpu::RenderPipelineDescriptor {
    label: Some("Render Pipeline"),
    layout: Some(&render_pipeline_layout),
    vertex: wgpu::VertexState {
        module: &shader,
        entry_point: Some("vs_main"), // 1.
        buffers: &[], // 2.
        compilation_options: wgpu::PipelineCompilationOptions::default(),
    },
    fragment: Some(wgpu::FragmentState { // 3.
        module: &shader,
        entry_point: Some("fs_main"),
        targets: &[Some(wgpu::ColorTargetState { // 4.
            format: config.format,
            blend: Some(wgpu::BlendState::REPLACE),
            write_mask: wgpu::ColorWrites::ALL,
        })],
        compilation_options: wgpu::PipelineCompilationOptions::default(),
    }),
    // continued ...
```

Hay varias cosas a tener en cuenta aquí:

1. Aquí puedes especificar qué función dentro del shader debe ser el `entry_point`. Estas son las funciones que marcamos con `@vertex` y `@fragment`
2. El campo `buffers` le dice a `wgpu` qué tipo de vértices queremos pasar al vertex shader. Estamos especificando los vértices en el propio vertex shader, así que dejaremos esto vacío. Pondremos algo aquí en el siguiente tutorial.
3. El `fragment` es técnicamente opcional, así que tienes que envolverlo en `Some()`. Lo necesitamos si queremos almacenar datos de color en la `surface`.
4. El campo `targets` le dice a `wgpu` qué salidas de color debe configurar. Actualmente, solo necesitamos una para la `surface`. Usamos el formato de `surface` para que copiar sea fácil, y especificamos que la mezcla solo debe reemplazar datos de píxeles antiguos con nuevos. También le decimos a `wgpu` que escriba en todos los colores: rojo, azul, verde y alfa. *Hablaremos más sobre* `color_state` *cuando hablemos sobre texturas.*

```rust
    primitive: wgpu::PrimitiveState {
        topology: wgpu::PrimitiveTopology::TriangleList, // 1.
        strip_index_format: None,
        front_face: wgpu::FrontFace::Ccw, // 2.
        cull_mode: Some(wgpu::Face::Back),
        // Setting this to anything other than Fill requires Features::NON_FILL_POLYGON_MODE
        polygon_mode: wgpu::PolygonMode::Fill,
        // Requires Features::DEPTH_CLIP_CONTROL
        unclipped_depth: false,
        // Requires Features::CONSERVATIVE_RASTERIZATION
        conservative: false,
    },
    // continuado...
```

El campo `primitive` describe cómo interpretar nuestros vértices al convertirlos en triángulos.

1. Usar `PrimitiveTopology::TriangleList` significa que cada tres vértices corresponderán a un triángulo.
2. Los campos `front_face` y `cull_mode` le dicen a `wgpu` cómo determinar si un triángulo dado está mirando hacia adelante o no. `FrontFace::Ccw` significa que un triángulo está mirando hacia adelante si los vértices están dispuestos en sentido antihorario. Los triángulos que no se consideran mirando hacia adelante se descartan (no se incluyen en el renderizado) como se especifica por `CullMode::Back`. Cubriremos más sobre culling cuando cubramos los `Buffer`s.

```rust
    depth_stencil: None, // 1.
    multisample: wgpu::MultisampleState {
        count: 1, // 2.
        mask: !0, // 3.
        alpha_to_coverage_enabled: false, // 4.
    },
    multiview: None, // 5.
    cache: None, // 6.
});
```

El resto del método es bastante simple:

1. No estamos usando un búfer de profundidad/stencil actualmente, así que dejamos `depth_stencil` como `None`. *Esto cambiará más adelante*.
2. `count` determina cuántas muestras usará el pipeline. El multimuestreo es un tema complejo, así que no entraremos en detalles aquí.
3. `mask` especifica qué muestras deben estar activas. En este caso, estamos usando todas ellas.
4. `alpha_to_coverage_enabled` tiene que ver con el suavizado. No cubrimos el suavizado aquí, así que dejaremos esto como falso por ahora.
5. `multiview` indica cuántas capas de matriz pueden tener los archivos adjuntos de renderizado. No renderizaremos a texturas de matriz, así que podemos establecer esto en `None`.
6. `cache` permite a wgpu almacenar en caché datos de compilación de shaders. Solo es realmente útil para objetivos de compilación de Android.

<!-- https://gamedev.stackexchange.com/questions/22507/what-is-the-alphatocoverage-blend-state-useful-for -->

Ahora, todo lo que tenemos que hacer es agregar el `render_pipeline` a `State`, y luego podemos usarlo.

```rust
// new()
Ok(Self {
    surface,
    device,
    queue,
    config,
    is_surface_configured: false,
    render_pipeline,
    window,
})
```

## Usando un pipeline

Si ejecutas tu programa ahora, tardará un poco más en iniciarse, pero aún mostrará la pantalla azul que obtuvimos en la sección anterior. Esto se debe a que creamos el `render_pipeline`, pero aún necesitamos modificar el código en `render()` para usarlo realmente.

```rust
// render()

// ...
{
    // 1.
    let mut render_pass = encoder.begin_render_pass(&wgpu::RenderPassDescriptor {
        label: Some("Render Pass"),
        color_attachments: &[
            // This is what @location(0) in the fragment shader targets
            Some(wgpu::RenderPassColorAttachment {
                view: &view,
                resolve_target: None,
                ops: wgpu::Operations {
                    load: wgpu::LoadOp::Clear(
                        wgpu::Color {
                            r: 0.1,
                            g: 0.2,
                            b: 0.3,
                            a: 1.0,
                        }
                    ),
                    store: wgpu::StoreOp::Store,
                }
            })
        ],
        depth_stencil_attachment: None,
    });

    // NEW!
    render_pass.set_pipeline(&self.render_pipeline); // 2.
    render_pass.draw(0..3, 0..1); // 3.
}
// ...
```

No cambiamos mucho, pero hablemos sobre qué cambiamos.

1. Renombramos `_render_pass` a `render_pass` y lo hicimos mutable.
2. Establecimos el pipeline en el `render_pass` usando el que acabamos de crear.
3. Le decimos a `wgpu` que dibuje *algo* con tres vértices y una instancia. Aquí es de donde viene `@builtin(vertex_index)`.

Con todo eso deberías estar viendo un hermoso triángulo marrón.

![Dicho hermoso triángulo marrón](./tutorial3-pipeline-triangle.png)

## Demo

<WasmExample example="tutorial3_pipeline"></WasmExample>

<AutoGithubLink/>

## Desafío

Crea un segundo pipeline que use los datos de posición del triángulo para crear un color que luego envíe al fragment shader. Haz que la aplicación alterne entre estos cuando presiones la barra espaciadora. *Pista: necesitarás modificar* `VertexOutput`
