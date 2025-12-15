# Trabajando con Luces

Aunque podemos notar que nuestra escena es 3D gracias a nuestra cámara, aún se ve muy plana. Esto se debe a que nuestro modelo mantiene el mismo color independientemente de su orientación. Si queremos cambiar eso, necesitamos agregar iluminación a nuestra escena.

En el mundo real, una fuente de luz emite fotones que rebotan hasta entrar en nuestros ojos. El color que vemos es el color original de la luz menos la energía que perdió mientras rebotaba.

En el mundo de los gráficos por computadora, modelar fotones individuales sería hilarantemente costoso computacionalmente. Una bombilla de 100 vatios emite aproximadamente 3.27 x 10^20 fotones *por segundo*. ¡Solo imagina eso para el sol! Para evitar esto, vamos a usar matemáticas para hacer trampa.

Vamos a discutir algunas opciones.

## Ray/Path Tracing

Este es un tema *avanzado*, y no lo cubriremos a profundidad aquí. Es el modelo más cercano a la forma en que la luz realmente funciona, así que sentí que tenía que mencionarlo. Consulta el [tutorial de ray tracing](../../todo/) si quieres aprender más.

## El Modelo Blinn-Phong

El ray/path tracing es a menudo demasiado costoso computacionalmente para la mayoría de las aplicaciones en tiempo real (aunque eso está comenzando a cambiar), por lo que se usa a menudo un método más eficiente, aunque menos preciso, basado en el [modelo de reflexión de Phong](https://en.wikipedia.org/wiki/Phong_shading). Divide el cálculo de iluminación en tres partes: iluminación ambiental, iluminación difusa e iluminación especular. Vamos a aprender el [modelo Blinn-Phong](https://en.wikipedia.org/wiki/Blinn%E2%80%93Phong_reflection_model), que hace un poco de trampa en el cálculo especular para acelerar las cosas.

Antes de entrar en eso, sin embargo, necesitamos agregar una luz a nuestra escena.

```rust
// lib.rs
#[repr(C)]
#[derive(Debug, Copy, Clone, bytemuck::Pod, bytemuck::Zeroable)]
struct LightUniform {
    position: [f32; 3],
    // Due to uniforms requiring 16 byte (4 float) spacing, we need to use a padding field here
    _padding: u32,
    color: [f32; 3],
    // Due to uniforms requiring 16 byte (4 float) spacing, we need to use a padding field here
    _padding2: u32,
}
```

Nuestro `LightUniform` representa un punto coloreado en el espacio. Solo vamos a usar luz blanca pura, pero es bueno permitir diferentes colores de luz.


<div class="note">

La regla general para la alineación con estructuras WGSL es que las alineaciones de campos son siempre potencias de 2. Por ejemplo, un `vec3` puede tener solo tres campos flotantes, dándole un tamaño de 12. La alineación se elevará a la siguiente potencia de 2 siendo 16. Esto significa que debes tener más cuidado con cómo diseñas tu estructura en Rust.

Algunos desarrolladores optan por usar `vec4` en lugar de `vec3` para evitar problemas de alineación. Puedes aprender más sobre las reglas de alineación en la [especificación WGSL](https://www.w3.org/TR/WGSL/#alignment-and-size)

</div>

Vamos a crear otro búfer para almacenar nuestra luz.

```rust
let light_uniform = LightUniform {
    position: [2.0, 2.0, 2.0],
    _padding: 0,
    color: [1.0, 1.0, 1.0],
    _padding2: 0,
};

 // We'll want to update our lights position, so we use COPY_DST
let light_buffer = device.create_buffer_init(
    &wgpu::util::BufferInitDescriptor {
        label: Some("Light VB"),
        contents: bytemuck::cast_slice(&[light_uniform]),
        usage: wgpu::BufferUsages::UNIFORM | wgpu::BufferUsages::COPY_DST,
    }
);
```


No olvides agregar `light_uniform` y `light_buffer` a `State`. Después de eso, necesitamos crear un diseño de grupo de vinculación y un grupo de vinculación para nuestra luz.

```rust
let light_bind_group_layout =
    device.create_bind_group_layout(&wgpu::BindGroupLayoutDescriptor {
        entries: &[wgpu::BindGroupLayoutEntry {
            binding: 0,
            visibility: wgpu::ShaderStages::VERTEX | wgpu::ShaderStages::FRAGMENT,
            ty: wgpu::BindingType::Buffer {
                ty: wgpu::BufferBindingType::Uniform,
                has_dynamic_offset: false,
                min_binding_size: None,
            },
            count: None,
        }],
        label: None,
    });

let light_bind_group = device.create_bind_group(&wgpu::BindGroupDescriptor {
    layout: &light_bind_group_layout,
    entries: &[wgpu::BindGroupEntry {
        binding: 0,
        resource: light_buffer.as_entire_binding(),
    }],
    label: None,
});
```

Agrega esos a `State` y también actualiza el `render_pipeline_layout`.

```rust
let render_pipeline_layout = device.create_pipeline_layout(&wgpu::PipelineLayoutDescriptor {
    bind_group_layouts: &[
        &texture_bind_group_layout, 
        &camera_bind_group_layout,
        &light_bind_group_layout,
    ],
});
```

Vamos a actualizar también la posición de la luz en el método `update()` para ver cómo se ven nuestros objetos desde diferentes ángulos.

```rust
// Update the light
let old_position: cgmath::Vector3<_> = self.light_uniform.position.into();
self.light_uniform.position =
    (cgmath::Quaternion::from_axis_angle((0.0, 1.0, 0.0).into(), cgmath::Deg(1.0))
        * old_position)
        .into();
self.queue.write_buffer(&self.light_buffer, 0, bytemuck::cast_slice(&[self.light_uniform]));
```

Esto hará que la luz gire alrededor del origen un grado cada fotograma.

## Ver la luz

Para propósitos de depuración, sería bueno si pudiéramos ver dónde está la luz para asegurar que la escena se vea correcta. Podríamos adaptar nuestro pipeline de renderizado existente para dibujar la luz, pero probablemente se interpondría. En su lugar, vamos a extraer nuestro código de creación del pipeline de renderizado en una nueva función llamada `create_render_pipeline()`.


```rust
fn create_render_pipeline(
    device: &wgpu::Device,
    layout: &wgpu::PipelineLayout,
    color_format: wgpu::TextureFormat,
    depth_format: Option<wgpu::TextureFormat>,
    vertex_layouts: &[wgpu::VertexBufferLayout],
    shader: wgpu::ShaderModuleDescriptor,
) -> wgpu::RenderPipeline {
    let shader = device.create_shader_module(shader);

    device.create_render_pipeline(&wgpu::RenderPipelineDescriptor {
        label: Some("Render Pipeline"),
        layout: Some(layout),
        vertex: wgpu::VertexState {
            module: &shader,
            entry_point: Some("vs_main"),
            buffers: vertex_layouts,
            compilation_options: Default::default(),
        },
        fragment: Some(wgpu::FragmentState {
            module: &shader,
            entry_point: Some("fs_main"),
            targets: &[Some(wgpu::ColorTargetState {
                format: color_format,
                blend: Some(wgpu::BlendState {
                    alpha: wgpu::BlendComponent::REPLACE,
                    color: wgpu::BlendComponent::REPLACE,
                }),
                write_mask: wgpu::ColorWrites::ALL,
            })],
            compilation_options: Default::default(),
        }),
        primitive: wgpu::PrimitiveState {
            topology: wgpu::PrimitiveTopology::TriangleList,
            strip_index_format: None,
            front_face: wgpu::FrontFace::Ccw,
            cull_mode: Some(wgpu::Face::Back),
            // Setting this to anything other than Fill requires Features::NON_FILL_POLYGON_MODE
            polygon_mode: wgpu::PolygonMode::Fill,
            // Requires Features::DEPTH_CLIP_CONTROL
            unclipped_depth: false,
            // Requires Features::CONSERVATIVE_RASTERIZATION
            conservative: false,
        },
        depth_stencil: depth_format.map(|format| wgpu::DepthStencilState {
            format,
            depth_write_enabled: true,
            depth_compare: wgpu::CompareFunction::Less,
            stencil: wgpu::StencilState::default(),
            bias: wgpu::DepthBiasState::default(),
        }),
        multisample: wgpu::MultisampleState {
            count: 1,
            mask: !0,
            alpha_to_coverage_enabled: false,
        },
        multiview: None,
    })
}
```

También necesitamos cambiar `State::new()` para usar esta función.

```rust
let render_pipeline = {
    let shader = wgpu::ShaderModuleDescriptor {
        label: Some("Normal Shader"),
        source: wgpu::ShaderSource::Wgsl(include_str!("shader.wgsl").into()),
    };
    create_render_pipeline(
        &device,
        &render_pipeline_layout,
        config.format,
        Some(texture::Texture::DEPTH_FORMAT),
        &[model::ModelVertex::desc(), InstanceRaw::desc()],
        shader,
    )
};
```

Vamos a necesitar modificar `model::DrawModel` para usar nuestro `light_bind_group`.

```rust
// model.rs
pub trait DrawModel<'a> {
    fn draw_mesh(
        &mut self,
        mesh: &'a Mesh,
        material: &'a Material,
        camera_bind_group: &'a wgpu::BindGroup,
        light_bind_group: &'a wgpu::BindGroup,
    );
    fn draw_mesh_instanced(
        &mut self,
        mesh: &'a Mesh,
        material: &'a Material,
        instances: Range<u32>,
        camera_bind_group: &'a wgpu::BindGroup,
        light_bind_group: &'a wgpu::BindGroup,
    );

    fn draw_model(
        &mut self,
        model: &'a Model,
        camera_bind_group: &'a wgpu::BindGroup,
        light_bind_group: &'a wgpu::BindGroup,
    );
    fn draw_model_instanced(
        &mut self,
        model: &'a Model,
        instances: Range<u32>,
        camera_bind_group: &'a wgpu::BindGroup,
        light_bind_group: &'a wgpu::BindGroup,
    );
}

impl<'a, 'b> DrawModel<'b> for wgpu::RenderPass<'a>
where
    'b: 'a,
{
    fn draw_mesh(
        &mut self,
        mesh: &'b Mesh,
        material: &'b Material,
        camera_bind_group: &'b wgpu::BindGroup,
        light_bind_group: &'b wgpu::BindGroup,
    ) {
        self.draw_mesh_instanced(mesh, material, 0..1, camera_bind_group, light_bind_group);
    }

    fn draw_mesh_instanced(
        &mut self,
        mesh: &'b Mesh,
        material: &'b Material,
        instances: Range<u32>,
        camera_bind_group: &'b wgpu::BindGroup,
        light_bind_group: &'b wgpu::BindGroup,
    ) {
        self.set_vertex_buffer(0, mesh.vertex_buffer.slice(..));
        self.set_index_buffer(mesh.index_buffer.slice(..), wgpu::IndexFormat::Uint32);
        self.set_bind_group(0, &material.bind_group, &[]);
        self.set_bind_group(1, camera_bind_group, &[]);
        self.set_bind_group(2, light_bind_group, &[]);
        self.draw_indexed(0..mesh.num_elements, 0, instances);
    }

    fn draw_model(
        &mut self,
        model: &'b Model,
        camera_bind_group: &'b wgpu::BindGroup,
        light_bind_group: &'b wgpu::BindGroup,
    ) {
        self.draw_model_instanced(model, 0..1, camera_bind_group, light_bind_group);
    }

    fn draw_model_instanced(
        &mut self,
        model: &'b Model,
        instances: Range<u32>,
        camera_bind_group: &'b wgpu::BindGroup,
        light_bind_group: &'b wgpu::BindGroup,
    ) {
        for mesh in &model.meshes {
            let material = &model.materials[mesh.material];
            self.draw_mesh_instanced(mesh, material, instances.clone(), camera_bind_group, light_bind_group);
        }
    }
}
```

Con eso hecho, podemos crear otro pipeline de renderizado para nuestra luz.

```rust
// lib.rs
let light_render_pipeline = {
    let layout = device.create_pipeline_layout(&wgpu::PipelineLayoutDescriptor {
        label: Some("Light Pipeline Layout"),
        bind_group_layouts: &[&camera_bind_group_layout, &light_bind_group_layout],
        push_constant_ranges: &[],
    });
    let shader = wgpu::ShaderModuleDescriptor {
        label: Some("Light Shader"),
        source: wgpu::ShaderSource::Wgsl(include_str!("light.wgsl").into()),
    };
    create_render_pipeline(
        &device,
        &layout,
        config.format,
        Some(texture::Texture::DEPTH_FORMAT),
        &[model::ModelVertex::desc()],
        shader,
    )
};
```

Opté por crear un diseño separado para el `light_render_pipeline`, ya que no necesita todos los recursos que el `render_pipeline` regular necesita (principalmente solo las texturas).

Con eso en su lugar, necesitamos escribir los sombreadores reales.

```wgsl
// light.wgsl
// Vertex shader

struct Camera {
    view_proj: mat4x4<f32>,
}
@group(0) @binding(0)
var<uniform> camera: Camera;

struct Light {
    position: vec3<f32>,
    color: vec3<f32>,
}
@group(1) @binding(0)
var<uniform> light: Light;

struct VertexInput {
    @location(0) position: vec3<f32>,
};

struct VertexOutput {
    @builtin(position) clip_position: vec4<f32>,
    @location(0) color: vec3<f32>,
};

@vertex
fn vs_main(
    model: VertexInput,
) -> VertexOutput {
    let scale = 0.25;
    var out: VertexOutput;
    out.clip_position = camera.view_proj * vec4<f32>(model.position * scale + light.position, 1.0);
    out.color = light.color;
    return out;
}

// Fragment shader

@fragment
fn fs_main(in: VertexOutput) -> @location(0) vec4<f32> {
    return vec4<f32>(in.color, 1.0);
}
```

Ahora, podríamos implementar manualmente el código de dibujo para la luz en `render()`, pero para mantener el patrón que desarrollamos, vamos a crear un nuevo trait llamado `DrawLight`.

```rust
// model.rs
pub trait DrawLight<'a> {
    fn draw_light_mesh(
        &mut self,
        mesh: &'a Mesh,
        camera_bind_group: &'a wgpu::BindGroup,
        light_bind_group: &'a wgpu::BindGroup,
    );
    fn draw_light_mesh_instanced(
        &mut self,
        mesh: &'a Mesh,
        instances: Range<u32>,
        camera_bind_group: &'a wgpu::BindGroup,
        light_bind_group: &'a wgpu::BindGroup,
    );

    fn draw_light_model(
        &mut self,
        model: &'a Model,
        camera_bind_group: &'a wgpu::BindGroup,
        light_bind_group: &'a wgpu::BindGroup,
    );
    fn draw_light_model_instanced(
        &mut self,
        model: &'a Model,
        instances: Range<u32>,
        camera_bind_group: &'a wgpu::BindGroup,
        light_bind_group: &'a wgpu::BindGroup,
    );
}

impl<'a, 'b> DrawLight<'b> for wgpu::RenderPass<'a>
where
    'b: 'a,
{
    fn draw_light_mesh(
        &mut self,
        mesh: &'b Mesh,
        camera_bind_group: &'b wgpu::BindGroup,
        light_bind_group: &'b wgpu::BindGroup,
    ) {
        self.draw_light_mesh_instanced(mesh, 0..1, camera_bind_group, light_bind_group);
    }

    fn draw_light_mesh_instanced(
        &mut self,
        mesh: &'b Mesh,
        instances: Range<u32>,
        camera_bind_group: &'b wgpu::BindGroup,
        light_bind_group: &'b wgpu::BindGroup,
    ) {
        self.set_vertex_buffer(0, mesh.vertex_buffer.slice(..));
        self.set_index_buffer(mesh.index_buffer.slice(..), wgpu::IndexFormat::Uint32);
        self.set_bind_group(0, camera_bind_group, &[]);
        self.set_bind_group(1, light_bind_group, &[]);
        self.draw_indexed(0..mesh.num_elements, 0, instances);
    }

    fn draw_light_model(
        &mut self,
        model: &'b Model,
        camera_bind_group: &'b wgpu::BindGroup,
        light_bind_group: &'b wgpu::BindGroup,
    ) {
        self.draw_light_model_instanced(model, 0..1, camera_bind_group, light_bind_group);
    }
    fn draw_light_model_instanced(
        &mut self,
        model: &'b Model,
        instances: Range<u32>,
        camera_bind_group: &'b wgpu::BindGroup,
        light_bind_group: &'b wgpu::BindGroup,
    ) {
        for mesh in &model.meshes {
            self.draw_light_mesh_instanced(mesh, instances.clone(), camera_bind_group, light_bind_group);
        }
    }
}
```

Finalmente, queremos agregar renderizado de luz a nuestros pases de renderizado.

```rust
impl State {
    // ...
   fn render(&mut self) -> Result<(), wgpu::SurfaceError> {
        // ...
        render_pass.set_vertex_buffer(1, self.instance_buffer.slice(..));

        use crate::model::DrawLight; // NEW!
        render_pass.set_pipeline(&self.light_render_pipeline); // NEW!
        render_pass.draw_light_model(
            &self.obj_model,
            &self.camera_bind_group,
            &self.light_bind_group,
        ); // NEW!

        render_pass.set_pipeline(&self.render_pipeline);
        render_pass.draw_model_instanced(
            &self.obj_model,
            0..self.instances.len() as u32,
            &self.camera_bind_group,
            &self.light_bind_group, // NEW
        );
}
```

Con todo eso, terminaremos con algo así.

![./light-in-scene.png](./light-in-scene.png)

## Iluminación Ambiental

La luz tiene la tendencia a rebotar antes de entrar en nuestros ojos. Por eso puedes ver en áreas que están en sombra. Modelar esta interacción sería computacionalmente costoso, así que haremos trampa. Definimos un valor de iluminación ambiental para la luz que rebota en otras partes de la escena para iluminar nuestros objetos.

La parte ambiental se basa en el color de la luz y el color del objeto. Ya hemos agregado nuestro `light_bind_group`, así que solo necesitamos usarlo en nuestro sombreador. En `shader.wgsl`, agrega lo siguiente debajo de los uniformes de textura.

```wgsl
struct Light {
    position: vec3<f32>,
    color: vec3<f32>,
}
@group(2) @binding(0)
var<uniform> light: Light;
```

Luego, necesitamos actualizar nuestro código de sombreador principal para calcular y usar el valor de color ambiental.

```wgsl
@fragment
fn fs_main(in: VertexOutput) -> @location(0) vec4<f32> {
    let object_color: vec4<f32> = textureSample(t_diffuse, s_diffuse, in.tex_coords);

    // We don't need (or want) much ambient light, so 0.1 is fine
    let ambient_strength = 0.1;
    let ambient_color = light.color * ambient_strength;

    let result = ambient_color * object_color.xyz;

    return vec4<f32>(result, object_color.a);
}
```

Con eso, deberíamos obtener algo así.

![./ambient_lighting.png](./ambient_lighting.png)

## Iluminación Difusa

¿Recuerdas los vectores normales que se incluían en nuestro modelo? Finalmente vamos a usarlos. Las normales representan la dirección a la que se enfrenta una superficie. Al comparar la normal de un fragmento con un vector que apunta a una fuente de luz, obtenemos un valor de cuán claro/oscuro debe ser ese fragmento. Comparamos los vectores usando el producto punto para obtener el coseno del ángulo entre ellos.

![./normal_diagram.png](./normal_diagram.png)

Si el producto punto de la normal y el vector de luz es 1.0, eso significa que el fragmento actual está directamente alineado con la fuente de luz y recibirá la intensidad completa de la luz. Un valor de 0.0 o inferior significa que la superficie es perpendicular o se enfrenta alejándose de la luz y, por lo tanto, estará oscura.

Vamos a necesitar introducir el vector normal en nuestro `shader.wgsl`.

```wgsl
struct VertexInput {
    @location(0) position: vec3<f32>,
    @location(1) tex_coords: vec2<f32>,
    @location(2) normal: vec3<f32>, // NEW!
};
```

También vamos a querer pasar ese valor, así como la posición del vértice, al sombreador de fragmentos.

```wgsl
struct VertexOutput {
    @builtin(position) clip_position: vec4<f32>,
    @location(0) tex_coords: vec2<f32>,
    @location(1) world_normal: vec3<f32>,
    @location(2) world_position: vec3<f32>,
};
```

Por ahora, pasemos la normal directamente tal como está. Esto es incorrecto, pero lo arreglaremos más tarde.

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
    var out: VertexOutput;
    out.tex_coords = model.tex_coords;
    out.world_normal = model.normal;
    var world_position: vec4<f32> = model_matrix * vec4<f32>(model.position, 1.0);
    out.world_position = world_position.xyz;
    out.clip_position = camera.view_proj * world_position;
    return out;
}
```

Con eso, podemos hacer el cálculo real. Agrega lo siguiente debajo del cálculo `ambient_color` pero arriba del `result`.

```wgsl
let light_dir = normalize(light.position - in.world_position);

let diffuse_strength = max(dot(in.world_normal, light_dir), 0.0);
let diffuse_color = light.color * diffuse_strength;
```

Ahora podemos incluir el `diffuse_color` en el `result`.

```wgsl
let result = (ambient_color + diffuse_color) * object_color.xyz;
```

Con eso, obtenemos algo así.

![./ambient_diffuse_wrong.png](./ambient_diffuse_wrong.png)

## La matriz normal

¿Recuerdas cuando dije que pasar la normal del vértice directamente al sombreador de fragmentos era incorrecto? Vamos a explorar eso removiendo todos los cubos de la escena excepto uno que será rotado 180 grados en el eje y.

```rust
const NUM_INSTANCES_PER_ROW: u32 = 1;

// In the loop, we create the instances in
let rotation = cgmath::Quaternion::from_axis_angle((0.0, 1.0, 0.0).into(), cgmath::Deg(180.0));
```

También quitaremos el `ambient_color` de nuestro `result` de iluminación.

```wgsl
let result = (diffuse_color) * object_color.xyz;
```

Eso debería darnos algo que se vea así.

![./diffuse_wrong.png](./diffuse_wrong.png)

Esto es claramente incorrecto, ya que la luz está iluminando el lado equivocado del cubo. Esto se debe a que no estamos rotando nuestras normales con nuestro objeto, por lo que sin importar la dirección en que se enfrente el objeto, las normales siempre enfrentarán el mismo lado.

![./normal_not_rotated.png](./normal_not_rotated.png)

Necesitamos usar la matriz de modelo para transformar las normales en la dirección correcta. Sin embargo, solo queremos los datos de rotación. Una normal representa una dirección y debe ser un vector unitario en todo el cálculo. Podemos obtener nuestras normales en la dirección correcta usando lo que se llama una matriz normal.

Podríamos calcular la matriz normal en el sombreador de vértices, pero eso implicaría invertir la `model_matrix`, y WGSL en realidad no tiene una función inversa. Tendríamos que codificar la nuestra. Además de eso, calcular la inversa de una matriz es realmente muy costoso, especialmente hacer ese cálculo para cada vértice.

En su lugar, vamos a agregar un campo de matriz `normal` a `InstanceRaw`. En lugar de invertir la matriz de modelo, simplemente usaremos la rotación de la instancia para crear una `Matrix3`.

<div class="note">

Estamos usando `Matrix3` en lugar de `Matrix4` ya que realmente solo necesitamos el componente de rotación de la matriz.

</div>

```rust
#[repr(C)]
#[derive(Debug, Copy, Clone, bytemuck::Pod, bytemuck::Zeroable)]
#[allow(dead_code)]
struct InstanceRaw {
    model: [[f32; 4]; 4],
    normal: [[f32; 3]; 3],
}

impl model::Vertex for InstanceRaw {
    fn desc() -> wgpu::VertexBufferLayout<'static> {
        use std::mem;
        wgpu::VertexBufferLayout {
            array_stride: mem::size_of::<InstanceRaw>() as wgpu::BufferAddress,
            // We need to switch from using a step mode of Vertex to Instance
            // This means that our shaders will only change to use the next
            // instance when the shader starts processing a new instance
            step_mode: wgpu::VertexStepMode::Instance,
            attributes: &[
                wgpu::VertexAttribute {
                    offset: 0,
                    // While our vertex shader only uses locations 0, and 1 now, in later tutorials, we'll
                    // be using 2, 3, and 4 for Vertex. We'll start at slot 5 to not conflict with them later
                    shader_location: 5,
                    format: wgpu::VertexFormat::Float32x4,
                },
                // A mat4 takes up 4 vertex slots as it is technically 4 vec4s. We need to define a slot
                // for each vec4. We don't have to do this in code, though.
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
                // NEW!
                wgpu::VertexAttribute {
                    offset: mem::size_of::<[f32; 16]>() as wgpu::BufferAddress,
                    shader_location: 9,
                    format: wgpu::VertexFormat::Float32x3,
                },
                wgpu::VertexAttribute {
                    offset: mem::size_of::<[f32; 19]>() as wgpu::BufferAddress,
                    shader_location: 10,
                    format: wgpu::VertexFormat::Float32x3,
                },
                wgpu::VertexAttribute {
                    offset: mem::size_of::<[f32; 22]>() as wgpu::BufferAddress,
                    shader_location: 11,
                    format: wgpu::VertexFormat::Float32x3,
                },
            ],
        }
    }
}
```

Necesitamos modificar `Instance` para crear la matriz normal.

```rust
struct Instance {
    position: cgmath::Vector3<f32>,
    rotation: cgmath::Quaternion<f32>,
}

