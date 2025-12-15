# Mapeo de Normales

Con solo iluminación, nuestra escena ya se ve bastante bien. Aún así, nuestros modelos siguen siendo demasiado lisos. Esto es comprensible porque estamos usando un modelo muy simple. Si usáramos una textura que se suponía que debería ser suave, esto no sería un problema, pero nuestra textura de ladrillo se supone que debe ser más áspera. Podríamos resolver esto agregando más geometría, pero eso ralentizaría nuestra escena y sería difícil saber dónde agregar nuevos polígonos. Aquí es donde entra en juego el mapeo de normales.

¿Recuerdas cuando experimentamos con almacenar datos de instancia en una textura en [el tutorial de instanciación](/beginner/tutorial7-instancing/#a-different-way-textures)? Un mapa de normales está haciendo exactamente eso con datos de normales. Usaremos las normales en el mapa de normales en nuestro cálculo de iluminación además de la normal del vértice.

La textura de ladrillo que encontré vino con un mapa de normales. ¡Veamos!

![./cube-normal.png](./cube-normal.png)

Los componentes r, g y b de la textura corresponden a los componentes x, y y z de las normales. Todos los valores z deben ser positivos. Por eso el mapa de normales tiene un tinte azulado.

Necesitaremos modificar nuestra estructura `Material` en `model.rs` para incluir una `normal_texture`.

```rust
pub struct Material {
    pub name: String,
    pub diffuse_texture: texture::Texture,
    pub normal_texture: texture::Texture, // UPDATED!
    pub bind_group: wgpu::BindGroup,
}
```

Tendremos que actualizar el `texture_bind_group_layout` para incluir también el mapa de normales.

```rust
let texture_bind_group_layout = device.create_bind_group_layout(&wgpu::BindGroupLayoutDescriptor {
    entries: &[
        // ...
        // normal map
        wgpu::BindGroupLayoutEntry {
            binding: 2,
            visibility: wgpu::ShaderStages::FRAGMENT,
            ty: wgpu::BindingType::Texture {
                multisampled: false,
                sample_type: wgpu::TextureSampleType::Float { filterable: true },
                view_dimension: wgpu::TextureViewDimension::D2,
            },
            count: None,
        },
        wgpu::BindGroupLayoutEntry {
            binding: 3,
            visibility: wgpu::ShaderStages::FRAGMENT,
            ty: wgpu::BindingType::Sampler(wgpu::SamplerBindingType::Filtering),
            count: None,
        },
    ],
    label: Some("texture_bind_group_layout"),
});
```

Necesitaremos cargar el mapa de normales. Haremos esto en el bucle donde creamos los materiales en la función `load_model()` en `resources.rs`.

```rust
// resources.rs
let mut materials = Vec::new();
for m in obj_materials? {
    let diffuse_texture = load_texture(&m.diffuse_texture, device, queue).await?;
    // NEW!
    let normal_texture = load_texture(&m.normal_texture, device, queue).await?;

    materials.push(model::Material::new(
        device,
        &m.name,
        diffuse_texture,
        normal_texture, // NEW!
        layout,
    ));
}
```

Notarás que estoy usando una función `Material::new()` que no teníamos anteriormente. Aquí está el código para eso:

```rust
impl Material {
    pub fn new(
        device: &wgpu::Device,
        name: &str,
        diffuse_texture: texture::Texture,
        normal_texture: texture::Texture, // NEW!
        layout: &wgpu::BindGroupLayout,
    ) -> Self {
        let bind_group = device.create_bind_group(&wgpu::BindGroupDescriptor {
            layout,
            entries: &[
                wgpu::BindGroupEntry {
                    binding: 0,
                    resource: wgpu::BindingResource::TextureView(&diffuse_texture.view),
                },
                wgpu::BindGroupEntry {
                    binding: 1,
                    resource: wgpu::BindingResource::Sampler(&diffuse_texture.sampler),
                },
                // NEW!
                wgpu::BindGroupEntry {
                    binding: 2,
                    resource: wgpu::BindingResource::TextureView(&normal_texture.view),
                },
                wgpu::BindGroupEntry {
                    binding: 3,
                    resource: wgpu::BindingResource::Sampler(&normal_texture.sampler),
                },
            ],
            label: Some(name),
        });

        Self {
            name: String::from(name),
            diffuse_texture,
            normal_texture, // NEW!
            bind_group,
        }
    }
}
```

Ahora podemos usar la textura en el shader de fragmentos.

```wgsl
// Fragment shader

@group(0) @binding(0)
var t_diffuse: texture_2d<f32>;
@group(0)@binding(1)
var s_diffuse: sampler;
@group(0)@binding(2)
var t_normal: texture_2d<f32>;
@group(0) @binding(3)
var s_normal: sampler;

@fragment
fn fs_main(in: VertexOutput) -> @location(0) vec4<f32> {
    let object_color: vec4<f32> = textureSample(t_diffuse, s_diffuse, in.tex_coords);
    let object_normal: vec4<f32> = textureSample(t_normal, s_normal, in.tex_coords);
    
    // No necesitamos (ni queremos) mucha luz ambiental, así que 0.1 está bien
    let ambient_strength = 0.1;
    let ambient_color = light.color * ambient_strength;

    // Crear los vectores de iluminación
    let tangent_normal = object_normal.xyz * 2.0 - 1.0;
    let light_dir = normalize(light.position - in.world_position);
    let view_dir = normalize(camera.view_pos.xyz - in.world_position);
    let half_dir = normalize(view_dir + light_dir);

    let diffuse_strength = max(dot(tangent_normal, light_dir), 0.0);
    let diffuse_color = light.color * diffuse_strength;

    let specular_strength = pow(max(dot(tangent_normal, half_dir), 0.0), 32.0);
    let specular_color = specular_strength * light.color;

    let result = (ambient_color + diffuse_color + specular_color) * object_color.xyz;

    return vec4<f32>(result, object_color.a);
}
```

Si ejecutamos el código ahora, notarás que las cosas no se ven bastante bien. Comparemos nuestros resultados con el tutorial anterior.

![](./normal_mapping_wrong.png)
![](./ambient_diffuse_specular_lighting.png)

Partes de la escena están oscuras cuando deberían estar iluminadas, y viceversa.

## Espacio Tangente a Espacio Mundial

Mencioné brevemente en el [tutorial de iluminación](/intermediate/tutorial10-lighting/#the-normal-matrix) que estábamos realizando nuestro cálculo de iluminación en "espacio mundial". Esto significaba que toda la escena estaba orientada con respecto al sistema de coordenadas del *mundo*. Cuando extraemos los datos normales de nuestra textura de normales, todas las normales están en lo que se conoce como apuntando aproximadamente en la dirección z positiva. Eso significa que nuestro cálculo de iluminación piensa que todas las superficies de nuestros modelos están orientadas aproximadamente en la misma dirección. Esto se conoce como `espacio tangente`.

Si recordamos el [tutorial de iluminación](/intermediate/tutorial10-lighting/#), usamos la normal del vértice para indicar la dirección de la superficie. Resulta que podemos usar eso para transformar nuestras normales de `espacio tangente` a `espacio mundial`. Para hacer eso, necesitamos profundizar en el álgebra lineal.

Podemos crear una matriz que represente un sistema de coordenadas usando tres vectores que sean perpendiculares (u ortonormales) entre sí. Básicamente, definimos los ejes x, y, y z de nuestro sistema de coordenadas.

```wgsl
let coordinate_system = mat3x3<f32>(
    vec3(1, 0, 0), // x-axis (right)
    vec3(0, 1, 0), // y-axis (up)
    vec3(0, 0, 1)  // z-axis (forward)
);
```

Vamos a crear una matriz que represente el espacio de coordenadas relativo a nuestras normales de vértice. Luego lo usaremos para transformar los datos de nuestro mapa de normales para que estén en espacio mundial.

## La tangente y la bitangente

Tenemos uno de los tres vectores que necesitamos, la normal. ¿Y los otros? Estos son los vectores tangente y bitangente. Una tangente representa cualquier vector paralelo a una superficie (es decir, no intersecta con ella). La tangente siempre es perpendicular al vector normal. La bitangente es un vector tangente que es perpendicular al otro vector tangente. Juntos, la tangente, bitangente y normal representan los ejes x, y, y z, respectivamente.

Algunos formatos de modelo incluyen la tangente y bitangente (a veces llamada binormal) en los datos del vértice, pero OBJ no lo hace. Tendremos que calcularlos manualmente. Afortunadamente, podemos derivar nuestra tangente y bitangente de nuestros datos de vértice existentes. Echa un vistazo al siguiente diagrama.

![](./tangent_space.png)

Básicamente, podemos usar los bordes de nuestros triángulos y nuestra normal para calcular la tangente y la bitangente. Pero primero, necesitamos actualizar nuestra estructura `ModelVertex` en `model.rs`.

```rust
#[repr(C)]
#[derive(Copy, Clone, Debug, bytemuck::Pod, bytemuck::Zeroable)]
pub struct ModelVertex {
    pub position: [f32; 3],
    pub tex_coords: [f32; 2],
    pub normal: [f32; 3],
    // NEW!
    pub tangent: [f32; 3],
    pub bitangent: [f32; 3],
}
```

Necesitaremos actualizar nuestro `VertexBufferLayout` también.

```rust
impl Vertex for ModelVertex {
    fn desc() -> wgpu::VertexBufferLayout<'static> {
        use std::mem;
        wgpu::VertexBufferLayout {
            array_stride: mem::size_of::<ModelVertex>() as wgpu::BufferAddress,
            step_mode: wgpu::VertexStepMode::Vertex,
            attributes: &[
                // ...

                // Tangente y bitangente
                wgpu::VertexAttribute {
                    offset: mem::size_of::<[f32; 8]>() as wgpu::BufferAddress,
                    shader_location: 3,
                    format: wgpu::VertexFormat::Float32x3,
                },
                wgpu::VertexAttribute {
                    offset: mem::size_of::<[f32; 11]>() as wgpu::BufferAddress,
                    shader_location: 4,
                    format: wgpu::VertexFormat::Float32x3,
                },
            ],
        }
    }
}
```

Ahora podemos calcular los nuevos vectores tangente y bitangente. Actualiza la generación de malla en `load_model()` en `resource.rs` para usar el siguiente código:

```rust
let meshes = models
    .into_iter()
    .map(|m| {
        let mut vertices = (0..m.mesh.positions.len() / 3)
            .map(|i| model::ModelVertex {
                position: [
                    m.mesh.positions[i * 3],
                    m.mesh.positions[i * 3 + 1],
                    m.mesh.positions[i * 3 + 2],
                ],
                tex_coords: [m.mesh.texcoords[i * 2], 1.0 - m.mesh.texcoords[i * 2 + 1]],
                normal: [
                    m.mesh.normals[i * 3],
                    m.mesh.normals[i * 3 + 1],
                    m.mesh.normals[i * 3 + 2],
                ],
                // Los calcularemos más tarde
                tangent: [0.0; 3],
                bitangent: [0.0; 3],
            })
            .collect::<Vec<_>>();

        let indices = &m.mesh.indices;
        let mut triangles_included = vec![0; vertices.len()];

        // Calcula tangentes y bitangentes. Vamos a
        // usar los triángulos, así que necesitamos recorrer
        // los índices en trozos de 3
        for c in indices.chunks(3) {
            let v0 = vertices[c[0] as usize];
            let v1 = vertices[c[1] as usize];
            let v2 = vertices[c[2] as usize];

            let pos0: cgmath::Vector3<_> = v0.position.into();
            let pos1: cgmath::Vector3<_> = v1.position.into();
            let pos2: cgmath::Vector3<_> = v2.position.into();

            let uv0: cgmath::Vector2<_> = v0.tex_coords.into();
            let uv1: cgmath::Vector2<_> = v1.tex_coords.into();
            let uv2: cgmath::Vector2<_> = v2.tex_coords.into();

            // Calcula los bordes del triángulo
            let delta_pos1 = pos1 - pos0;
            let delta_pos2 = pos2 - pos0;

            // Esto nos dará una dirección para calcular la
            // tangente y bitangente
            let delta_uv1 = uv1 - uv0;
            let delta_uv2 = uv2 - uv0;

            // Resolver el siguiente sistema de ecuaciones nos
            // dará la tangente y bitangente.
            //     delta_pos1 = delta_uv1.x * T + delta_u.y * B
            //     delta_pos2 = delta_uv2.x * T + delta_uv2.y * B
            // Afortunadamente, el lugar donde encontré esta ecuación proporcionó
            // la solución!
            let r = 1.0 / (delta_uv1.x * delta_uv2.y - delta_uv1.y * delta_uv2.x);
            let tangent = (delta_pos1 * delta_uv2.y - delta_pos2 * delta_uv1.y) * r;
            // Invertimos la bitangente para habilitar normales
            // orientadas a la derecha con el sistema de
            // coordenadas de textura de wgpu
            let bitangent = (delta_pos2 * delta_uv1.x - delta_pos1 * delta_uv2.x) * -r;

            // Usaremos la misma tangente/bitangente para cada vértice en el triángulo
            vertices[c[0] as usize].tangent =
                (tangent + cgmath::Vector3::from(vertices[c[0] as usize].tangent)).into();
            vertices[c[1] as usize].tangent =
                (tangent + cgmath::Vector3::from(vertices[c[1] as usize].tangent)).into();
            vertices[c[2] as usize].tangent =
                (tangent + cgmath::Vector3::from(vertices[c[2] as usize].tangent)).into();
            vertices[c[0] as usize].bitangent =
                (bitangent + cgmath::Vector3::from(vertices[c[0] as usize].bitangent)).into();
            vertices[c[1] as usize].bitangent =
                (bitangent + cgmath::Vector3::from(vertices[c[1] as usize].bitangent)).into();
            vertices[c[2] as usize].bitangent =
                (bitangent + cgmath::Vector3::from(vertices[c[2] as usize].bitangent)).into();

            // Se utiliza para promediar las tangentes/bitangentes
            triangles_included[c[0] as usize] += 1;
            triangles_included[c[1] as usize] += 1;
            triangles_included[c[2] as usize] += 1;
        }

        // Promedia las tangentes/bitangentes
        for (i, n) in triangles_included.into_iter().enumerate() {
            let denom = 1.0 / n as f32;
            let mut v = &mut vertices[i];
            v.tangent = (cgmath::Vector3::from(v.tangent) * denom).into();
            v.bitangent = (cgmath::Vector3::from(v.bitangent) * denom).into();
        }

        let vertex_buffer = device.create_buffer_init(&wgpu::util::BufferInitDescriptor {
            label: Some(&format!("{:?} Vertex Buffer", file_name)),
            contents: bytemuck::cast_slice(&vertices),
            usage: wgpu::BufferUsages::VERTEX,
        });
        let index_buffer = device.create_buffer_init(&wgpu::util::BufferInitDescriptor {
            label: Some(&format!("{:?} Index Buffer", file_name)),
            contents: bytemuck::cast_slice(&m.mesh.indices),
            usage: wgpu::BufferUsages::INDEX,
        });

        model::Mesh {
            name: file_name.to_string(),
            vertex_buffer,
            index_buffer,
            num_elements: m.mesh.indices.len() as u32,
            material: m.mesh.material_id.unwrap_or(0),
        }
    })
    .collect::<Vec<_>>();
```

## Espacio Mundial a Espacio Tangente

Dado que el mapa de normales, por defecto, está en espacio tangente, necesitamos transformar todas las otras variables utilizadas en ese cálculo al espacio tangente también. Necesitaremos construir la matriz tangente en el shader de vértices. Primero, necesitamos que nuestro `VertexInput` incluya la tangente y bitangentes que calculamos anteriormente.

```wgsl
struct VertexInput {
    @location(0) position: vec3<f32>,
    @location(1) tex_coords: vec2<f32>,
    @location(2) normal: vec3<f32>,
    @location(3) tangent: vec3<f32>,
    @location(4) bitangent: vec3<f32>,
};
```

A continuación, construiremos la `tangent_matrix` y luego transformaremos la posición de luz y vista del vértice al espacio tangente.

```wgsl
struct VertexOutput {
    @builtin(position) clip_position: vec4<f32>,
    @location(0) tex_coords: vec2<f32>,
    // UPDATED!
    @location(1) tangent_position: vec3<f32>,
    @location(2) tangent_light_position: vec3<f32>,
    @location(3) tangent_view_position: vec3<f32>,
};

@vertex
fn vs_main(
    model: VertexInput,
    instance: InstanceInput,
) -> VertexOutput {
    // ...
    let normal_matrix = mat3x3<f32>(
        instance.normal_matrix_0,
        instance.normal_matrix_1,
        instance.normal_matrix_2,
    );

    // Construir la matriz tangente
    let world_normal = normalize(normal_matrix * model.normal);
    let world_tangent = normalize(normal_matrix * model.tangent);
    let world_bitangent = normalize(normal_matrix * model.bitangent);
    let tangent_matrix = transpose(mat3x3<f32>(
        world_tangent,
        world_bitangent,
        world_normal,
    ));

    let world_position = model_matrix * vec4<f32>(model.position, 1.0);

    var out: VertexOutput;
    out.clip_position = camera.view_proj * world_position;
    out.tex_coords = model.tex_coords;
    out.tangent_position = tangent_matrix * world_position.xyz;
    out.tangent_view_position = tangent_matrix * camera.view_pos.xyz;
    out.tangent_light_position = tangent_matrix * light.position;
    return out;
}
```

Finalmente, actualizaremos el shader de fragmentos para usar estos valores de iluminación transformados.

```wgsl
@fragment
fn fs_main(in: VertexOutput) -> @location(0) vec4<f32> {
    // Muestrear texturas..

    // Crear los vectores de iluminación
    let tangent_normal = object_normal.xyz * 2.0 - 1.0;
    let light_dir = normalize(in.tangent_light_position - in.tangent_position);
    let view_dir = normalize(in.tangent_view_position - in.tangent_position);

    // Realizar cálculos de iluminación...
}
```

Obtenemos lo siguiente de este cálculo.

![](./normal_mapping_correct.png)

## Srgb y texturas de normales

Hemos estado usando `Rgba8UnormSrgb` para todas nuestras texturas. Srgb es un espacio de color no lineal. Es ideal para monitores porque la percepción del color humano tampoco es lineal y Srgb fue diseñado para coincidir con las peculiaridades de nuestra percepción del color humano.

Pero Srgb es un espacio de color inapropiado para datos que deben operarse matemáticamente. Tales datos deben estar en un espacio de color lineal (no gamma-corregido). Cuando una GPU muestrea una textura con Srgb en el nombre, convierte los datos del espacio Srgb gamma-corregido no lineal a un espacio de color no gamma-corregido lineal primero para que puedas hacer matemáticas en él (y hace la conversión opuesta si escribes de nuevo a una textura Srgb).

Los mapas de normales ya se almacenan en un formato lineal. Entonces deberíamos especificar el espacio lineal para la textura para que no haga una conversión inapropiada cuando leamos de ella.

Necesitamos especificar `Rgba8Unorm` cuando creamos la textura. Agreguemos un método `is_normal_map` a nuestra estructura Texture.

```rust
pub fn from_image(
    device: &wgpu::Device,
    queue: &wgpu::Queue,
    img: &image::DynamicImage,
    label: Option<&str>,
    is_normal_map: bool, // NUEVO!
) -> Result<Self> {
    // ...
    // NUEVO!
    let format = if is_normal_map {
        wgpu::TextureFormat::Rgba8Unorm
    } else {
        wgpu::TextureFormat::Rgba8UnormSrgb
    };
    let texture = device.create_texture(&wgpu::TextureDescriptor {
        label,
        size,
        mip_level_count: 1,
        sample_count: 1,
        dimension: wgpu::TextureDimension::D2,
        // ACTUALIZADO!
        format,
        usage: wgpu::TextureUsages::TEXTURE_BINDING | wgpu::TextureUsages::COPY_DST,
        view_formats: &[],
    });

    // ...

    Ok(Self {
        texture,
        view,
        sampler,
    })
}
```

Necesitaremos propagar este cambio a los otros métodos que lo usan.

```rust
pub fn from_bytes(
    device: &wgpu::Device,
    queue: &wgpu::Queue,
    bytes: &[u8],
    label: &str,
    is_normal_map: bool, // NUEVO!
) -> Result<Self> {
    let img = image::load_from_memory(bytes)?;
    Self::from_image(device, queue, &img, Some(label), is_normal_map) // ACTUALIZADO!
}
```

También necesitamos actualizar `resource.rs`.

```rust
pub async fn load_texture(
    file_name: &str,
    is_normal_map: bool,
    device: &wgpu::Device,
    queue: &wgpu::Queue,
) -> anyhow::Result<texture::Texture> {
    let data = load_binary(file_name).await?;
    texture::Texture::from_bytes(device, queue, &data, file_name, is_normal_map)
}

pub async fn load_model(
    file_name: &str,
    device: &wgpu::Device,
    queue: &wgpu::Queue,
    layout: &wgpu::BindGroupLayout,
) -> anyhow::Result<model::Model> {
    // ...

    let mut materials = Vec::new();
    for m in obj_materials? {
        let diffuse_texture = load_texture(&m.diffuse_texture, false, device, queue).await?; // ACTUALIZADO!
        let normal_texture = load_texture(&m.normal_texture, true, device, queue).await?; // ACTUALIZADO!

        materials.push(model::Material::new(
            device,
            &m.name,
            diffuse_texture,
            normal_texture,
            layout,
        ));
    }
}

```

Eso nos da lo siguiente.

![](./no_srgb.png)

## Cosas sin relación

Quería jugar con otros materiales, así que agregué un `draw_model_instanced_with_material()` al trait `DrawModel`.

```rust
pub trait DrawModel<'a> {
    // ...
    fn draw_model_instanced_with_material(
        &mut self,
        model: &'a Model,
        material: &'a Material,
        instances: Range<u32>,
        camera_bind_group: &'a wgpu::BindGroup,
        light_bind_group: &'a wgpu::BindGroup,
    );
}

impl<'a, 'b> DrawModel<'b> for wgpu::RenderPass<'a>
where
    'b: 'a,
{
    // ...
    fn draw_model_instanced_with_material(
        &mut self,
        model: &'b Model,
        material: &'b Material,
        instances: Range<u32>,
        camera_bind_group: &'b wgpu::BindGroup,
        light_bind_group: &'b wgpu::BindGroup,
    ) {
        for mesh in &model.meshes {
            self.draw_mesh_instanced(mesh, material, instances.clone(), camera_bind_group, light_bind_group);
        }
    }
}
```

Encontré una textura de adoquines con un mapa de normales coincidente y creé un `debug_material` para eso.

```rust
// lib.rs
impl State {
    async fn new(window: &Window) -> Result<Self> {
        // ...
        let debug_material = {
            let diffuse_bytes = include_bytes!("../res/cobble-diffuse.png");
            let normal_bytes = include_bytes!("../res/cobble-normal.png");

            let diffuse_texture = texture::Texture::from_bytes(&device, &queue, diffuse_bytes, "res/alt-diffuse.png", false).unwrap();
            let normal_texture = texture::Texture::from_bytes(&device, &queue, normal_bytes, "res/alt-normal.png", true).unwrap();

            model::Material::new(&device, "alt-material", diffuse_texture, normal_texture, &texture_bind_group_layout)
        };
        Self {
            // ...
            #[allow(dead_code)]
            debug_material,
        }
    }
}
```

Luego, para renderizar con el `debug_material`, usé el `draw_model_instanced_with_material()` que creé.

```rust
render_pass.set_pipeline(&self.render_pipeline);
render_pass.draw_model_instanced_with_material(
    &self.obj_model,
    &self.debug_material,
    0..self.instances.len() as u32,
    &self.camera_bind_group,
    &self.light_bind_group,
);
```

Eso nos da algo como esto.

![](./debug_material.png)

Puedes encontrar las texturas que utilizo en el repositorio de GitHub.

## Demo

<WasmExample example="tutorial11_normals"></WasmExample>

<AutoGithubLink/>
