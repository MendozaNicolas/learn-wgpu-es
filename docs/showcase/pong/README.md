# Pong

![A screenshot of pong](./pong.png)

<div class="warning">

This example is not working as of `wgpu = "27.0.0"`. If the crate updates to
the latest version I'll switch it over, but given that the crate maintainer
is directing users to use [glypon](https://github.com/grovesNL/glyphon?tab=readme-ov-file)
I'm considering either switching to using that, or writing my own text code.

</div>

Prácticamente el "¡Hola Mundo!" de los juegos. Pong ha sido recreado miles de veces. Yo conozco Pong. Tú conoces Pong. Todos conocemos Pong. Dicho esto, esta vez quise hacer un esfuerzo un poco mayor que el que la mayoría de la gente suele hacer. Este showcase tiene un sistema de menú básico, sonidos y diferentes estados de juego.

La arquitectura no es la mejor ya que me adhería a la mentalidad de "hacer las cosas hechas". Si fuera a rehacer este proyecto, cambiaría muchas cosas. Independientemente, entremos en el postmortem.

## La Arquitectura

Estuve experimentando con separar el estado del código de renderizado. Terminó siendo similar a un modelo de Entity Component System.

Tenía una clase `State` con todos los objetos en la escena. Esto incluía la pelota y las paletas, así como el texto de las puntuaciones e incluso el menú. `State` también incluía un campo `game_state` de tipo `GameState`.

```rust
#[derive(Debug, Copy, Clone, Eq, PartialEq)]
pub enum GameState {
    MainMenu,
    Serving,
    Playing,
    GameOver,
    Quiting,
}
```

La clase `State` no tenía ningún método ya que estaba tomando un enfoque más orientado a datos. En su lugar, creé un trait `System` y creé múltiples structs que lo implementaban.

```rust
pub trait System {
    #[allow(unused_variables)]
    fn start(&mut self, state: &mut state::State) {}
    fn update_state(
        &self,
        input: &input::Input,
        state: &mut state::State,
        events: &mut Vec<state::Event>,
    );
}
```

Los sistemas estarían a cargo de controlar la actualización de los estados de los diferentes objetos (posición, visibilidad, etc), así como actualizar el campo `game_state`. Creé todos los sistemas al inicio y usé un `match` en `game_state` para determinar cuáles deberían estar permitidos para ejecutarse (el `visiblity_system` siempre se ejecuta ya que siempre es necesario).

```rust
visiblity_system.update_state(&input, &mut state, &mut events);
match state.game_state {
    state::GameState::MainMenu => {
        menu_system.update_state(&input, &mut state, &mut events);
        if state.game_state == state::GameState::Serving {
            serving_system.start(&mut state);
        }
    },
    state::GameState::Serving => {
        serving_system.update_state(&input, &mut state, &mut events);
        play_system.update_state(&input, &mut state, &mut events);
        if state.game_state == state::GameState::Playing {
            play_system.start(&mut state);
        }
    },
    state::GameState::Playing => {
        ball_system.update_state(&input, &mut state, &mut events);
        play_system.update_state(&input, &mut state, &mut events);
        if state.game_state == state::GameState::Serving {
            serving_system.start(&mut state);
        } else if state.game_state == state::GameState::GameOver {
            game_over_system.start(&mut state);
        }
    },
    state::GameState::GameOver => {
        game_over_system.update_state(&input, &mut state, &mut events);
        if state.game_state == state::GameState::MainMenu {
            menu_system.start(&mut state);
        }
    },
    state::GameState::Quiting => {},
}
```

Definitivamente no es el código más limpio, pero funciona.

Terminé teniendo 6 sistemas en total.

1. Agregué el `VisibilitySystem` cerca del final del desarrollo. Hasta ese momento, todos los sistemas tenían que establecer el campo `visible` de los objetos. Eso era molesto y saturaba la lógica. En su lugar, decidí crear el `VisiblitySystem` para manejar eso.

2. El `MenuSystem` manejaba controlar qué texto estaba enfocado y qué sucedería cuando el usuario presionara la tecla enter. Si el botón `Play` estaba enfocado, presionar enter cambiaría `game_state` a `GameState::Serving` que iniciaría el juego. El botón `Quit` cambiaría a `GameState::Quiting`.

3. El `ServingSystem` establece la posición de la pelota a `(0.0, 0.0)`, actualiza los textos de puntuación y cambia a `GameState::Playing` después de un temporizador.

4. El `PlaySystem` controla los jugadores. Les permite moverse y evita que salgan del espacio de juego. Este sistema se ejecuta tanto en `GameState::Playing` como en `GameState::Serving`. Hice esto para permitir que los jugadores se reposicionaran antes del saque. El `PlaySystem` también cambiará a `GameState::GameOver` cuando la puntuación de uno de los jugadores sea mayor que 2.

5. El sistema `BallSystem` controla el movimiento de la pelota así como su rebote en las paredes/jugadores. También actualiza la puntuación y cambia a `GameState::Serving` cuando la pelota se sale del lado de la pantalla.

6. El sistema `GameOver` actualiza el `win_text` y cambia a `GameState::MainMenu` después de un retraso.

Encontré el enfoque de sistemas muy agradable para trabajar. Mi implementación no era la mejor, pero me gustaría trabajar con ella nuevamente. Incluso podría implementar mi propio ECS.

## Input

El trait `System`, originalmente tenía un método `process_input`. Esto se convirtió en un problema cuando estaba implementando permitir que los jugadores se muevan entre saques. Los jugadores se quedaban atrapados cuando el `game_state` cambiaba de `Serving` a `Playing` ya que las entradas se quedaban atrapadas. Solo llamaba a `process_input` en sistemas que estaban actualmente en uso. Cambiar eso sería complicado, así que decidí mover todo el código de entrada a su propio struct.

```rust
use winit::event::{VirtualKeyCode, ElementState};

#[derive(Debug, Default)]
pub struct Input {
    pub p1_up_pressed: bool,
    pub p1_down_pressed: bool,
    pub p2_up_pressed: bool,
    pub p2_down_pressed: bool,
    pub enter_pressed: bool,
}

impl Input {
    pub fn new() -> Self {
        Default::default()
    }

    pub fn update(&mut self, key: VirtualKeyCode, state: ElementState) -> bool {
        let pressed = state == ElementState::Pressed;
        match key {
            VirtualKeyCode::Up => {
                self.p2_up_pressed = pressed;
                true
            }
            VirtualKeyCode::Down => {
                self.p2_down_pressed = pressed;
                true
            }
            VirtualKeyCode::W => {
                self.p1_up_pressed = pressed;
                true
            }
            VirtualKeyCode::S => {
                self.p1_down_pressed = pressed;
                true
            }
            VirtualKeyCode::Return => {
                self.enter_pressed = pressed;
                true
            }
            _ => false
        }
    }

    pub fn ui_up_pressed(&self) -> bool {
        self.p1_up_pressed || self.p2_up_pressed
    }

    pub fn ui_down_pressed(&self) -> bool {
        self.p1_down_pressed || self.p2_down_pressed
    }
}
```

Funciona muy bien. Simplemente paso este struct al método `update_state`.

## Render

Usé [wgpu_glyph](https://docs.rs/wgpu_glyph) para el texto y quads blancos para la pelota y las paletas. No hay mucho que decir aquí, es Pong después de todo.

Sí experimenté con batching, sin embargo. Fue totalmente excesivo para este proyecto, pero fue una buena experiencia de aprendizaje. Aquí está el código si te interesa.

```rust
pub struct QuadBufferBuilder {
    vertex_data: Vec<Vertex>,
    index_data: Vec<u32>,
    current_quad: u32,
}

impl QuadBufferBuilder {
    pub fn new() -> Self {
        Self {
            vertex_data: Vec::new(),
            index_data: Vec::new(),
            current_quad: 0,
        }
    }

    pub fn push_ball(self, ball: &state::Ball) -> Self {
        if ball.visible {
            let min_x = ball.position.x - ball.radius;
            let min_y = ball.position.y - ball.radius;
            let max_x = ball.position.x + ball.radius;
            let max_y = ball.position.y + ball.radius;

            self.push_quad(min_x, min_y, max_x, max_y)
        } else {
            self
        }
    }

    pub fn push_player(self, player: &state::Player) -> Self {
        if player.visible {
            self.push_quad(
                player.position.x - player.size.x * 0.5,
                player.position.y - player.size.y * 0.5,
                player.position.x + player.size.x * 0.5,
                player.position.y + player.size.y * 0.5,
            )
        } else {
            self
        }
    }

    pub fn push_quad(mut self, min_x: f32, min_y: f32, max_x: f32, max_y: f32) -> Self {
        self.vertex_data.extend(&[
            Vertex {
                position: (min_x, min_y).into(),
            },
            Vertex {
                position: (max_x, min_y).into(),
            },
            Vertex {
                position: (max_x, max_y).into(),
            },
            Vertex {
                position: (min_x, max_y).into(),
            },
        ]);
        self.index_data.extend(&[
            self.current_quad * 4 + 0,
            self.current_quad * 4 + 1,
            self.current_quad * 4 + 2,
            self.current_quad * 4 + 0,
            self.current_quad * 4 + 2,
            self.current_quad * 4 + 3,
        ]);
        self.current_quad += 1;
        self
    }

    pub fn build(self, device: &wgpu::Device) -> (StagingBuffer, StagingBuffer, u32) {
        (
            StagingBuffer::new(device, &self.vertex_data),
            StagingBuffer::new(device, &self.index_data),
            self.index_data.len() as u32,
        )
    }
}
```

## Sound

Usé [rodio](https://docs.rs/rodio) para el sonido. Creé una clase `SoundPack` para almacenar los sonidos. Decidir cómo lograr que los sonidos se reprodujeran requirió algo de pensamiento. Elegí pasar un `Vec<state::Event>` al método `update_state`. El sistema entonces empujaría un evento al `Vec`. El enum `Event` se enumera a continuación.

```rust
#[derive(Debug, Copy, Clone)]
pub enum Event {
    ButtonPressed,
    FocusChanged,
    BallBounce(cgmath::Vector2<f32>),
    Score(u32),
}
```

Iba a hacer que `BallBounce` reprodujera un sonido posicionado usando un `SpatialSink`, pero tenía problemas de recorte, y quería terminar con el proyecto. Aparte de eso, el sistema de eventos funcionó bien.

## WASM Support

Este ejemplo funciona en la web, pero hay algunos pasos que necesitaba seguir para que las cosas funcionaran. El primero fue que necesitaba cambiar a usar un `lib.rs` en lugar de solo `main.rs`. Opté por usar [wasm-pack](https://rustwasm.github.io/wasm-pack/) para crear el web assembly. Habría podido mantener el formato anterior usando wasm-bindgen directamente, pero tenía problemas al usar la versión incorrecta de wasm-bindgen, así que elegí quedarme con wasm-pack.

Para que wasm-pack funcione correctamente, primero necesitaba agregar algunas dependencias:

```toml[dependencies]
anyhow = "1.0"
env_logger = "0.10"
winit = { version = "0.30", features = ["android-native-activity"] }
anyhow = "1.0"
bytemuck = { version = "1.24", features = [ "derive" ] }
cgmath = "0.18"
pollster = "0.3"
wgpu = { version = "27.0.0", features = ["spirv"]}
wgpu_glyph = "0.19"
rand = "0.8"
rodio = { version = "0.15", default-features = false, features = ["wav"] }
log = "0.4"
instant = "0.1"

[target.'cfg(target_arch = "wasm32")'.dependencies]
console_error_panic_hook = "0.1.6"
console_log = "1.0"
getrandom = { version = "0.2", features = ["js"] }
rodio = { version = "0.15", default-features = false, features = ["wasm-bindgen", "wav"] }
wasm-bindgen-futures = "0.4.20"
wasm-bindgen = "0.2"
web-sys = { version = "0.3", features = [
    "Document",
    "Window",
    "Element",
]}
wgpu = { version = "27.0.0", features = ["spirv", "webgl"]}

[build-dependencies]
anyhow = "1.0"
fs_extra = "1.2"
glob = "0.3"
rayon = "1.4"
naga = { version = "27.0", features = ["glsl-in", "spv-out", "wgsl-out"]}

```

Destacaré algunos de estos:

- rand: Si deseas usar rand en la web, necesitas incluir getrandom directamente y habilitar su característica `js`.
- rodio: Tuve que deshabilitar todas las características para la compilación WASM y luego habilitarlas por separado. La característica `mp3` específicamente no funcionaba para mí. Podría haber habido una solución alternativa, pero como no estoy usando mp3 en este ejemplo, elegí solo usar wav.
- instant: Este crate es básicamente solo un envoltorio alrededor de `std::time::Instant`. En una compilación normal, es solo un alias de tipo. En compilaciones web utiliza las funciones de tiempo del navegador.
- cfg-if: Este es un crate conveniente para hacer que el código específico de la plataforma sea menos horrible de escribir.
- env_logger and console_log: env_logger no funciona en web assembly, así que necesitamos usar un registrador diferente. console_log es el que se usa en los tutoriales de web assembly, así que fui con ese.
- wasm-bindgen: Este crate es el pegamento que hace que el código Rust funcione en la web. Si estás compilando usando el comando wasm-bindgen, necesitas asegurarte de que la versión del comando de wasm-bindgen coincida con la versión en Cargo.toml **exactamente**, de lo contrario tendrás problemas. Si usas wasm-pack descargará el binario wasm-bindgen apropiado para usar con tu crate.
- web-sys: Esto tiene funciones y tipos que te permiten usar diferentes métodos disponibles en js como "getElementById()".

Ahora que eso está fuera del camino, hablemos de algo de código. Primero, necesitamos crear una función que inicie nuestro bucle de eventos.

```rust
#[cfg(target_arch="wasm32")]
use wasm_bindgen::prelude::*;

#[cfg_attr(target_arch="wasm32", wasm_bindgen(start))]
pub fn start() {
    // Snipped...
}
```

El `wasm_bindgen(start)` le dice a wasm-bindgen que esta función debe iniciarse tan pronto como el módulo web assembly se carga en javascript. La mayoría del código dentro de esta función es igual a lo que encontrarías en otros ejemplos en este sitio, pero hay cosas específicas que necesitamos hacer en la web.

```rust
cfg_if::cfg_if! {
    if #[cfg(target_arch = "wasm32")] {
        console_log::init_with_level(log::Level::Warn).expect("Could't initialize logger");
        std::panic::set_hook(Box::new(console_error_panic_hook::hook));
    } else {
        env_logger::init();
    }
}
```

Este código debe ejecutarse antes de intentar hacer algo significativo. Configura el registrador según la arquitectura para la que estés compilando. La mayoría de las arquitecturas usarán `env_logger`. La arquitectura `wasm32` usará `console_log`. También es importante que le digamos a Rust que reenvíe los panics a javascript. Si no hiciéramos esto, no tendríamos idea de cuándo el código Rust entra en pánico.

A continuación, creamos una ventana. Gran parte de ella es como lo hemos hecho antes, pero como estamos compatibilizando pantalla completa, necesitamos hacer algunos pasos adicionales.

```rust
let event_loop = EventLoop::new();
let monitor = event_loop.primary_monitor().unwrap();
let video_mode = monitor.video_modes().next();
let size = video_mode.clone().map_or(PhysicalSize::new(800, 600), |vm| vm.size());
let window = WindowBuilder::new()
    .with_visible(false)
    .with_title("Pong")
    .with_fullscreen(video_mode.map(|vm| Fullscreen::Exclusive(vm)))
    .build(&event_loop)
    .unwrap();

// WASM builds don't have access to monitor information, so
// we should specify a fallback resolution
if window.fullscreen().is_none() {
    window.set_inner_size(PhysicalSize::new(512, 512));
}
```

Luego tenemos que hacer algunas cosas específicas de la web si estamos en esa plataforma.

```rust
#[cfg(target_arch = "wasm32")]
{
    use winit::platform::web::WindowExtWebSys;
    web_sys::window()
        .and_then(|win| win.document())
        .and_then(|doc| {
            let dst = doc.get_element_by_id("wasm-example")?;
            let canvas = web_sys::Element::from(window.canvas()?);
            dst.append_child(&canvas).ok()?;

            // Request fullscreen, if denied, continue as normal
            match canvas.request_fullscreen() {
                Ok(_) => {},
                Err(_) => ()
            }

            Some(())
        })
        .expect("Couldn't append canvas to document body.");
}
```

Todo lo demás funciona igual.

## Summary

Un proyecto divertido en el que trabajar. Estaba excesivamente arquitectado y era bastante difícil hacer cambios, pero una buena experiencia de todas formas.

<!-- Try the code down below! (Controls currently require a keyboard.)

<WasmExample example="pong"></WasmExample> -->