# Dependencias y la ventana

## Aburrido, lo sé

Algunos de ustedes que leen esto tienen mucha experiencia abriendo ventanas en Rust y probablemente tengan su biblioteca de ventanas favorita, pero esta guía está diseñada para todos, así que es algo que necesitamos cubrir. Afortunadamente, no necesitas leer esto si sabes lo que estás haciendo. Una cosa que sí necesitas saber es que cualquier solución de ventanas que uses debe soportar el crate [raw-window-handle](https://github.com/rust-windowing/raw-window-handle).

## ¿Qué crates estamos usando?

Para las cosas de principiantes, vamos a mantener las cosas muy simples. Agregaremos cosas sobre la marcha, pero he listado las partes relevantes de `Cargo.toml` a continuación.

```toml
[dependencies]
anyhow = "1.0"
winit = { version = "0.30", features = ["android-native-activity"] }
env_logger = "0.10"
log = "0.4"
wgpu = "27.0.0"
pollster = "0.3"
```

## Usando el nuevo resolver de Rust

A partir de la versión 0.10, wgpu requiere el [resolver de características más nuevo](https://doc.rust-lang.org/cargo/reference/resolver.html#feature-resolver-version-2) de Cargo, que es el predeterminado en la edición 2021 (cualquier proyecto nuevo iniciado con Rust versión 1.56.0 o posterior). Sin embargo, si todavía estás usando la edición 2018, debes incluir `resolver = "2"` en la sección `[package]` de `Cargo.toml` si estás trabajando en un solo crate o en la sección `[workspace]` del `Cargo.toml` raíz en un espacio de trabajo.

## env_logger

Es muy importante habilitar el logging mediante `env_logger::init();`.
Cuando wgpu encuentra cualquier error, entra en pánico con un mensaje genérico, mientras registra el error real a través del crate log.
Esto significa que si no incluyes `env_logger::init()`, wgpu fallará silenciosamente, ¡dejándote muy confundido!
(Esto ya se ha hecho en el código a continuación)

## Crear un nuevo proyecto

ejecuta ```cargo new nombre_proyecto``` donde nombre_proyecto es el nombre del proyecto.
(En el ejemplo a continuación, he usado 'tutorial1_window')

## El código

Vamos a querer un lugar donde poner todo nuestro estado, así que creemos una estructura `State`.

```rust
use std::sync::Arc;

use winit::{
    application::ApplicationHandler, event::*, event_loop::{ActiveEventLoop, EventLoop}, keyboard::{KeyCode, PhysicalKey}, window::Window
};

#[cfg(target_arch = "wasm32")]
use wasm_bindgen::prelude::*;

// Esto almacenará el estado de nuestro juego
pub struct State {
    window: Arc<Window>,
}

impl State {
    // No necesitamos que esto sea async ahora mismo,
    // pero lo necesitaremos en el próximo tutorial
    pub async fn new(window: Arc<Window>) -> anyhow::Result<Self> {
        Ok(Self {
            window,
        })
    }

    pub fn resize(&mut self, _width: u32, _height: u32) {
        // Haremos cosas aquí en el próximo tutorial
    }

    pub fn render(&mut self) {
        self.window.request_redraw();

        // Haremos más cosas aquí en el próximo tutorial
    }
}

// ...
```

No hay mucho sucediendo aquí, pero una vez que comencemos a usar WGPU, comenzaremos a llenarlo bastante rápido. La mayoría de los métodos en esta estructura son marcadores de posición, aunque en `render()` le pedimos a la ventana que dibuje otro fotograma lo antes posible, ya que winit solo dibuja un fotograma a menos que la ventana se redimensione o solicitemos que dibuje otro.

Ahora que tenemos nuestra estructura `State`, necesitamos decirle a winit cómo usarla. Crearemos una estructura `App` para esto.

```rust
pub struct App {
    #[cfg(target_arch = "wasm32")]
    proxy: Option<winit::event_loop::EventLoopProxy<State>>,
    state: Option<State>,
}

impl App {
    pub fn new(#[cfg(target_arch = "wasm32")] event_loop: &EventLoop<State>) -> Self {
        #[cfg(target_arch = "wasm32")]
        let proxy = Some(event_loop.create_proxy());
        Self {
            state: None,
            #[cfg(target_arch = "wasm32")]
            proxy,
        }
    }
}
```

Entonces, la estructura `App` tiene dos campos: `state` y `proxy`.

La variable `state` almacena nuestra estructura `State` como una opción. La razón por la que necesitamos una opción es que `State::new()` necesita una ventana y no podemos crear una ventana hasta que la aplicación llegue al estado `Resumed`. Entraremos en más detalles sobre eso en un momento.

La variable `proxy` solo se necesita en la web. La razón de esto es que crear recursos WGPU es un proceso asíncrono. De nuevo, entraremos en eso en un momento.

Ahora que tenemos una estructura `App`, necesitamos implementar el trait `ApplicationHandler`. Esto nos dará una variedad de funciones diferentes que podemos usar para obtener eventos de la aplicación, como pulsaciones de teclas, movimientos del mouse y varios eventos del ciclo de vida. Comenzaremos cubriendo primero los métodos `resumed` y `user_event`.

```rust
impl ApplicationHandler<State> for App {
    fn resumed(&mut self, event_loop: &ActiveEventLoop) {
        #[allow(unused_mut)]
        let mut window_attributes = Window::default_attributes();

        #[cfg(target_arch = "wasm32")]
        {
            use wasm_bindgen::JsCast;
            use winit::platform::web::WindowAttributesExtWebSys;

            const CANVAS_ID: &str = "canvas";

            let window = wgpu::web_sys::window().unwrap_throw();
            let document = window.document().unwrap_throw();
            let canvas = document.get_element_by_id(CANVAS_ID).unwrap_throw();
            let html_canvas_element = canvas.unchecked_into();
            window_attributes = window_attributes.with_canvas(Some(html_canvas_element));
        }

        let window = Arc::new(event_loop.create_window(window_attributes).unwrap());

        #[cfg(not(target_arch = "wasm32"))]
        {
            // Si no estamos en web, podemos usar pollster para
            // esperar el futuro
            self.state = Some(pollster::block_on(State::new(window)).unwrap());
        }

        #[cfg(target_arch = "wasm32")]
        {
            // Ejecutar el futuro de forma asíncrona y usar el
            // proxy para enviar los resultados al event loop
            if let Some(proxy) = self.proxy.take() {
                wasm_bindgen_futures::spawn_local(async move {
                    assert!(proxy
                        .send_event(
                            State::new(window)
                                .await
                                .expect("Unable to create canvas!!!")
                        )
                        .is_ok())
                });
            }
        }
    }

    #[allow(unused_mut)]
    fn user_event(&mut self, _event_loop: &ActiveEventLoop, mut event: State) {
        // Aquí es donde termina proxy.send_event()
        #[cfg(target_arch = "wasm32")]
        {
            event.window.request_redraw();
            event.resize(
                event.window.inner_size().width,
                event.window.inner_size().height,
            );
        }
        self.state = Some(event);
    }

    // ...
}
```

El método `resumed` parece que hace mucho, pero solo hace algunas cosas:

- Define atributos sobre la ventana, incluyendo algunas cosas específicas de la web.
- Usamos esos atributos para crear la ventana.
- Creamos un futuro que crea nuestra estructura `State`
- En nativo usamos pollster para esperar el futuro
- En web ejecutamos el futuro de forma asíncrona, que envía los resultados a la función `user_event`

La función `user_event` solo sirve como un punto de llegada para nuestro futuro `State`. `resumed` no es async, así que necesitamos descargar el futuro y enviar los resultados a algún lugar.

A continuación, hablaremos sobre `window_event`.

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
            WindowEvent::CloseRequested => event_loop.exit(),
            WindowEvent::Resized(size) => state.resize(size.width, size.height),
            WindowEvent::RedrawRequested => {
                state.render();
            }
            WindowEvent::KeyboardInput {
                event:
                    KeyEvent {
                        physical_key: PhysicalKey::Code(code),
                        state: key_state,
                        ..
                    },
                ..
            } => match (code, key_state.is_pressed()) {
                (KeyCode::Escape, true) => event_loop.exit(),
                _ => {}
            },
            _ => {}
        }
    }
}
```

Aquí es donde podemos procesar eventos como entradas de teclado y movimientos del mouse, así como otros eventos de ventana, como cuando la ventana quiere dibujar o se redimensiona. Podemos llamar a los métodos que definimos en `State` aquí.

A continuación, necesitamos ejecutar nuestro código. Crearemos una función `run()` para hacer eso.

```rust
pub fn run() -> anyhow::Result<()> {
    #[cfg(not(target_arch = "wasm32"))]
    {
        env_logger::init();
    }
    #[cfg(target_arch = "wasm32")]
    {
        console_log::init_with_level(log::Level::Info).unwrap_throw();
    }

    let event_loop = EventLoop::with_user_event().build()?;
    let mut app = App::new(
        #[cfg(target_arch = "wasm32")]
        &event_loop,
    );
    event_loop.run_app(&mut app)?;

    Ok(())
}
```

Esta función configura el logger, así como crea el `event_loop` y nuestra `app` y luego ejecuta nuestra `app` hasta su finalización.

## Soporte agregado para la web

Para que nuestra aplicación se ejecute en la web, necesitamos hacer algunos cambios en nuestro `Cargo.toml`:

```toml
[lib]
crate-type = ["cdylib", "rlib"]
```

Estas líneas le dicen a Cargo que queremos permitir que nuestro crate construya una biblioteca estática nativa de Rust (rlib) y una biblioteca compatible con C/C++ (cdylib). Necesitamos rlib si queremos ejecutar wgpu en un entorno de escritorio. Necesitamos cdylib para crear el Web Assembly que ejecutará el navegador.

<div class="note">

## Web Assembly

Web Assembly, es decir, WASM, es un formato binario compatible con la mayoría de los navegadores modernos que permite que lenguajes de bajo nivel como Rust se ejecuten en una página web. Esto nos permite escribir la mayor parte de nuestra aplicación en Rust y usar algunas líneas de Javascript para ejecutarla en un navegador web.

</div>

Ahora, todo lo que necesitamos son algunas dependencias más que son específicas para ejecutarse en WASM:

```toml
# Esto debería ir en el Cargo.toml en el directorio raíz
[profile.release]
strip = true

