# Una Cámara Mejor

He estado posponiendo esto por un tiempo. Implementar una cámara no está específicamente relacionado con usar WGPU correctamente, pero me ha estado molestando, así que hagámoslo.

`lib.rs` se está volviendo un poco abarrotado, así que vamos a crear un archivo `camera.rs` para poner nuestro código de cámara. Las primeras cosas que vamos a poner en él son algunas importaciones y nuestro `OPENGL_TO_WGPU_MATRIX`.

```rust
use cgmath::*;
use winit::event::*;
use winit::dpi::PhysicalPosition;
use instant::Duration;
use std::f32::consts::FRAC_PI_2;

#[rustfmt::skip]
pub const OPENGL_TO_WGPU_MATRIX: cgmath::Matrix4<f32> = cgmath::Matrix4::from_cols(
    cgmath::Vector4::new(1.0, 0.0, 0.0, 0.0),
    cgmath::Vector4::new(0.0, 1.0, 0.0, 0.0),
    cgmath::Vector4::new(0.0, 0.0, 0.5, 0.0),
    cgmath::Vector4::new(0.0, 0.0, 0.5, 1.0),
);

const SAFE_FRAC_PI_2: f32 = FRAC_PI_2 - 0.0001;
```

<div class="note">

`std::time::Instant` entra en pánico en WASM, así que usaremos el [instant crate](https://docs.rs/instant). Querrás incluirlo en tu `Cargo.toml`:

```toml
[dependencies]
# ...
instant = "0.1"

[target.'cfg(target_arch = "wasm32")'.dependencies]
instant = { version = "0.1", features = [ "wasm-bindgen" ] }
```

</div>

## La Cámara

A continuación, necesitamos crear una nueva estructura `Camera`. Vamos a usar una cámara de estilo FPS, así que guardaremos la posición, el yaw (rotación horizontal) y el pitch (rotación vertical). Tendremos un método `calc_matrix` para crear nuestra matriz de vista.

```rust
#[derive(Debug)]
pub struct Camera {
    pub position: Point3<f32>,
    yaw: Rad<f32>,
    pitch: Rad<f32>,
}

impl Camera {
    pub fn new<
        V: Into<Point3<f32>>,
        Y: Into<Rad<f32>>,
        P: Into<Rad<f32>>,
    >(
        position: V,
        yaw: Y,
        pitch: P,
    ) -> Self {
        Self {
            position: position.into(),
            yaw: yaw.into(),
            pitch: pitch.into(),
        }
    }

    pub fn calc_matrix(&self) -> Matrix4<f32> {
        let (sin_pitch, cos_pitch) = self.pitch.0.sin_cos();
        let (sin_yaw, cos_yaw) = self.yaw.0.sin_cos();

        Matrix4::look_to_rh(
            self.position,
            Vector3::new(
                cos_pitch * cos_yaw,
                sin_pitch,
                cos_pitch * sin_yaw
            ).normalize(),
            Vector3::unit_y(),
        )
    }
}
```

## La Proyección

He decidido separar la proyección de la cámara. La proyección solo necesita cambiar si la ventana se redimensiona, así que vamos a crear una estructura `Projection`.

```rust
pub struct Projection {
    aspect: f32,
    fovy: Rad<f32>,
    znear: f32,
    zfar: f32,
}

impl Projection {
    pub fn new<F: Into<Rad<f32>>>(
        width: u32,
        height: u32,
        fovy: F,
        znear: f32,
        zfar: f32,
    ) -> Self {
        Self {
            aspect: width as f32 / height as f32,
            fovy: fovy.into(),
            znear,
            zfar,
        }
    }

    pub fn resize(&mut self, width: u32, height: u32) {
        self.aspect = width as f32 / height as f32;
    }

    pub fn calc_matrix(&self) -> Matrix4<f32> {
        OPENGL_TO_WGPU_MATRIX * perspective(self.fovy, self.aspect, self.znear, self.zfar)
    }
}
```

Una cosa a notar: `cgmath` actualmente devuelve una matriz de proyección diestra de la función `perspective`. Esto significa que el eje z apunta fuera de la pantalla. Si quieres que el eje z sea *hacia* la pantalla (es decir, una matriz de proyección zurda), tendrás que programar la tuya propia.

Puedes notar la diferencia entre un sistema de coordenadas diestro y uno zurdo usando tus manos. Apunta tu pulgar hacia la derecha. Este es el eje x. Apunta tu dedo índice hacia arriba. Este es el eje y. Extiende tu dedo medio. Este es el eje z. En tu mano derecha, tu dedo medio debe estar apuntando hacia ti. En tu mano izquierda, debe estar apuntando hacia afuera.

![./left_right_hand.gif](./left_right_hand.gif)

# El Controlador de Cámara

Nuestra cámara es diferente, así que necesitaremos un nuevo controlador de cámara. Agrega lo siguiente a `camera.rs`.

```rust
#[derive(Debug)]
pub struct CameraController {
    amount_left: f32,
    amount_right: f32,
    amount_forward: f32,
    amount_backward: f32,
    amount_up: f32,
    amount_down: f32,
    rotate_horizontal: f32,
    rotate_vertical: f32,
    scroll: f32,
    speed: f32,
    sensitivity: f32,
}

impl CameraController {
    pub fn new(speed: f32, sensitivity: f32) -> Self {
        Self {
            amount_left: 0.0,
            amount_right: 0.0,
            amount_forward: 0.0,
            amount_backward: 0.0,
            amount_up: 0.0,
            amount_down: 0.0,
            rotate_horizontal: 0.0,
            rotate_vertical: 0.0,
            scroll: 0.0,
            speed,
            sensitivity,
        }
    }

    pub fn process_keyboard(&mut self, key: VirtualKeyCode, state: ElementState) -> bool{
        let amount = if state == ElementState::Pressed { 1.0 } else { 0.0 };
        match key {
            VirtualKeyCode::W | VirtualKeyCode::Up => {
                self.amount_forward = amount;
                true
            }
            VirtualKeyCode::S | VirtualKeyCode::Down => {
                self.amount_backward = amount;
                true
            }
            VirtualKeyCode::A | VirtualKeyCode::Left => {
                self.amount_left = amount;
                true
            }
            VirtualKeyCode::D | VirtualKeyCode::Right => {
                self.amount_right = amount;
                true
            }
            VirtualKeyCode::Space => {
                self.amount_up = amount;
                true
            }
            VirtualKeyCode::LShift => {
                self.amount_down = amount;
                true
            }
            _ => false,
        }
    }

    pub fn handle_mouse(&mut self, mouse_dx: f64, mouse_dy: f64) {
        self.rotate_horizontal = mouse_dx as f32;
        self.rotate_vertical = mouse_dy as f32;
    }

    pub fn handle_mouse_scroll(&mut self, delta: &MouseScrollDelta) {
        self.scroll = -match delta {
            // I'm assuming a line is about 100 pixels
            MouseScrollDelta::LineDelta(_, scroll) => scroll * 100.0,
            MouseScrollDelta::PixelDelta(PhysicalPosition {
                y: scroll,
                ..
            }) => *scroll as f32,
        };
    }

    pub fn update_camera(&mut self, camera: &mut Camera, dt: Duration) {
        let dt = dt.as_secs_f32();

        // Move forward/backward and left/right
        let (yaw_sin, yaw_cos) = camera.yaw.0.sin_cos();
        let forward = Vector3::new(yaw_cos, 0.0, yaw_sin).normalize();
        let right = Vector3::new(-yaw_sin, 0.0, yaw_cos).normalize();
        camera.position += forward * (self.amount_forward - self.amount_backward) * self.speed * dt;
        camera.position += right * (self.amount_right - self.amount_left) * self.speed * dt;

        // Move in/out (aka. "zoom")
        // Note: this isn't an actual zoom. The camera's position
        // changes when zooming. I've added this to make it easier
        // to get closer to an object you want to focus on.
        let (pitch_sin, pitch_cos) = camera.pitch.0.sin_cos();
        let scrollward = Vector3::new(pitch_cos * yaw_cos, pitch_sin, pitch_cos * yaw_sin).normalize();
        camera.position += scrollward * self.scroll * self.speed * self.sensitivity * dt;
        self.scroll = 0.0;

        // Move up/down. Since we don't use roll, we can just
        // modify the y coordinate directly.
        camera.position.y += (self.amount_up - self.amount_down) * self.speed * dt;

        // Rotate
        camera.yaw += Rad(self.rotate_horizontal) * self.sensitivity * dt;
        camera.pitch += Rad(-self.rotate_vertical) * self.sensitivity * dt;

        // If process_mouse isn't called every frame, these values
        // will not get set to zero, and the camera will rotate
        // when moving in a non-cardinal direction.
        self.rotate_horizontal = 0.0;
        self.rotate_vertical = 0.0;

        // Keep the camera's angle from going too high/low.
        if camera.pitch < -Rad(SAFE_FRAC_PI_2) {
            camera.pitch = -Rad(SAFE_FRAC_PI_2);
        } else if camera.pitch > Rad(SAFE_FRAC_PI_2) {
            camera.pitch = Rad(SAFE_FRAC_PI_2);
        }
    }
}
```

## Limpiando `lib.rs`

Lo primero es lo primero, necesitamos eliminar `Camera` y `CameraController`, así como el `OPENGL_TO_WGPU_MATRIX` adicional de `lib.rs`. Una vez que hayas hecho eso, importa `camera.rs`.

```rust
mod model;
mod texture;
mod camera; // NEW!
```

Necesitamos actualizar `update_view_proj` para usar nuestra nueva `Camera` y `Projection`.

```rust

impl CameraUniform {
    // ...

    // UPDATED!
    fn update_view_proj(&mut self, camera: &camera::Camera, projection: &camera::Projection) {
        self.view_position = camera.position.to_homogeneous().into();
        self.view_proj = (projection.calc_matrix() * camera.calc_matrix()).into();
    }
}
```

Necesitamos cambiar nuestro `State` para usar nuestra `Camera`, `CameraProjection` y `Projection` también. También añadiremos un campo `mouse_pressed` para almacenar si el ratón fue presionado.

```rust
pub struct State {
    // ...
    camera: camera::Camera, // UPDATED!
    projection: camera::Projection, // NEW!
    camera_controller: camera::CameraController, // UPDATED!
    // ...
    // NEW!
    mouse_pressed: bool,
}
```

Necesitarás importar `winit::dpi::PhysicalPosition` si aún no lo has hecho.

También necesitamos actualizar `new()`.

```rust
impl State {
    async fn new(window: Arc<Window>) -> anyhow::Result<State> {
        // ...

        // UPDATED!
        let camera = camera::Camera::new((0.0, 5.0, 10.0), cgmath::Deg(-90.0), cgmath::Deg(-20.0));
        let projection = camera::Projection::new(config.width, config.height, cgmath::Deg(45.0), 0.1, 100.0);
        let camera_controller = camera::CameraController::new(4.0, 0.4);

        // ...

        camera_uniform.update_view_proj(&camera, &projection); // UPDATED!

        // ...

        Self {
            // ...
            camera,
            projection, // NEW!
            camera_controller,
            // ...
            mouse_pressed: false, // NEW!
        }
    }
}
```

También necesitamos cambiar nuestra `projection` en `resize`.

```rust
fn resize(&mut self, width: u32, height: u32) {
    // UPDATED!
    self.projection.resize(width, height);
    // ...
}
```

`input()` también necesitará ser actualizado. Hasta este punto, hemos estado usando `WindowEvent`s para nuestros controles de cámara. Aunque funciona, no es la mejor solución. La [documentación de winit](https://docs.rs/winit/0.24.0/winit/event/enum.WindowEvent.html?search=#variant.CursorMoved) nos informa que el SO a menudo transformará los datos del evento `CursorMoved` para permitir efectos como la aceleración del cursor.

Ahora, para arreglarlo, podríamos cambiar la función `input()` para procesar `DeviceEvent` en lugar de `WindowEvent`, pero los eventos de teclado y botones no se emiten como `DeviceEvent`s en MacOS y WASM. En su lugar, simplemente eliminaremos la verificación `CursorMoved` en `input()` y una llamada manual a `camera_controller.process_mouse()` en la función `run()`.

```rust
// UPDATED!
fn input(&mut self, event: &WindowEvent) -> bool {
    match event {
        WindowEvent::KeyboardInput {
            event:
                KeyEvent {
                    physical_key: PhysicalKey::Code(key),
                    state,
                    ..
                },
            ..
        } => self.camera_controller.process_keyboard(*key, *state),
        WindowEvent::MouseWheel { delta, .. } => {
            self.camera_controller.process_scroll(delta);
            true
        }
        WindowEvent::MouseInput {
            button: MouseButton::Left,
            state,
            ..
        } => {
            self.mouse_pressed = *state == ElementState::Pressed;
            true
        }
        _ => false,
    }
}
```

Aquí están los cambios a `run()`:

```rust
fn main() {
    // ...
    event_loop.run(move |event, control_flow| {
        match event {
            // ...
            // NEW!
            Event::DeviceEvent {
                event: DeviceEvent::MouseMotion{ delta, },
                .. // We're not using device_id currently
            } => if state.mouse_pressed {
                state.camera_controller.process_mouse(delta.0, delta.1)
            }
            // UPDATED!
            Event::WindowEvent {
                ref event,
                window_id,
            } if window_id == state.window().id() && !state.input(event) => {
                match event {
                    #[cfg(not(target_arch="wasm32"))]
                    WindowEvent::CloseRequested
                    | WindowEvent::KeyboardInput {
                        event:
                            KeyEvent {
                                state: ElementState::Pressed,
                                physical_key: PhysicalKey::Code(KeyCode::Escape),
                                ..
                            },
                        ..
                    } => control_flow.exit(),
                    WindowEvent::Resized(physical_size) => {
                        state.resize(*physical_size);
                    }
                    WindowEvent::ScaleFactorChanged { new_inner_size, .. } => {
                        state.resize(**new_inner_size);
                    }
                    _ => {}
                }
            }
            // ...
        }
    });
}
```

La función `update` requiere un poco más de explicación. La función `update_camera` en el `CameraController` tiene un parámetro `dt: Duration`, que es el tiempo delta o tiempo entre fotogramas. Esto es para ayudar a suavizar el movimiento de la cámara para que no esté bloqueado por la velocidad de fotogramas. Actualmente, no estamos calculando `dt`, así que decidí pasarlo a `update` como parámetro.

```rust
fn update(&mut self, dt: instant::Duration) {
    // UPDATED!
    self.camera_controller.update_camera(&mut self.camera, dt);
    self.camera_uniform.update_view_proj(&self.camera, &self.projection);

    // ..
}
```

Mientras estamos en eso, también usemos `dt` para la rotación de la luz.

```rust
self.light_uniform.position =
    (cgmath::Quaternion::from_axis_angle((0.0, 1.0, 0.0).into(), cgmath::Deg(60.0 * dt.as_secs_f32()))
    * old_position).into(); // UPDATED!
```

Aún necesitamos calcular `dt`. Hagamos eso en la función `main`.

```rust
fn main() {
    // ...
    let mut state = State::new(&window).await;
    let mut last_render_time = instant::Instant::now();  // NEW!
    event_loop.run(move |event, control_flow| {
        match event {
            // ...
            // UPDATED!
            Event::RedrawRequested(window_id) if window_id == state.window().id() => {
                let now = instant::Instant::now();
                let dt = now - last_render_time;
                last_render_time = now;
                state.update(dt);
                // ...
            }
            _ => {}
        }
    });
}
```

Con eso, deberíamos poder mover nuestra cámara a donde queramos.

![./screenshot.png](./screenshot.png)

## Demostración

<WasmExample example="tutorial12_camera"></WasmExample>

<AutoGithubLink/>