impl Instance {
    fn to_raw(&self) -> InstanceRaw {
        let model =
            cgmath::Matrix4::from_translation(self.position) * cgmath::Matrix4::from(self.rotation);
        InstanceRaw {
            model: model.into(),
            // NEW!
            normal: cgmath::Matrix3::from(self.rotation).into(),
        }
    }
}
```

Ahora, necesitamos reconstruir la matriz normal en el sombreador de vértices.

```wgsl
struct InstanceInput {
    @location(5) model_matrix_0: vec4<f32>,
    @location(6) model_matrix_1: vec4<f32>,
    @location(7) model_matrix_2: vec4<f32>,
    @location(8) model_matrix_3: vec4<f32>,
    // NEW!
    @location(9) normal_matrix_0: vec3<f32>,
    @location(10) normal_matrix_1: vec3<f32>,
    @location(11) normal_matrix_2: vec3<f32>,
};

struct VertexOutput {
    @builtin(position) clip_position: vec4<f32>,
    @location(0) tex_coords: vec2<f32>,
    @location(1) world_normal: vec3<f32>,
    @location(2) world_position: vec3<f32>,
};

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
    // NEW!
    let normal_matrix = mat3x3<f32>(
        instance.normal_matrix_0,
        instance.normal_matrix_1,
        instance.normal_matrix_2,
    );
    var out: VertexOutput;
    out.tex_coords = model.tex_coords;
    out.world_normal = normal_matrix * model.normal; // UPDATED!
    var world_position: vec4<f32> = model_matrix * vec4<f32>(model.position, 1.0);
    out.world_position = world_position.xyz;
    out.clip_position = camera.view_proj * world_position;
    return out;
}
```

<div class="note">

Actualmente estoy haciendo cosas en [espacio mundial](https://gamedev.stackexchange.com/questions/65783/what-are-world-space-and-eye-space-in-game-development). Hacer cosas en espacio de vista, también conocido como espacio ocular, es más estándar ya que los objetos pueden tener problemas de iluminación cuando están más lejos del origen. Si quisiéramos usar espacio de vista, también habríamos incluido la rotación debida a la matriz de vista. También tendríamos que transformar la posición de nuestra luz usando algo como `view_matrix * model_matrix * light_position` para evitar que el cálculo se arruine cuando la cámara se mueve.

Hay ventajas en usar espacio de vista. La principal es que cuando tienes mundos masivos haciendo iluminación y otros cálculos en espaciado de modelo, puede causar problemas ya que la precisión de punto flotante se degrada cuando los números se vuelven muy grandes. El espacio de vista mantiene la cámara en el origen, lo que significa que todos los cálculos usarán números más pequeños. Las matemáticas de iluminación reales terminan siendo iguales, pero requiere un poco más de configuración.

</div>

Con ese cambio, nuestra iluminación ahora se ve correcta.

![./diffuse_right.png](./diffuse_right.png)

Traer de vuelta nuestros otros objetos y agregar la iluminación ambiental nos da esto.

![./ambient_diffuse_lighting.png](./ambient_diffuse_lighting.png);

<div class="note">

Si puedes garantizar que tu matriz de modelo siempre aplicará escala uniforme a tus objetos, puedes arreglártelas solo con la matriz de modelo. El usuario de Github @julhe compartió conmigo este código que hace el truco:

```wgsl
out.world_normal = (model_matrix * vec4<f32>(model.normal, 0.0)).xyz;
```

Esto funciona explotando el hecho de que al multiplicar una matriz 4x4 por un vector con 0 en el componente w, solo se aplicarán la rotación y la escala al vector. Sin embargo, necesitarás normalizar este vector, ya que las normales necesitan tener longitud unitaria para que los cálculos funcionen.

El factor de escala *necesita* ser uniforme para que esto funcione. Si no lo es, la normal resultante estará sesgada, como puedes ver en la siguiente imagen.

![./normal-scale-issue.png](./normal-scale-issue.png)

</div>

## Iluminación Especular

La iluminación especular describe los reflejos que aparecen en los objetos cuando se ven desde ciertos ángulos. Si alguna vez has mirado un auto, son las partes super brillantes. Básicamente, algo de luz puede reflejarse en la superficie como en un espejo. La ubicación del reflejo cambia dependiendo del ángulo en el que lo veas.

![./specular_diagram.png](./specular_diagram.png)

Debido a que esto es relativo al ángulo de vista, vamos a necesitar pasar la posición de la cámara tanto al sombreador de fragmentos como al sombreador de vértices.

```wgsl
struct Camera {
    view_pos: vec4<f32>,
    view_proj: mat4x4<f32>,
}
@group(1) @binding(0)
var<uniform> camera: Camera;
```

<div class="note">

No olvides actualizar la estructura `Camera` en `light.wgsl` también, ya que si no coincide con la estructura `CameraUniform` en rust, la luz se renderizará incorrectamente.

</div>

También vamos a necesitar actualizar la estructura `CameraUniform`.

```rust
// lib.rs
#[repr(C)]
#[derive(Copy, Clone, bytemuck::Pod, bytemuck::Zeroable)]
struct CameraUniform {
    view_position: [f32; 4],
    view_proj: [[f32; 4]; 4],
}

