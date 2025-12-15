# La Superficie

## Primero, un poco de limpieza: State

Creamos state en el tutorial anterior, ahora pongamos cosas
en él.

```rust
// lib.rs

pub struct State {
    surface: wgpu::Surface<'static>,
    device: wgpu::Device,
    queue: wgpu::Queue,
    config: wgpu::SurfaceConfiguration,
    is_surface_configured: bool,
    window: Arc<Window>,
}
```

Estoy simplificando los campos de `State`, pero tendrán más sentido cuando explique el código detrás de estos métodos.

## State::new()

El código para esto es bastante directo, pero vamos a desglosarlo un poco.

```rust
impl State {
    // ...
    async fn new(window: Arc<Window>) -> anyhow::Result<State> {
        let size = window.inner_size();

        // La instancia es un manejador de nuestra GPU
        // BackendBit::PRIMARY => Vulkan + Metal + DX12 + Browser WebGPU
        let instance = wgpu::Instance::new(&wgpu::InstanceDescriptor {
            #[cfg(not(target_arch = "wasm32"))]
            backends: wgpu::Backends::PRIMARY,
            #[cfg(target_arch = "wasm32")]
            backends: wgpu::Backends::GL,
            ..Default::default()
        });

        let surface = instance.create_surface(window.clone()).unwrap();

        let adapter = instance
            .request_adapter(&wgpu::RequestAdapterOptions {
                power_preference: wgpu::PowerPreference::default(),
                compatible_surface: Some(&surface),
                force_fallback_adapter: false,
            })
            .await?;

        // ...
    }
```

### Instance y Adapter

La `instance` es lo primero que creas cuando usas wgpu. Su propósito principal
es crear `Adapter`s y `Surface`s.

El `adapter` es un manejador de nuestra tarjeta gráfica real. Puedes usarlo para obtener información sobre la tarjeta gráfica, como su nombre y qué backend usa el adaptador. Lo usamos para crear nuestro `Device` y `Queue` más adelante. Vamos a discutir los campos de `RequestAdapterOptions`.

* `power_preference` tiene dos variantes: `LowPower` e `HighPerformance`. `LowPower` elegirá un adaptador que favorezca la duración de la batería, como una GPU integrada. `HighPerformance` elegirá un adaptador para GPUs más potentes pero que consumen más, como una tarjeta gráfica dedicada. WGPU favorecerá `LowPower` si no hay un adaptador para la opción `HighPerformance`.
* El campo `compatible_surface` le dice a wgpu que encuentre un adaptador que pueda presentarse a la superficie suministrada.
* El `force_fallback_adapter` obliga a wgpu a elegir un adaptador que funcionará en todo el hardware. Esto generalmente significa que el backend de renderizado usará un sistema "software" en lugar de hardware como una GPU.

<div class="note">

Las opciones que pasé a `request_adapter` no están garantizadas para funcionar en todos los dispositivos, pero funcionarán en la mayoría. Si wgpu no puede encontrar un adaptador con los permisos requeridos, `request_adapter` devolverá `None`. Si quieres obtener todos los adaptadores para un backend particular, puedes usar `enumerate_adapters`. Esto te dará un iterador que puedes recorrer para verificar si uno de los adaptadores funciona para tus necesidades.

```rust
let adapter = instance
    .enumerate_adapters(wgpu::Backends::all())
    .filter(|adapter| {
        // Verifica si este adaptador soporta nuestra superficie
        adapter.is_surface_supported(&surface)
    })
    .next()
    .unwrap()
```

Una cosa a notar es que `enumerate_adapters` no está disponible en WASM, así que tienes que usar `request_adapter`.

Otra cosa a notar es que los `Adapter`s están bloqueados a un backend específico. Si estás en Windows y tienes dos tarjetas gráficas, tendrás al menos cuatro adaptadores disponibles para usar: 2 Vulkan y 2 DirectX.