[dependencies]
# las otras dependencias regulares...

[target.'cfg(target_arch = "wasm32")'.dependencies]
console_error_panic_hook = "0.1.6"
console_log = "1.0"
wgpu = { version = "27.0.0", features = ["webgl"]}
wasm-bindgen = "0.2"
wasm-bindgen-futures = "0.4.30"
web-sys = { version = "0.3", features = [
    "Document",
    "Window",
    "Element",
]}
```

La línea `[target.'cfg(target_arch = "wasm32")'.dependencies]` le dice a Cargo que solo incluya estas dependencias si estamos apuntando a la arquitectura `wasm32`. Las siguientes dependencias simplemente hacen que la interfaz con JavaScript sea mucho más fácil.

- [console_error_panic_hook](https://docs.rs/console_error_panic_hook) configura la macro `panic!` para enviar errores a la consola de javascript. Sin esto, cuando encuentres pánicos, te quedarás en la oscuridad sobre qué los causó.
- [console_log](https://docs.rs/console_log) implementa la API [log](https://docs.rs/log). Envía todos los logs a la consola de javascript. Se puede configurar para enviar solo logs de un nivel de log particular. Esto también es genial para depuración.
- Necesitamos habilitar la característica WebGL en wgpu si queremos ejecutar en la mayoría de los navegadores actuales. El soporte está en desarrollo para usar la API WebGPU directamente, pero eso solo es posible en versiones experimentales de navegadores como Firefox Nightly y Chrome Canary.<br>
  Eres bienvenido a probar este código en estos navegadores (y los desarrolladores de wgpu también lo apreciarían), pero por simplicidad, voy a seguir usando la característica WebGL hasta que la API WebGPU llegue a un estado más estable.<br>
  Si quieres más detalles, consulta la guía para compilar para la web en [el repositorio de wgpu](https://github.com/gfx-rs/wgpu/wiki/Running-on-the-Web-with-WebGPU-and-WebGL)
- [wasm-bindgen](https://docs.rs/wasm-bindgen) es la dependencia más importante en esta lista. Es responsable de generar el código repetitivo que le dirá al navegador cómo usar nuestro crate. También nos permite exponer métodos en Rust que se pueden usar en JavaScript y viceversa.<br>
  No entraré en los detalles específicos de wasm-bindgen, así que si necesitas una introducción (o simplemente un repaso), consulta [esto](https://wasm-bindgen.github.io/wasm-bindgen/)
- [web-sys](https://docs.rs/web-sys) es un crate con muchos métodos y estructuras disponibles en una aplicación javascript normal: `get_element_by_id`, `append_child`. Las características listadas son solo el mínimo indispensable de lo que necesitamos actualmente.

## Más código

Creemos una función para ejecutar nuestro código en web.

```rust
#[cfg(target_arch = "wasm32")]
#[wasm_bindgen(start)]
pub fn run_web() -> Result<(), wasm_bindgen::JsValue> {
    console_error_panic_hook::set_once();
    run().unwrap_throw();

    Ok(())
}
```

Esto configurará `console_error_panic_hook` para que cuando nuestro código entre en pánico lo veamos en la consola del navegador. También llamará a la otra función `run()`.

## Wasm Pack

Ahora puedes construir una aplicación wgpu con solo wasm-bindgen, pero me encontré con algunos problemas al hacer eso. Por un lado, necesitas instalar wasm-bindgen en tu computadora además de incluirlo como dependencia. La versión que instalas como dependencia **necesita** coincidir exactamente con la versión que instalaste. De lo contrario, tu compilación fallará.

Para evitar esta limitación y hacer la vida de todos los que leen esto más fácil, opté por agregar [wasm-pack](https://drager.github.io/wasm-pack/) a la mezcla. Wasm-pack maneja la instalación de la versión correcta de wasm-bindgen por ti, y soporta la compilación para diferentes tipos de objetivos web: navegador, NodeJS y empaquetadores como webpack.

Para usar wasm-pack, primero, necesitas [instalarlo](https://drager.github.io/wasm-pack/).

Una vez que hayas hecho eso, podemos usarlo para construir nuestro crate. Si solo tienes un crate en tu proyecto, puedes usar simplemente `wasm-pack build`. Si estás usando un espacio de trabajo, tendrás que especificar qué crate quieres construir. Imagina que tu crate es un directorio llamado `game`. Entonces usarías:

```bash
wasm-pack build game
```

Una vez que wasm-pack haya terminado de construir, tendrás un directorio `pkg` en el mismo directorio que tu crate. Esto tiene todo el código javascript necesario para ejecutar el código WASM. Luego importarías el módulo WASM en javascript:

```js
const init = await import('./pkg/game.js');
init().then(() => console.log("WASM Loaded"));
```

Este sitio usa [Vuepress](https://vuepress.vuejs.org/), así que cargo el WASM en un componente Vue. Cómo manejes tu WASM dependerá de lo que quieras hacer. Si quieres ver cómo estoy haciendo las cosas, echa un vistazo a [esto](https://github.com/sotrh/learn-wgpu/blob/master/docs/.vuepress/components/WasmExample.vue).

<div class="note">

Si tienes la intención de usar tu módulo WASM en un sitio web HTML simple, necesitarás decirle a wasm-pack que apunte a la web:

```bash
wasm-pack build --target web
```

Luego necesitarás ejecutar el código WASM en un módulo ES6:

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta http-equiv="X-UA-Compatible" content="IE=edge" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Learn WGPU</title>
    <style>
      * {
        padding: 0;
        margin: 0;
      }
      canvas {
        background-color: black;
        width: 100%;
        height: 100%;
      }
    </style>
  </head>

  <body id="wasm-example">
    <canvas id="canvas"></canvas>
    <script type="module">
      import init from "./pkg/tutorial2_surface.js";
      init().then(() => {
        console.log("WASM Loaded");
      });
    </script>
  </body>
</html>
```

</div>

## Demo

¡Presiona el botón a continuación y verás el código ejecutándose!

<WasmExample example="tutorial1_window"></WasmExample>

<AutoGithubLink/>
