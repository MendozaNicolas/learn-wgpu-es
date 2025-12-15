# Introducción a los Compute Pipelines

Los compute pipelines son una de las características más emocionantes que proporciona WebGPU.
Te permiten ejecutar cargas de trabajo de cómputo arbitrarias a velocidades solo posibles con
los masivos conteos de núcleos de GPU modernos. Puedes ejecutar modelos de aprendizaje automático en la
web, realizar manipulación de imágenes sin necesidad de configurar los pasos del pipeline de renderizado
como procesamiento de vértices y sombreado de fragmentos, procesar números masivos de
partículas, animar cientos de personajes rigged, etc.

Hay muchos temas que podríamos cubrir, y lo que específicamente quieras usar
compute shaders podría no estar cubierto aquí, pero espero que sea suficiente
para comenzar. Además de eso, estoy intentando un nuevo formato donde incluiré menos
código boilerplate y enfocaré más en los conceptos. El código seguirá siendo
vinculado al final del artículo si te atascas con tu implementación.

## Por qué el cómputo en GPU es rápido

Las GPUs generalmente se consideran más rápidas que las CPUs, pero técnicamente no es
preciso. La velocidad de procesamiento de GPU es aproximadamente la misma que las CPUs, a veces incluso más lenta.
Según [NVIDIA](https://www.nvidia.com/en-us/geforce/graphics-cards/compare/)
la mayoría de sus tarjetas modernas tienen velocidades de reloj alrededor de 2.5 GHz.
[Qualcomm anuncia](https://www.qualcomm.com/products/mobile/snapdragon/laptops-and-tablets/snapdragon-x-elite)
que el Snapdragon X Elite tiene velocidades de reloj de 3.4 - 4.3 Ghz.

Entonces, ¿por qué las GPUs son tan populares para cargas de cómputo masivas?

La respuesta es el conteo de núcleos. El Snapdragon X Elite tiene 12 núcleos. La RTX 5090 tiene
nada menos que 21760 núcleos. Esa es una diferencia de 4 órdenes de magnitud. Con algunas matemáticas
al dorso si un algoritmo tarda un segundo en ejecutar una operación en la CPU y
2 en la GPU, entonces dado 12000 elementos la CPU tardará 1000 segundos (alrededor de 16 minutos)
mientras que la GPU tardará 2 segundos (sin contar enviar datos hacia/desde la GPU y
tiempo de configuración).

Quizás una demostración sea apropiada.

<iframe width="560" height="315" src="https://www.youtube.com/embed/vGWoV-8lteA?si=Sgl2Qq0CFoaGXMQa" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

Las GPUs son rápidas porque pueden hacer miles de cosas al mismo tiempo. Dicho esto,
no todos los algoritmos se benefician de aprovechar este poder de cómputo.

## ¿Cuándo debo usar compute pipelines?

No puedo hacer una lista completa de todas las cosas para las que podrías usar una GPU,
pero aquí hay algunas reglas generales:

- Tareas que pueden ser fácilmente paralelizadas. Las GPUs no les gusta cambiar de tareas, así que si
necesitas que el cálculo use datos de operaciones anteriores, los compute shaders probablemente
serán más lentos que un enfoque basado en CPU. Si cada operación puede ejecutarse sin
conocimiento de otras operaciones, puedes obtener mucho de la GPU.
- Ya tienes los datos en la GPU. Si estás trabajando con datos de texturas o modelos,
a menudo puede ser más rápido procesarlos con un compute shader en lugar de copiar los datos
a la CPU, modificarlos, y luego enviarlos de vuelta a la GPU.
- Tienes una cantidad masiva de datos. En algún momento el tamaño de tus datos comienza a superar
el tiempo de configuración y la complejidad de usar un compute pipeline. Aún necesitarás adaptar
tu enfoque a los datos y el procesamiento que necesites hacer.

¡Ahora que hemos aclarado eso, comencemos!

## Configurar el dispositivo y la cola

Usar compute shaders requiere mucho menos código que usar un render pipeline. No
necesitamos una ventana, así que podemos obtener una instancia de WGPU, solicitar un adaptador, y solicitar
un dispositivo y cola con este código simple:

```rust
    let instance = wgpu::Instance::new(&Default::default());
    let adapter = instance.request_adapter(&Default::default()).await.unwrap();
    let (device, queue) = adapter.request_device(&Default::default()).await.unwrap();
```

<div class="note">

Estoy usando [pollster](https://docs.rs/pollster) para manejar `async` en el código nativo en
estos ejemplos. Puedes usar cualquier implementación de `async` que prefieras. También estoy
usando [anyhow](https://docs.rs/anyhow) para manejo de errores, y [flume](https://docs.rs/flume)
para su implementación de canal `async`.

</div>

Si quieres más información sobre estas llamadas y los posibles argumentos que puedes pasar
a ellas, revisa [la guía de renderizado](../../beginner/tutorial2-surface/).

Ahora que tenemos un dispositivo para comunicarnos con la GPU, comencemos a hablar sobre cómo configurar
un compute pipeline.

## Compute Pipelines

Los compute pipelines son mucho más simples de configurar que los render pipelines. No tenemos que configurar
el pipeline de vértices tradicional. ¡Echa un vistazo!

```rust
    let shader = device.create_shader_module(wgpu::include_wgsl!("introduction.wgsl"));

    let pipeline = device.create_compute_pipeline(&wgpu::ComputePipelineDescriptor {
        label: Some("Introduction Compute Pipeline"),
        layout: None,
        module: &shader,
        entry_point: None,
        compilation_options: Default::default(),
        cache: Default::default(),
    });
```

Estoy usando los valores por defecto para todo aquí excepto la `label` y el `module` del shader
que contiene el código del shader real. No estoy especificando un `layout` de bind group, lo que significa
que wgpu usará el código del shader para derivar uno. No proporciono un `entry_point` ya que WGPU
seleccionará una función con etiqueta `@compute` si hay solo una en el archivo.

El código del shader para este ejemplo también es simple:

```wgsl
// A read-only storage buffer that stores and array of unsigned 32bit integers
@group(0) @binding(0) var<storage, read> input: array<u32>;
// This storage buffer can be read from and written to
@group(0) @binding(1) var<storage, read_write> output: array<u32>;

// Tells wgpu that this function is a valid compute pipeline entry_point
@compute
// Specifies the "dimension" of this work group
@workgroup_size(64)
fn main(
    // global_invocation_id specifies our position in the invocation grid
    @builtin(global_invocation_id) global_invocation_id: vec3<u32>
) {
    let index = global_invocation_id.x;
    let total = arrayLength(&input);

    // workgroup_size may not be a multiple of the array size so
    // we need to exit out a thread that would index out of bounds.
    if (index >= total) {
        return;
    }

    // a simple copy operation
    output[global_invocation_id.x] = input[global_invocation_id.x];
}
```

Este shader es muy simple. Todo lo que hace es copiar el contenido de un buffer a otro.
La única cosa que siento que necesita explicación es el concepto de workgroups y `workgroup_size`.

## Workgroups

Si bien las GPUs prefieren que cada hilo pueda procesar ciegamente su trabajo, los problemas reales
requieren cierta cantidad de sincronización. Los compute shaders logran esto a través de workgroups.

Un workgroup es un grupo de `X * Y * Z` hilos que comparten información sobre una tarea.
Definimos el tamaño de este workgroup usando la bandera `workgroup_size`. Vimos una
versión abreviada de eso arriba, pero aquí está la versión completa:

```wgsl
@workgroup_size(64, 1, 1)
```

Esto significa que nuestro compute shader creará workgroups con `64 * 1 * 1` hilos que se simplifica
a solo 64 hilos por workgroup. Si en su lugar usáramos:

```wgsl
@workgroup_size(64, 64, 1)
```

Obtendríamos `64 * 64 * 1` hilos, u 4096 hilos por workgroup.

El tamaño máximo de workgroup soportado puede variar dependiendo de tu dispositivo, pero la especificación de WebGPU garantiza
lo siguiente:

- Un tamaño máximo de workgroup X de 256
- Un tamaño máximo de workgroup Y de 256
- Un tamaño máximo de workgroup Z de 64
- Un tamaño total de workgroup de 256

Esto significa que es posible que no podamos usar `@workgroup_size(64, 64, 1)` pero `@workgroup_size(16, 16, 1)`
debería funcionar en la mayoría de dispositivos.

<div class="note">

### ¿Por qué XYZ?

Una gran parte de los datos utilizados en programación de GPU vienen en arrays 2D e incluso 3D. Por esto `workgroup_size`
usa 3 dimensiones en lugar de 1 para hacer que escribir código multidimensional sea más conveniente.

Por ejemplo, un blur en una imagen 2D se beneficiaría de un workgroup 2D para que cada hilo
coincida con un píxel en la imagen. Una implementación de marching cubes se beneficiaría de un workgroup 3D,
para que cada hilo maneje la geometría para un voxel en la cuadrícula de voxeles.

</div>

## El global invocation id

Cada hilo en un workgroup tiene un id asociado que dice a qué workgroup
pertenece el hilo. Si accedemos a esto usando el built-in `workgroup_id`.

```wgsl
@compute
@workgroup_size(64)
fn main(
    @builtin(workgroup_id) workgroup_id: vec3<u32>,
) {
    // ...
}
```

Saber dónde estamos en el workgroup también es útil, y lo hacemos usando el
built-in `local_invocation_id`.

```wgsl
@compute
@workgroup_size(64)
fn main(
    @builtin(workgroup_id) workgroup_id: vec3<u32>,
    @builtin(local_invocation_id) local_invocation_id: vec3<u32>,
) {
    // ...
}
```

Luego podemos calcular nuestra posición global en la cuadrícula de invocación del workgroup usando

```wgsl
let id = workgroup_id * workgroup_size + local_invocation_id;
```

También podemos simplemente usar el built-in `global_invocation_id` como hicimos en el código del shader
listado arriba.

### ¿De dónde viene workgroup_id?

Cuando despachamos nuestro compute shader necesitamos especificar las dimensiones X, Y, y Z
de lo que se llama la "cuadrícula de compute shader". Considera este código.

```rust

    {
        // We specified 64 threads per workgroup in the shader, so we need to compute how many
        // workgroups we need to dispatch.
        let num_dispatches = input_data.len().div_ceil(64) as u32;

        let mut pass = encoder.begin_compute_pass(&Default::default());
        pass.set_pipeline(&pipeline);
        pass.set_bind_group(0, &bind_group, &[]);
        pass.dispatch_workgroups(num_dispatches, 1, 1);
    }
```

En la llamada `pass.dispatch_workgroups()` usamos una cuadrícula con dimensiones `(num_dispatches, 1, 1)`
lo que significa que lanzaremos `num_dispatches * 1 * 1` workgroups. La GPU luego asigna a cada workgroup
un id con la coordenada x siendo entre 0 y `num_dispatches - 1`.

Es importante saberlo porque si cambias el tamaño del workgroup, el `global_invocation_id` puede cambiar
lo que significa que potencialmente usarías más hilos de los que necesitas o no lo suficiente.

## Buffers

Aunque he cubierto buffers en la [guía de renderizado](../../beginner/tutorial4-buffer/),
también los revisaré brevemente aquí. En WebGPU un buffer es memoria en la GPU que has
apartado. Esta memoria puede usarse para cualquier cosa desde datos de vértices hasta neuronas en una
red neuronal. En su mayor parte a la GPU no le importa qué datos contiene el buffer,
pero le importa cómo se usan esos datos.

Aquí hay un ejemplo de configuración de un buffer de entrada y salida.

```rust
    let input_buffer = device.create_buffer_init(&BufferInitDescriptor {
        label: Some("input"),
        contents: bytemuck::cast_slice(&input_data),
        usage: wgpu::BufferUsages::COPY_DST | wgpu::BufferUsages::STORAGE,
    });

    let output_buffer = device.create_buffer(&wgpu::BufferDescriptor {
        label: Some("output"),
        size: input_buffer.size(),
        usage: wgpu::BufferUsages::COPY_SRC | wgpu::BufferUsages::STORAGE,
        mapped_at_creation: false,
    });
```

Específicamente necesitamos el uso de `STORAGE` para nuestro buffer en este shader. Podemos
usar `UNIFORM` para algunas cosas, pero los buffers uniformes son más limitados en cuanto a
qué tamaño pueden tener y no pueden ser modificados en el shader.

## Configuración de Bindgroup

De nuevo no entraré en detalle sobre cómo definir bind groups aquí, ya que
ya lo hice en [la guía de renderizado](../../beginner/tutorial5-textures/),
pero cubriré la teoría. En WebGPU un bind group describe recursos que pueden
ser usados por el shader. Estos pueden ser texturas, buffers, samplers, etc. Un
`BindGroupLayout` define cómo estos recursos se agrupan, qué etapas de shader
tienen acceso a ellos, y cómo el shader interpretará los recursos.

Puedes especificar manualmente el `BindGroupLayout`, pero WGPU puede inferir el layout
basándose en el código del shader. Por ejemplo:

```wgsl
@group(0) @binding(0) var<storage, read> input: array<u32>;
@group(0) @binding(1) var<storage, read_write> output: array<u32>;
```

WGPU interpreta esto como un layout con 2 entradas, un buffer de almacenamiento de solo lectura
llamado `input` en la vinculación 0, y un buffer de almacenamiento que puede ser leído y
escrito llamado `output` en la vinculación 1. Podemos crear fácilmente un bindgroup que
satisface esto con el siguiente código:

```rust
    let bind_group = device.create_bind_group(&wgpu::BindGroupDescriptor {
        label: None,
        layout: &pipeline.get_bind_group_layout(0),
        entries: &[
            wgpu::BindGroupEntry {
                binding: 0,
                resource: input_buffer.as_entire_binding(),
            },
            wgpu::BindGroupEntry {
                binding: 1,
                resource: output_buffer.as_entire_binding(),
            },
        ],
    });
```

## Obtener datos fuera de la GPU

Dependiendo de las necesidades de tu aplicación, los datos que procesas en un compute shader
pueden quedarse en la GPU ya que solo se usan para renderizado u otros compute pipelines.
Si necesitas obtener esos datos de la GPU a la CPU, o si simplemente quieres
echar un vistazo a ellos, afortunadamente hay una forma de hacerlo.

El proceso es un poco complicado, así que veamos el código.

```rust
{
        // The mapping process is async, so we'll need to create a channel to get
        // the success flag for our mapping
        let (tx, rx) = channel();

        // We send the success or failure of our mapping via a callback
        temp_buffer.map_async(wgpu::MapMode::Read, .., move |result| tx.send(result).unwrap());

        // The callback we submitted to map async will only get called after the
        // device is polled or the queue submitted
        device.poll(wgpu::PollType::wait_indefinitely())?;

        // We check if the mapping was successful here
        rx.recv()??;

        // We then get the bytes that were stored in the buffer
        let output_data = temp_buffer.get_mapped_range(..);

        // Now we have the data on the CPU we can do what ever we want to with it
        assert_eq!(&input_data, bytemuck::cast_slice(&output_data));
    }

    // We need to unmap the buffer to be able to use it again
    temp_buffer.unmap();
```

Es posible que hayas notado que usé una variable llamada `temp_buffer` y no `output_buffer`
en el mapeo. La razón es que necesitamos que el buffer siendo mapeado tenga
el uso de `MAP_READ`. Este uso solo es compatible con el uso de `COPY_DST`, lo que significa
que no puede tener el uso de `STORAGE` ni de `UNIFORM`, lo que significa que no podemos usar el buffer
en un compute shader. Lo evitamos creando un buffer temporal al que copiamos
el `output_buffer`, y luego lo mapeamos. Aquí está el código de configuración para el `temp_buffer`:

```rust
    let temp_buffer = device.create_buffer(&wgpu::BufferDescriptor {
        label: Some("temp"),
        size: input_buffer.size(),
        usage: wgpu::BufferUsages::COPY_DST | wgpu::BufferUsages::MAP_READ,
        mapped_at_creation: false,
    });
```

Necesitamos realizar esta copia antes de enviar la cola.

```rust
    encoder.copy_buffer_to_buffer(&output_buffer, 0, &temp_buffer, 0, output_buffer.size());

    queue.submit([encoder.finish()]);
```

## Conclusión

Eso es todo. No es demasiado difícil, especialmente comparado con configurar un render pipeline. Ahora que
sabemos cómo usar un compute pipeline podemos realmente comenzar a hacer cosas más interesantes.
Esta guía no puede cubrir posiblemente todas las formas de usar compute shaders, pero planeo cubrir
algunos de los bloques de construcción centrales que necesitas para construir la mayoría de algoritmos. Después de eso puedes tomar
los conceptos y aplicarlos a tus propios proyectos!

## Demo

<WasmExample example="compute" noCanvas="true" autoLoad="true"></WasmExample>

<AutoGithubLink path="/compute/src/"/>