Para más campos que puedes usar para refinar tu búsqueda, [consulta la documentación](https://docs.rs/wgpu/latest/wgpu/struct.Adapter.html).

</div>

### La Superficie

La `surface` es la parte de la ventana en la que dibujamos. La necesitamos para dibujar directamente en la pantalla. Nuestra `window` necesita implementar el trait `HasRawWindowHandle` de [raw-window-handle](https://crates.io/crates/raw-window-handle) para crear una superficie. Afortunadamente, la `Window` de winit es perfecta para esto. También la necesitamos para solicitar nuestro `adapter`.

### Device y Queue

Usemos el `adapter` para crear el device y la queue.

```rust
        let (device, queue) = adapter
            .request_device(&wgpu::DeviceDescriptor {
                label: None,
                required_features: wgpu::Features::empty(),
                experimental_features: wgpu::ExperimentalFeatures::disabled(),
                // WebGL no soporta todas las características de wgpu, así que si
                // estamos compilando para la web tendremos que deshabilitar algunas.
                required_limits: if cfg!(target_arch = "wasm32") {
                    wgpu::Limits::downlevel_webgl2_defaults()
                } else {
                    wgpu::Limits::default()
                },
                memory_hints: Default::default(),
                trace: wgpu::Trace::Off,
            })
            .await?;
```

El campo `required_features` en `DeviceDescriptor` nos permite especificar qué características extra queremos. Para este ejemplo simple, decidí no usar ninguna característica extra.

<div class="note">

La tarjeta gráfica que tienes limita las características que puedes usar. Si quieres usar ciertas características, es posible que tengas que limitar qué dispositivos soportas u ofrecer soluciones alternativas.

Puedes obtener una lista de características soportadas por tu dispositivo usando `adapter.features()` o `device.features()`.

Puedes ver una lista completa de características [aquí](https://docs.rs/wgpu/latest/wgpu/struct.Features.html).

</div>

El campo `experimental_features` especifica si tenemos la intención de usar características que
aún no son estables. Dejaremos esto deshabilitado por ahora.

El campo `required_limits` describe el límite de ciertos tipos de recursos que podemos crear. Usaremos los valores por defecto para este tutorial para que podamos soportar la mayoría de dispositivos. Puedes ver una lista de límites [aquí](https://docs.rs/wgpu/latest/wgpu/struct.Limits.html).

El campo `memory_hints` proporciona al adaptador una estrategia de asignación de memoria preferida, si se soporta. Puedes ver las opciones disponibles [aquí](https://wgpu.rs/doc/wgpu/enum.MemoryHints.html).

```rust
        let surface_caps = surface.get_capabilities(&adapter);
        // El código de shader en este tutorial asume una textura de superficie sRGB. Usar una diferente
        // resultará en que todos los colores salgan más oscuros. Si quieres soportar
        // superficies non-sRGB, necesitarás tenerlo en cuenta al dibujar al frame.
        let surface_format = surface_caps.formats.iter()
            .find(|f| f.is_srgb())
            .copied()
            .unwrap_or(surface_caps.formats[0]);
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
```

Aquí estamos definiendo una configuración para nuestra superficie. Esto definirá cómo la superficie crea sus `SurfaceTexture`s subyacentes. Hablaremos sobre `SurfaceTexture` cuando lleguemos a la función `render`. Por ahora, hablemos sobre los campos de la configuración.

El campo `usage` describe cómo se usarán los `SurfaceTexture`s. `RENDER_ATTACHMENT` especifica que las texturas se usarán para escribir en la pantalla (hablaremos sobre más `TextureUsages` más adelante).

El `format` define cómo se almacenarán los `SurfaceTexture`s en la GPU. Podemos obtener un formato soportado de las `SurfaceCapabilities`.

`width` y `height` son el ancho y alto en píxeles de un `SurfaceTexture`. Esto generalmente debe ser el ancho y alto de la ventana.

<div class="warning">

Asegúrate de que el ancho y alto del `SurfaceTexture` no sean 0, ya que eso puede hacer que tu aplicación falle.

</div>

`present_mode` usa la enumeración `wgpu::PresentMode`, que determina cómo sincronizar la superficie con la pantalla. Por simplicidad, seleccionamos la primera opción disponible. Si no quieres selección en tiempo de ejecución, `PresentMode::Fifo` limitará la velocidad de fotogramas a la velocidad de fotogramas de la pantalla. Esto es esencialmente VSync. Se garantiza que este modo sea soportado en todas las plataformas. Hay otras opciones, y puedes verlas todas [en la documentación](https://docs.rs/wgpu/latest/wgpu/enum.PresentMode.html)

<div class="note">

Si quieres permitir que tus usuarios elijan qué `PresentMode` usar, puedes usar [SurfaceCapabilities::present_modes](https://docs.rs/wgpu/latest/wgpu/struct.SurfaceCapabilities.html#structfield.present_modes) para obtener una lista de todos los `PresentMode`s que la superficie soporta:

```rust
let modes = &surface_caps.present_modes;
```

De cualquier forma, `PresentMode::Fifo` siempre será soportado, y `PresentMode::AutoVsync` y `PresentMode::AutoNoVsync` tienen soporte de fallback y por lo tanto funcionarán en todas las plataformas.

</div>

`alpha_mode` honestamente no es algo con lo que esté familiarizado. Creo que tiene algo que ver con ventanas transparentes, pero siéntete libre de abrir un pull request. Por ahora, usaremos el primer `AlphaMode` en la lista dada por `surface_caps`.

`view_formats` es una lista de `TextureFormat`s que puedes usar cuando creas `TextureView`s (cubriremos esos brevemente más adelante en este tutorial así como en más profundidad [en el tutorial de texturas](../tutorial5-textures)). Al momento de escribir, esto significa que si tu superficie está en espacio de color sRGB, puedes crear una vista de textura que use un espacio de color lineal.

Ahora que hemos configurado nuestra superficie correctamente, podemos agregar estos nuevos campos al final del método. El campo `is_surface_configured` se usará más adelante.

```rust
    async fn new(window: Arc<Window>) -> anyhow::Result<State> {
        // ...

        Ok(Self {
            surface,
            device,
            queue,
            config,
            is_surface_configured: false,
            window,
        })
    }
```

## resize()

Si queremos soportar el redimensionamiento en nuestra aplicación, vamos a necesitar reconfigurar la `surface` cada vez que cambie el tamaño de la ventana. Esa es la razón por la cual almacenamos el `size` físico y la `config` usada para configurar la `surface`. Con todo esto, el método resize es muy simple.

```rust
// impl State
pub fn resize(&mut self, width: u32, height: u32) {
    if width > 0 && height > 0 {
        self.config.width = width;
        self.config.height = height;
        self.surface.configure(&self.device, &self.config);
        self.is_surface_configured = true;
    }
}
```

Aquí es donde configuramos la `surface`. Necesitamos que la superficie esté configurada antes de poder hacer nada con ella. Establecemos el indicador `is_surface_configured` a true aquí y lo verificaremos en la función `render()`.

## handle_key()

Aquí es donde manejaremos los eventos de teclado. Actualmente solo queremos salir de la aplicación cuando se presione la tecla Escape. Haremos otras cosas más adelante.

```rust
// impl State
fn handle_key(&self, event_loop: &ActiveEventLoop, code: KeyCode, is_pressed: bool) {
    match (code, is_pressed) {
        (KeyCode::Escape, true) => event_loop.exit(),
        _ => {}
    }
}
```

Necesitaremos llamar a nuestra nueva función `handle_key()` en la función `window_event()` en `App`.

```rust
impl ApplicationHandler<State> for App {
    // ...

    fn window_event(
        &mut self,
        event_loop: &ActiveEventLoop,
        _window_id: winit::window::WindowId,
        event: WindowEvent,
    ) {
        let state = match &mut self.state {
            Some(canvas) => canvas,
            None => return,
        };

        match event {
            // ...
            WindowEvent::KeyboardInput {
                event:
                    KeyEvent {
                        physical_key: PhysicalKey::Code(code),
                        state: key_state,
                        ..
                    },
                ..
            } => state.handle_key(event_loop, code, key_state.is_pressed()),
            _ => {}
        }
    }
}
```

## update()

No tenemos nada que actualizar aún, así que dejamos el método vacío.

```rust
fn update(&mut self) {
    // elimina `todo!()`
}
```

Agregaremos código aquí más adelante para mover objetos alrededor.

## render()

Aquí es donde ocurre la magia. Primero, necesitamos obtener un frame para renderizar.

```rust
// impl State

fn render(&mut self) -> Result<(), wgpu::SurfaceError> {
    self.window.request_redraw();

    // No podemos renderizar a menos que la superficie esté configurada
    if !self.is_surface_configured {
        return Ok(());
    }

    let output = self.surface.get_current_texture()?;
```

La función `get_current_texture` esperará a que la `surface` proporcione un nuevo `SurfaceTexture` que renderizaremos. Almacenaremos esto en `output` para más adelante.

```rust
    let view = output.texture.create_view(&wgpu::TextureViewDescriptor::default());
```

Esta línea crea una `TextureView` con configuración por defecto. Necesitamos hacer esto porque queremos controlar cómo el código de renderizado interactúa con la textura.

También necesitamos crear un `CommandEncoder` para crear los comandos reales a enviar a la GPU. La mayoría de los frameworks gráficos modernos esperan que los comandos se almacenen en un buffer de comandos antes de ser enviados a la GPU. El `encoder` construye un buffer de comandos que luego podemos enviar a la GPU.

```rust
    let mut encoder = self.device.create_command_encoder(&wgpu::CommandEncoderDescriptor {
        label: Some("Render Encoder"),
    });
```

Ahora podemos proceder a limpiar la pantalla (algo que ha tardado mucho en llegar). Necesitamos usar el `encoder` para crear un `RenderPass`. El `RenderPass` tiene todos los métodos para el dibujo real. El código para crear un `RenderPass` está un poco anidado, así que lo copiaré aquí antes de hablar sobre sus partes.

```rust
    {
        let _render_pass = encoder.begin_render_pass(&wgpu::RenderPassDescriptor {
            label: Some("Render Pass"),
            color_attachments: &[Some(wgpu::RenderPassColorAttachment {
                view: &view,
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
            depth_stencil_attachment: None,
            occlusion_query_set: None,
            timestamp_writes: None,
        });
    }

    // submit will accept anything that implements IntoIter
    self.queue.submit(std::iter::once(encoder.finish()));
    output.present();

    Ok(())
}
```

Lo primero es lo primero, hablemos sobre el bloque extra (`{}`) alrededor de `encoder.begin_render_pass(...)`. `begin_render_pass()` toma prestado `encoder` de forma mutable (es decir, `&mut self`). No podemos llamar a `encoder.finish()` hasta que liberemos ese préstamo mutable. El bloque le dice a Rust que elimine cualquier variable dentro de él cuando el código sale de ese ámbito, liberando así el préstamo mutable en `encoder` y permitiéndonos hacer `finish()`. Si no te gusta el `{}`, también puedes usar `drop(render_pass)` para lograr el mismo efecto.

Las últimas líneas del código le dicen a `wgpu` que termine el buffer de comandos y lo envíe a la cola de renderizado de la GPU.

Necesitamos actualizar el event loop nuevamente para llamar a este método. También llamaremos a `update()` antes de él.

```rust
// run()
    fn window_event(
        &mut self,
        event_loop: &ActiveEventLoop,
        _window_id: winit::window::WindowId,
        event: WindowEvent,
    ) {
        let state = match &mut self.state {
            Some(canvas) => canvas,
            None => return,
        };

        match event {
            // ...
            WindowEvent::RedrawRequested => {
                state.update();
                match state.render() {
                    Ok(_) => {}
                    // Reconfigura la superficie si está perdida u obsoleta
                    Err(wgpu::SurfaceError::Lost | wgpu::SurfaceError::Outdated) => {
                        let size = state.window.inner_size();
                        state.resize(size.width, size.height);
                    }
                    Err(e) => {
                        log::error!("Unable to render {}", e);
                    }
                }
            }
            // ...
        }
    }
```

Con todo eso, deberías obtener algo que se vea así.

![Ventana con fondo azul](./cleared-window.png)

## Espera, ¿qué está sucediendo con RenderPassDescriptor?

Algunos de ustedes podrían ser capaces de decir qué está sucediendo solo mirándolo, pero sería negligente si no lo revisara. Echemos un vistazo al código nuevamente.

```rust
&wgpu::RenderPassDescriptor {
    label: Some("Render Pass"),
    color_attachments: &[
        // ...
    ],
    depth_stencil_attachment: None,
}
```

Un `RenderPassDescriptor` solo tiene tres campos: `label`, `color_attachments` y `depth_stencil_attachment`. Los `color_attachments` describen dónde vamos a dibujar nuestro color. Usamos la `TextureView` que creamos anteriormente para asegurarnos de que renderizamos a la pantalla.

<div class="note">

El campo `color_attachments` es un arreglo "sparse". Esto te permite usar un pipeline que espera múltiples render targets y solo suministrar los que te importan.

</div>

Usaremos `depth_stencil_attachment` más adelante, pero lo estableceremos en `None` por ahora.

```rust
Some(wgpu::RenderPassColorAttachment {
    view: &view,
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
})
```

El `RenderPassColorAttachment` tiene el campo `view`, que informa a `wgpu` qué textura usar para guardar los colores. En este caso, especificamos la `view` que creamos usando `surface.get_current_texture()`. Esto significa que cualquier color que dibujemos en este attachment será dibujado en la pantalla.

El `resolve_target` es la textura que recibirá la salida resuelta. Será lo mismo que `view` a menos que el multisampling esté habilitado. No necesitamos especificar esto, así que lo dejamos en `None`.

El campo `ops` toma un objeto `wgpu::Operations`. Esto le dice a wgpu qué hacer con los colores en la pantalla (especificados por `view`). El campo `load` le dice a wgpu cómo manejar los colores almacenados del frame anterior. Actualmente, estamos limpiando la pantalla con un color azulado. El campo `store` le dice a wgpu si queremos almacenar los resultados renderizados en la `Texture` detrás de nuestra `TextureView` (en este caso, es el `SurfaceTexture`). Usamos `StoreOp::Store` porque queremos almacenar nuestros resultados de renderizado.

<div class="note">

No es raro no limpiar la pantalla si la pantalla va a estar completamente cubierta por objetos. Sin embargo, si tu escena no cubre toda la pantalla, puedes terminar con algo como esto.

![./no-clear.png](./no-clear.png)

</div>

## Errores de validación?

Si wgpu está usando Vulkan en tu máquina, es posible que encuentres errores de validación si estás ejecutando una versión anterior del SDK de Vulkan. Deberías estar usando al menos la versión `1.2.182` ya que las versiones antiguas pueden generar algunos falsos positivos. Si los errores persisten, es posible que hayas encontrado un bug en wgpu. Puedes publicar un issue en [https://github.com/gfx-rs/wgpu](https://github.com/gfx-rs/wgpu)

## Demo

<WasmExample example="tutorial2_surface"></WasmExample>

<AutoGithubLink/>

## Desafío

Crea un método `handle_mouse_moved()` para capturar eventos del ratón, y actualiza el color de limpieza usando eso. *Pista: probablemente necesitarás usar `WindowEvent::CursorMoved`*.