impl CameraUniform {
    fn new() -> Self {
        Self {
            view_position: [0.0; 4],
            view_proj: cgmath::Matrix4::identity().into(),
        }
    }

    fn update_view_proj(&mut self, camera: &Camera) {
        // We're using Vector4 because of the uniforms 16 byte spacing requirement
        self.view_position = camera.eye.to_homogeneous().into();
        self.view_proj = (OPENGL_TO_WGPU_MATRIX * camera.build_view_projection_matrix()).into();
    }
}
```

Dado que ahora queremos usar nuestros uniformes en el sombreador de fragmentos, necesitamos cambiar su visibilidad.

```rust
// lib.rs
let camera_bind_group_layout = device.create_bind_group_layout(&wgpu::BindGroupLayoutDescriptor {
    entries: &[
        wgpu::BindGroupLayoutBinding {
            // ...
            visibility: wgpu::ShaderStages::VERTEX | wgpu::ShaderStages::FRAGMENT, // Updated!
            // ...
        },
        // ...
    ],
    label: None,
});
```

Vamos a obtener la dirección desde la posición del fragmento hacia la cámara y usarla con la normal para calcular el `reflect_dir`.

```wgsl
// shader.wgsl
// In the fragment shader...
let view_dir = normalize(camera.view_pos.xyz - in.world_position);
let reflect_dir = reflect(-light_dir, in.world_normal);
```

Luego, usamos el producto punto para calcular el `specular_strength` y usamos eso para calcular el `specular_color`.

```wgsl
let specular_strength = pow(max(dot(view_dir, reflect_dir), 0.0), 32.0);
let specular_color = specular_strength * light.color;
```

Finalmente, lo agregamos al resultado.

```wgsl
let result = (ambient_color + diffuse_color + specular_color) * object_color.xyz;
```

Con eso, deberías tener algo como esto.

![./ambient_diffuse_specular_lighting.png](./ambient_diffuse_specular_lighting.png)

Si solo miramos el `specular_color` por sí solo, obtenemos esto.

![./specular_lighting.png](./specular_lighting.png)

## La dirección media

Hasta este punto, en realidad solo hemos implementado la parte Phong de Blinn-Phong. El modelo de reflexión de Phong funciona bien, pero puede fallar bajo [ciertas circunstancias](https://learnopengl.com/Advanced-Lighting/Advanced-Lighting). La parte Blinn de Blinn-Phong viene del darse cuenta de que si sumas `view_dir` y `light_dir` juntos, normalizas el resultado y usas el producto punto de eso y la `normal`, obtienes resultados aproximadamente iguales sin los problemas que tenía usar `reflect_dir`.

```wgsl
let view_dir = normalize(camera.view_pos.xyz - in.world_position);
let half_dir = normalize(view_dir + light_dir);

let specular_strength = pow(max(dot(in.world_normal, half_dir), 0.0), 32.0);
```

Es difícil notar la diferencia, pero aquí están los resultados.

![./half_dir.png](./half_dir.png)

## Demo

<WasmExample example="tutorial10_lighting"></WasmExample>

<AutoGithubLink/>
