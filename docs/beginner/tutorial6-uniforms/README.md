# Búferes uniformes y una cámara 3D

Aunque todo nuestro trabajo anterior parecía estar en 2D, en realidad hemos estado trabajando en 3D todo el tiempo! Por eso nuestra estructura `Vertex` tiene `position` como un array de 3 floats en lugar de solo 2. No podemos realmente ver la naturaleza 3D de nuestra escena porque estamos viendo las cosas de frente. Vamos a cambiar nuestro punto de vista creando una `Camera`.

## Una cámara de perspectiva

Este tutorial es más sobre aprender a usar wgpu que sobre álgebra lineal, así que voy a pasar por alto mucha de la matemática involucrada. Hay mucho material de lectura en línea si estás interesado en lo que sucede bajo el capó. Vamos a usar [cgmath](https://docs.rs/cgmath) para manejar toda la matemática por nosotros. Añade lo siguiente a tu `Cargo.toml`.

```toml
[dependencies]
# other deps...
cgmath = "0.18"
```

Ahora que tenemos una biblioteca de matemática, ¡usémosla! Crea una estructura `Camera` arriba de la estructura `State`.

```rust
struct Camera {
    eye: cgmath::Point3<f32>,
    target: cgmath::Point3<f32>,
    up: cgmath::Vector3<f32>,
    aspect: f32,
    fovy: f32,
    znear: f32,
    zfar: f32,
}

impl Camera {
    fn build_view_projection_matrix(&self) -> cgmath::Matrix4<f32> {
        // 1.
        let view = cgmath::Matrix4::look_at_rh(self.eye, self.target, self.up);
        // 2.
        let proj = cgmath::perspective(cgmath::Deg(self.fovy), self.aspect, self.znear, self.zfar);

        // 3.
        return OPENGL_TO_WGPU_MATRIX * proj * view;
    }
}
```

El `build_view_projection_matrix` es donde ocurre la magia.
1. La matriz `view` mueve el mundo para que esté en la posición y rotación de la cámara. Es esencialmente una inversa de lo que sería la matriz de transformación de la cámara.
2. La matriz `proj` distorsiona la escena para dar el efecto de profundidad. Sin esto, los objetos cerca serían del mismo tamaño que los objetos lejanos.
3. El sistema de coordenadas en Wgpu se basa en los sistemas de coordenadas de DirectX y Metal. Eso significa que en [coordenadas de dispositivo normalizadas](https://github.com/gfx-rs/gfx/tree/master/src/backend/dx12#normalized-coordinates), el eje x y el eje y están en el rango de -1.0 a +1.0, y el eje z es de 0.0 a +1.0. El crate `cgmath` (así como la mayoría de los crates de matemática de juegos) está construido para el sistema de coordenadas de OpenGL. Esta matriz escalará y trasladará nuestra escena del sistema de coordenadas de OpenGL al de WGPU. Lo definiremos como sigue.

```rust
#[rustfmt::skip]
pub const OPENGL_TO_WGPU_MATRIX: cgmath::Matrix4<f32> = cgmath::Matrix4::from_cols(
    cgmath::Vector4::new(1.0, 0.0, 0.0, 0.0),
    cgmath::Vector4::new(0.0, 1.0, 0.0, 0.0),
    cgmath::Vector4::new(0.0, 0.0, 0.5, 0.0),
    cgmath::Vector4::new(0.0, 0.0, 0.5, 1.0),
);
```

* Nota: No necesitamos explícitamente la `OPENGL_TO_WGPU_MATRIX`, pero los modelos centrados en (0, 0, 0) estarán a mitad de camino dentro del área de recorte. Esto solo es un problema si no estás usando una matriz de cámara.

Ahora vamos a añadir un campo `camera` a `State`.

```rust
pub struct State {
    // ...
    camera: Camera,
    // ...
}

async fn new(window: Window) -> Self {
    // let diffuse_bind_group ...

    let camera = Camera {
        // posiciona la cámara 1 unidad hacia arriba y 2 unidades hacia atrás
        // +z está fuera de la pantalla
        eye: (0.0, 1.0, 2.0).into(),
        // haz que mire al origen
        target: (0.0, 0.0, 0.0).into(),
        // cuál es la dirección "arriba"
        up: cgmath::Vector3::unit_y(),
        aspect: config.width as f32 / config.height as f32,
        fovy: 45.0,
        znear: 0.1,
        zfar: 100.0,
    };

    Self {
        // ...
        camera,
        // ...
    }
}
```

Ahora que tenemos nuestra cámara, y puede crear una matriz de proyección de vista, necesitamos un lugar para ponerla. También necesitamos una forma de introducirla en nuestros shaders.

## El búfer uniforme

Hasta este punto, hemos usado `Buffer`s para almacenar nuestros datos de vértices e índices, e incluso para cargar nuestras texturas. Vamos a usarlos de nuevo para crear lo que se conoce como un búfer uniforme. Un uniforme es un blob de datos disponible para cada invocación de un conjunto de shaders. Técnicamente, ya hemos usado uniformes para nuestra textura y sampler. Vamos a usarlos de nuevo para almacenar nuestra matriz de proyección de vista. Para empezar, vamos a crear una estructura para mantener nuestro uniforme.

```rust
// Necesitamos esto para que Rust almacene nuestros datos correctamente para los shaders
#[repr(C)]
// Esto es para que podamos almacenar esto en un búfer
#[derive(Debug, Copy, Clone, bytemuck::Pod, bytemuck::Zeroable)]
struct CameraUniform {
    // No podemos usar cgmath con bytemuck directamente, así que tendremos
    // que convertir la Matrix4 en un array f32 de 4x4
    view_proj: [[f32; 4]; 4],
}

impl CameraUniform {
    fn new() -> Self {
        use cgmath::SquareMatrix;
        Self {
            view_proj: cgmath::Matrix4::identity().into(),
        }
    }

    fn update_view_proj(&mut self, camera: &Camera) {
        self.view_proj = camera.build_view_projection_matrix().into();
    }
}
```

Ahora que tenemos nuestros datos estructurados, vamos a hacer nuestro `camera_buffer`.

```rust
// en new() después de crear `camera`

let mut camera_uniform = CameraUniform::new();
camera_uniform.update_view_proj(&camera);

let camera_buffer = device.create_buffer_init(
    &wgpu::util::BufferInitDescriptor {
        label: Some("Camera Buffer"),
        contents: bytemuck::cast_slice(&[camera_uniform]),
        usage: wgpu::BufferUsages::UNIFORM | wgpu::BufferUsages::COPY_DST,
    }
);
```

## Búferes uniformes y grupos de enlace

¡Bien! Ahora que tenemos un búfer uniforme, ¿qué hacemos con él? La respuesta es que creamos un grupo de enlace para él. Primero, tenemos que crear el diseño del grupo de enlace.

```rust
let camera_bind_group_layout = device.create_bind_group_layout(&wgpu::BindGroupLayoutDescriptor {
    entries: &[
        wgpu::BindGroupLayoutEntry {
            binding: 0,
            visibility: wgpu::ShaderStages::VERTEX,
            ty: wgpu::BindingType::Buffer {
                ty: wgpu::BufferBindingType::Uniform,
                has_dynamic_offset: false,
                min_binding_size: None,
            },
            count: None,
        }
    ],
    label: Some("camera_bind_group_layout"),
});
```

Algunas cosas a tener en cuenta:

1. Configuramos `visibility` a `ShaderStages::VERTEX` porque realmente solo necesitamos información de cámara en el vertex shader,
    ya que es lo que usaremos para manipular nuestros vértices.
2. `has_dynamic_offset` significa que la ubicación de los datos en el búfer puede cambiar. Este será el caso si
    almacenas múltiples conjuntos de datos de tamaño variable en un único búfer. Si lo estableces en true, tendrás que
    suministrar los desplazamientos más tarde.
3. `min_binding_size` especifica el tamaño más pequeño que puede tener el búfer. No tienes que especificar esto, así que
    lo dejamos como `None`. Si quieres saber más, puedes revisar [la documentación](https://docs.rs/wgpu/latest/wgpu/enum.BindingType.html#variant.Buffer.field.min_binding_size).

Ahora, podemos crear el grupo de enlace real.

```rust
let camera_bind_group = device.create_bind_group(&wgpu::BindGroupDescriptor {
    layout: &camera_bind_group_layout,
    entries: &[
        wgpu::BindGroupEntry {
            binding: 0,
            resource: camera_buffer.as_entire_binding(),
        }
    ],
    label: Some("camera_bind_group"),
});
```

Como con nuestra textura, necesitamos registrar nuestro `camera_bind_group_layout` con el pipeline de renderizado.

```rust
let render_pipeline_layout = device.create_pipeline_layout(
    &wgpu::PipelineLayoutDescriptor {
        label: Some("Render Pipeline Layout"),
        bind_group_layouts: &[
            &texture_bind_group_layout,
            &camera_bind_group_layout,
        ],
        push_constant_ranges: &[],
    }
);
```

Ahora necesitamos añadir `camera_buffer` y `camera_bind_group` a `State`

```rust
pub struct State {
    // ...
    camera: Camera,
    camera_uniform: CameraUniform,
    camera_buffer: wgpu::Buffer,
    camera_bind_group: wgpu::BindGroup,
}

async fn new(window: Window) -> Self {
    // ...
    Self {
        // ...
        camera,
        camera_uniform,
        camera_buffer,
        camera_bind_group,
    }
}
```

La última cosa que necesitamos hacer antes de entrar en shaders es usar el grupo de enlace en `render()`.

```rust
render_pass.set_pipeline(&self.render_pipeline);
render_pass.set_bind_group(0, &self.diffuse_bind_group, &[]);
// ¡NUEVO!
render_pass.set_bind_group(1, &self.camera_bind_group, &[]);
render_pass.set_vertex_buffer(0, self.vertex_buffer.slice(..));
render_pass.set_index_buffer(self.index_buffer.slice(..), wgpu::IndexFormat::Uint16);

render_pass.draw_indexed(0..self.num_indices, 0, 0..1);
```

## Usando el uniforme en el vertex shader

Modifica el vertex shader para incluir lo siguiente.

```wgsl
// Vertex shader
struct CameraUniform {
    view_proj: mat4x4<f32>,
};
@group(1) @binding(0) // 1.
var<uniform> camera: CameraUniform;

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
    out.clip_position = camera.view_proj * vec4<f32>(model.position, 1.0); // 2.
    return out;
}
```

1. Porque hemos creado un nuevo grupo de enlace, necesitamos especificar cuál estamos usando en el shader. El número está determinado por nuestro `render_pipeline_layout`. El `texture_bind_group_layout` se enumera primero, por lo que es `group(0)`, y `camera_bind_group` es el segundo, así que es `group(1)`.
2. El orden de multiplicación es importante cuando se trata de matrices. El vector va a la derecha, y las matrices van a la izquierda en orden de importancia.

## Un controlador para nuestra cámara

Si ejecutas el código ahora mismo, deberías obtener algo como esto.

![./static-tree.png](./static-tree.png)

La forma ahora está menos estirada, pero sigue siendo bastante estática. Puedes experimentar moviendo la posición de la cámara, pero la mayoría de las cámaras en los juegos se mueven. Como este tutorial es sobre usar wgpu y no sobre cómo procesar la entrada del usuario, simplemente voy a publicar el código de `CameraController` a continuación.

```rust
struct CameraController {
    speed: f32,
    is_forward_pressed: bool,
    is_backward_pressed: bool,
    is_left_pressed: bool,
    is_right_pressed: bool,
}

impl CameraController {
    fn new(speed: f32) -> Self {
        Self {
            speed,
            is_forward_pressed: false,
            is_backward_pressed: false,
            is_left_pressed: false,
            is_right_pressed: false,
        }
    }

    fn handle_key(&mut self, code: KeyCode, is_pressed: bool) -> bool {
        match code {
            KeyCode::KeyW | KeyCode::ArrowUp => {
                self.is_forward_pressed = is_pressed;
                true
            }
            KeyCode::KeyA | KeyCode::ArrowLeft => {
                self.is_left_pressed = is_pressed;
                true
            }
            KeyCode::KeyS | KeyCode::ArrowDown => {
                self.is_backward_pressed = is_pressed;
                true
            }
            KeyCode::KeyD | KeyCode::ArrowRight => {
                self.is_right_pressed = is_pressed;
                true
            }
            _ => false,
        }
    }

    fn update_camera(&self, camera: &mut Camera) {
        use cgmath::InnerSpace;
        let forward = camera.target - camera.eye;
        let forward_norm = forward.normalize();
        let forward_mag = forward.magnitude();

        // Previene parpadeos cuando la cámara se acerca demasiado al
        // centro de la escena.
        if self.is_forward_pressed && forward_mag > self.speed {
            camera.eye += forward_norm * self.speed;
        }
        if self.is_backward_pressed {
            camera.eye -= forward_norm * self.speed;
        }

        let right = forward_norm.cross(camera.up);

        // Recalcula el radio en caso de que se presione adelante/atrás.
        let forward = camera.target - camera.eye;
        let forward_mag = forward.magnitude();

        if self.is_right_pressed {
            // Reescala la distancia entre el objetivo y el ojo para que
            // no cambie. El ojo, por lo tanto, sigue siendo
            // situado en el círculo hecho por el objetivo y el ojo.
            camera.eye = camera.target - (forward + right * self.speed).normalize() * forward_mag;
        }
        if self.is_left_pressed {
            camera.eye = camera.target - (forward - right * self.speed).normalize() * forward_mag;
        }
    }
}
```

Este código no es perfecto. La cámara se mueve lentamente hacia atrás cuando la rotas. Funciona para nuestros propósitos, sin embargo. ¡Siéntete libre de mejorarlo!

Todavía necesitamos conectar esto a nuestro código existente para que haga algo. Añade el controlador a `State` y créalo en `new()`.

```rust
pub struct State {
    // ...
    camera: Camera,
    // NEW!
    camera_controller: CameraController,
    // ...
}
// ...
impl State {
    async fn new(window: Arc<Window>) -> anyhow::Result<State> {
        // ...
        let camera_controller = CameraController::new(0.2);
        // ...

        Self {
            // ...
            camera_controller,
            // ...
        }
    }
}
```

Actualizaremos el `camera_controller` en la función `handle_key`.

```rust
    fn handle_key(&mut self, event_loop: &ActiveEventLoop, code: KeyCode, is_pressed: bool) {
        if code == KeyCode::Escape && is_pressed {
            event_loop.exit();
        } else {
            self.camera_controller.handle_key(code, is_pressed);
        }
    }
```

Hasta este punto, el controlador de cámara no está haciendo nada. Los valores en nuestro búfer uniforme necesitan ser actualizados. Hay algunos métodos principales para hacer eso.

1. Podemos crear un búfer separado y copiar su contenido a nuestro `camera_buffer`. El nuevo búfer se conoce como un búfer de preparación. Este método es generalmente cómo se hace, ya que permite que el contenido del búfer principal (en este caso, `camera_buffer`) sea accesible solo por la GPU. La GPU puede hacer algunas optimizaciones de velocidad, que no podría si pudiéramos acceder al búfer a través de la CPU.
2. Podemos llamar a uno de los métodos de mapeo `map_read_async` y `map_write_async` en el búfer mismo. Estos nos permiten acceder directamente al contenido de un búfer pero requieren que tratemos con el aspecto `async` de estos métodos. Esto también requiere que nuestro búfer use `BufferUsages::MAP_READ` y/o `BufferUsages::MAP_WRITE`. No hablaremos de ello aquí, pero consulta el tutorial [Wgpu sin ventana](../../showcase/windowless) si quieres saber más.
3. Podemos usar `write_buffer` en `queue`.

Vamos a usar la opción número 3.

```rust
fn update(&mut self) {
    self.camera_controller.update_camera(&mut self.camera);
    self.camera_uniform.update_view_proj(&self.camera);
    self.queue.write_buffer(&self.camera_buffer, 0, bytemuck::cast_slice(&[self.camera_uniform]));
}
```

Eso es todo lo que necesitamos hacer. Si ejecutas el código ahora, deberías ver un pentágono con nuestra textura de árbol que puedes rotar alrededor y hacer zoom con las teclas wasd/flechas.

## Demo

<WasmExample example="tutorial6_uniforms"></WasmExample>

<AutoGithubLink/>

## Desafío

Haz que nuestro modelo rote por su cuenta independientemente de la cámara. *Pista: necesitarás otra matriz para esto.*
