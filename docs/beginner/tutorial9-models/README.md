# Carga de Modelos

Hasta ahora, hemos estado creando nuestros modelos manualmente. Aunque esta es una forma aceptable de hacerlo, es muy lento si queremos incluir modelos complejos con muchos polígonos. Por esta razón, vamos a modificar nuestro código para aprovechar el formato de modelo `.obj` para que podamos crear un modelo en software como Blender y mostrarlo en nuestro código.

Nuestro archivo `lib.rs` está bastante desordenado. Vamos a crear un archivo `model.rs` donde podemos poner nuestro código de carga de modelos.

```rust
// model.rs
pub trait Vertex {
    fn desc() -> wgpu::VertexBufferLayout<'static>;
}

#[repr(C)]
#[derive(Copy, Clone, Debug, bytemuck::Pod, bytemuck::Zeroable)]
pub struct ModelVertex {
    pub position: [f32; 3],
    pub tex_coords: [f32; 2],
    pub normal: [f32; 3],
}

impl Vertex for ModelVertex {
    fn desc() -> wgpu::VertexBufferLayout<'static> {
        todo!();
    }
}
```

Notarás un par de cosas aquí. En `lib.rs`, teníamos `Vertex` como una estructura, pero aquí estamos usando un trait. Podríamos tener múltiples tipos de vértices (modelo, interfaz de usuario, datos de instancia, etc.). Hacer de `Vertex` un trait nos permitirá abstraer el código de creación de `VertexBufferLayout` para hacer que la creación de `RenderPipeline`s sea más simple.

Otra cosa a mencionar es el campo `normal` en `ModelVertex`. No lo usaremos hasta que hablemos de iluminación, pero lo añadiremos a la estructura por ahora.

Vamos a definir nuestro `VertexBufferLayout`.

```rust
impl Vertex for ModelVertex {
    fn desc() -> wgpu::VertexBufferLayout<'static> {
        use std::mem;
        wgpu::VertexBufferLayout {
            array_stride: mem::size_of::<ModelVertex>() as wgpu::BufferAddress,
            step_mode: wgpu::VertexStepMode::Vertex,
            attributes: &[
                wgpu::VertexAttribute {
                    offset: 0,
                    shader_location: 0,
                    format: wgpu::VertexFormat::Float32x3,
                },
                wgpu::VertexAttribute {
                    offset: mem::size_of::<[f32; 3]>() as wgpu::BufferAddress,
                    shader_location: 1,
                    format: wgpu::VertexFormat::Float32x2,
                },
                wgpu::VertexAttribute {
                    offset: mem::size_of::<[f32; 5]>() as wgpu::BufferAddress,
                    shader_location: 2,
                    format: wgpu::VertexFormat::Float32x3,
                },
            ],
        }
    }
}
```

Esto es básicamente lo mismo que el `VertexBufferLayout` original, pero añadimos un `VertexAttribute` para la `normal`. Elimina la estructura `Vertex` en `lib.rs` ya que no la necesitaremos más, y usa nuestro nuevo `Vertex` de `model` para el `RenderPipeline`.

También eliminaremos nuestro `vertex_buffer`, `index_buffer` y `num_indices` hechos a mano.

```rust
let render_pipeline = device.create_render_pipeline(&wgpu::RenderPipelineDescriptor {
    // ...
    vertex: wgpu::VertexState {
        // ...
        buffers: &[model::ModelVertex::desc(), InstanceRaw::desc()],
    },
    // ...
});
```

Como el método `desc` está implementado en el trait `Vertex`, el trait necesita ser importado antes de que el método sea accesible. Pon la importación hacia la parte superior del archivo con las otras.

```rust
use model::Vertex;
```

Con todo eso en su lugar, necesitamos un modelo para renderizar. Si ya tienes uno, eso es excelente, pero he proporcionado un [archivo zip](https://github.com/sotrh/learn-wgpu/blob/master/code/beginner/tutorial9-models/res/cube.zip) con el modelo y todas sus texturas. Vamos a poner este modelo en una nueva carpeta `res` junto a la carpeta `src` existente.

## Acceso a archivos en la carpeta res

Cuando Cargo construye y ejecuta nuestro programa, establece lo que se conoce como el directorio de trabajo actual. Este directorio generalmente contiene el `Cargo.toml` raíz de tu proyecto. La ruta a nuestra carpeta res puede diferir según la estructura del proyecto. En la carpeta `res`, el código de ejemplo para este tutorial se encuentra en `code/beginner/tutorial9-models/res/`. Cuando carguemos nuestro modelo, podríamos usar esta ruta y simplemente añadir `cube.obj`. Esto está bien, pero si cambiamos la estructura de nuestro proyecto, nuestro código se romperá.

Vamos a solucionar eso modificando nuestro script de construcción para copiar nuestra carpeta `res` a donde Cargo crea nuestro ejecutable, y la referenciaremos desde allí. Crea un archivo llamado `build.rs` y añade lo siguiente:

```rust
use anyhow::*;
use fs_extra::copy_items;
use fs_extra::dir::CopyOptions;
use std::env;

fn main() -> Result<()> {
    // This tells Cargo to rerun this script if something in /res/ changes.
    println!("cargo:rerun-if-changed=res/*");

    let out_dir = env::var("OUT_DIR")?;
    let mut copy_options = CopyOptions::new();
    copy_options.overwrite = true;
    let mut paths_to_copy = Vec::new();
    paths_to_copy.push("res/");
    copy_items(&paths_to_copy, out_dir, &copy_options)?;

    Ok(())
}
```

<div class="note">

Asegúrate de poner `build.rs` en la misma carpeta que `Cargo.toml`. Si no lo haces, Cargo no lo ejecutará cuando se construya tu crate.

</div>

<div class="note">

`OUT_DIR` es una variable de entorno que Cargo usa para especificar dónde se construirá nuestra aplicación.

</div>

Necesitarás modificar tu `Cargo.toml` para que esto funcione correctamente. Añade lo siguiente debajo de tu bloque `[dependencies]`.

```toml
[build-dependencies]
anyhow = "1.0"
fs_extra = "1.2"
glob = "0.3"
```

## Acceso a archivos desde WASM

Por diseño, no puedes acceder a archivos en el sistema de archivos del usuario en Web Assembly. En su lugar, sirviremos esos archivos usando un servidor web y luego cargaremos esos archivos en nuestro código usando una solicitud http. Para simplificar esto, vamos a crear un archivo llamado `resources.rs` para manejar esto por nosotros. Crearemos dos funciones que carguen archivos de texto y binarios, respectivamente.

```rust
use std::io::{BufReader, Cursor};

use wgpu::util::DeviceExt;

use crate::{model, texture};

#[cfg(target_arch = "wasm32")]
fn format_url(file_name: &str) -> reqwest::Url {
    let window = web_sys::window().unwrap();
    let location = window.location();
    let mut origin = location.origin().unwrap();
    if !origin.ends_with("learn-wgpu") {
        origin = format!("{}/learn-wgpu", origin);
    }
    let base = reqwest::Url::parse(&format!("{}/", origin,)).unwrap();
    base.join(file_name).unwrap()
}

pub async fn load_string(file_name: &str) -> anyhow::Result<String> {
    #[cfg(target_arch = "wasm32")]
    let txt = {
        let url = format_url(file_name);
        reqwest::get(url).await?.text().await?
    };
    #[cfg(not(target_arch = "wasm32"))]
    let txt = {
        let path = std::path::Path::new(env!("OUT_DIR"))
            .join("res")
            .join(file_name);
        std::fs::read_to_string(path)?
    };

    Ok(txt)
}

pub async fn load_binary(file_name: &str) -> anyhow::Result<Vec<u8>> {
    #[cfg(target_arch = "wasm32")]
    let data = {
        let url = format_url(file_name);
        reqwest::get(url).await?.bytes().await?.to_vec()
    };
    #[cfg(not(target_arch = "wasm32"))]
    let data = {
        let path = std::path::Path::new(env!("OUT_DIR"))
            .join("res")
            .join(file_name);
        std::fs::read(path)?
    };

    Ok(data)
}
```

<div class="note">

Estamos usando `OUT_DIR` en el escritorio para acceder a nuestra carpeta `res`.

</div>

Estoy usando [reqwest](https://docs.rs/reqwest) para manejar la carga de solicitudes cuando se usa WASM. Añade lo siguiente a `Cargo.toml`:

```toml
[target.'cfg(target_arch = "wasm32")'.dependencies]
# Other dependencies
reqwest = { version = "0.11" }
```

También necesitaremos añadir la característica `Location` a `web-sys`:

```toml
web-sys = { version = "0.3", features = [
    "Document",
    "Window",
    "Element",
    "Location",
]}
```

Asegúrate de añadir `resources` como un módulo en `lib.rs`:

```rust
mod resources;
```

## Carga de modelos con TOBJ

Vamos a usar la librería [tobj](https://docs.rs/tobj/3.0/tobj/) para cargar nuestro modelo. Vamos a añadirla a nuestro `Cargo.toml`.

```toml
[dependencies]
# other dependencies...
tobj = { version = "3.2", default-features = false, features = ["async"]}
```

Antes de que podamos cargar nuestro modelo, necesitamos un lugar donde ponerlo.

```rust
// model.rs
pub struct Model {
    pub meshes: Vec<Mesh>,
    pub materials: Vec<Material>,
}
```

Notarás que nuestra estructura `Model` tiene un `Vec` para los `meshes` y `materials`. Esto es importante ya que nuestro archivo obj puede incluir múltiples mallas y materiales. Todavía necesitamos crear las clases `Mesh` y `Material`, así que vamos a hacerlo.

```rust
pub struct Material {
    pub name: String,
    pub diffuse_texture: texture::Texture,
    pub bind_group: wgpu::BindGroup,
}

pub struct Mesh {
    pub name: String,
    pub vertex_buffer: wgpu::Buffer,
    pub index_buffer: wgpu::Buffer,
    pub num_elements: u32,
    pub material: usize,
}
```

El `Material` es bastante simple. Es solo el nombre y una textura. Nuestro cubo obj en realidad tiene dos texturas, pero una es un mapa normal, y hablaremos de esos [más tarde](../../intermediate/tutorial11-normals). El nombre es más para propósitos de depuración.

Hablando de texturas, necesitaremos añadir una función para cargar una `Texture` en `resources.rs`.

```rust

pub async fn load_texture(
    file_name: &str,
    device: &wgpu::Device,
    queue: &wgpu::Queue,
) -> anyhow::Result<texture::Texture> {
    let data = load_binary(file_name).await?;
    texture::Texture::from_bytes(device, queue, &data, file_name)
}
```

El método `load_texture` será útil cuando carguemos las texturas de nuestros modelos, ya que `include_bytes!` requiere que conozcamos el nombre del archivo en tiempo de compilación, lo que no podemos garantizar realmente con texturas de modelo.

`Mesh` contiene un búfer de vértices, un búfer de índices y el número de índices en la malla. Estamos usando un `usize` para el material. Este `usize` indexará la lista de `materials` cuando llegue el momento de dibujar.

Con todo eso fuera del camino, podemos proceder a cargar nuestro modelo.

```rust
pub async fn load_model(
    file_name: &str,
    device: &wgpu::Device,
    queue: &wgpu::Queue,
    layout: &wgpu::BindGroupLayout,
) -> anyhow::Result<model::Model> {
    let obj_text = load_string(file_name).await?;
    let obj_cursor = Cursor::new(obj_text);
    let mut obj_reader = BufReader::new(obj_cursor);

    let (models, obj_materials) = tobj::load_obj_buf_async(
        &mut obj_reader,
        &tobj::LoadOptions {
            triangulate: true,
            single_index: true,
            ..Default::default()
        },
        |p| async move {
            let mat_text = load_string(&p).await.unwrap();
            tobj::load_mtl_buf(&mut BufReader::new(Cursor::new(mat_text)))
        },
    )
    .await?;

    let mut materials = Vec::new();
    for m in obj_materials? {
        let diffuse_texture = load_texture(&m.diffuse_texture, device, queue).await?;
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
            ],
            label: None,
        });

        materials.push(model::Material {
            name: m.name,
            diffuse_texture,
            bind_group,
        })
    }

    let meshes = models
        .into_iter()
        .map(|m| {
                let vertices = (0..m.mesh.positions.len() / 3)
                .map(|i| {
                    if m.mesh.normals.is_empty(){
                        model::ModelVertex {
                            position: [
                                m.mesh.positions[i * 3],
                                m.mesh.positions[i * 3 + 1],
                                m.mesh.positions[i * 3 + 2],
                            ],
                            tex_coords: [m.mesh.texcoords[i * 2], 1.0 - m.mesh.texcoords[i * 2 + 1]],
                            normal: [0.0, 0.0, 0.0],
                        }
                    }else{
                        model::ModelVertex {
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
                        }
                    }
                })
                .collect::<Vec<_>>();

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

    Ok(model::Model { meshes, materials })
}

```

## Renderizado de una malla

Antes de que podamos dibujar el modelo, necesitamos poder dibujar una malla individual. Vamos a crear un trait llamado `DrawModel` e implementarlo para `RenderPass`.

```rust
// model.rs
pub trait DrawModel<'a> {
    fn draw_mesh(&mut self, mesh: &'a Mesh);
    fn draw_mesh_instanced(
        &mut self,
        mesh: &'a Mesh,
        instances: Range<u32>,
    );
}
impl<'a, 'b> DrawModel<'b> for wgpu::RenderPass<'a>
where
    'b: 'a,
{
    fn draw_mesh(&mut self, mesh: &'b Mesh) {
        self.draw_mesh_instanced(mesh, 0..1);
    }

    fn draw_mesh_instanced(
        &mut self,
        mesh: &'b Mesh,
        instances: Range<u32>,
    ){
        self.set_vertex_buffer(0, mesh.vertex_buffer.slice(..));
        self.set_index_buffer(mesh.index_buffer.slice(..), wgpu::IndexFormat::Uint32);
        self.draw_indexed(0..mesh.num_elements, 0, instances);
    }
}
```

Podríamos haber puesto estos métodos en un `impl Model`, pero sentí que tenía más sentido que `RenderPass` hiciera toda la renderización, ya que ese es su trabajo. Sin embargo, esto significa que tenemos que importar `DrawModel` cuando vayamos a renderizar.

Cuando eliminamos `vertex_buffer`, etc., también eliminamos su configuración de render_pass.

```rust
// lib.rs
render_pass.set_vertex_buffer(1, self.instance_buffer.slice(..));
render_pass.set_pipeline(&self.render_pipeline);
render_pass.set_bind_group(0, &self.diffuse_bind_group, &[]);
render_pass.set_bind_group(1, &self.camera_bind_group, &[]);

use model::DrawModel;
render_pass.draw_mesh_instanced(&self.obj_model.meshes[0], 0..self.instances.len() as u32);
```

Antes de eso, sin embargo, necesitamos cargar el modelo y guardarlo en `State`. Pon lo siguiente en `State::new()`.

```rust
let obj_model =
    resources::load_model("cube.obj", &device, &queue, &texture_bind_group_layout)
        .await
        .unwrap();
```

Nuestro nuevo modelo es un poco más grande que el anterior, así que necesitaremos ajustar un poco el espaciado de nuestras instancias.

```rust
const SPACE_BETWEEN: f32 = 3.0;
let instances = (0..NUM_INSTANCES_PER_ROW).flat_map(|z| {
    (0..NUM_INSTANCES_PER_ROW).map(move |x| {
        let x = SPACE_BETWEEN * (x as f32 - NUM_INSTANCES_PER_ROW as f32 / 2.0);
        let z = SPACE_BETWEEN * (z as f32 - NUM_INSTANCES_PER_ROW as f32 / 2.0);

        let position = cgmath::Vector3 { x, y: 0.0, z };

        let rotation = if position.is_zero() {
            cgmath::Quaternion::from_axis_angle(cgmath::Vector3::unit_z(), cgmath::Deg(0.0))
        } else {
            cgmath::Quaternion::from_axis_angle(position.normalize(), cgmath::Deg(45.0))
        };

        Instance {
            position, rotation,
        }
    })
}).collect::<Vec<_>>();
```

Con todo eso hecho, deberías obtener algo como esto.

![cubes.png](./cubes.png)

## Uso de las texturas correctas

Si miras los archivos de textura de nuestro obj, verás que no coinciden con nuestro obj. La textura que queremos ver es esta,

![cube-diffuse.jpg](./cube-diffuse.jpg)

pero seguimos obteniendo nuestra textura de árbol feliz.

La razón de esto es bastante simple. Aunque hemos creado nuestras texturas, no hemos creado un grupo de enlace para darle a `RenderPass`. Todavía estamos usando nuestro `diffuse_bind_group` antiguo. Si queremos cambiar eso, necesitamos usar el grupo de enlace de nuestros materiales - el miembro `bind_group` de la estructura `Material`.

Vamos a añadir un parámetro de material a `DrawModel`.

```rust
pub trait DrawModel<'a> {
    fn draw_mesh(&mut self, mesh: &'a Mesh, material: &'a Material, camera_bind_group: &'a wgpu::BindGroup);
    fn draw_mesh_instanced(
        &mut self,
        mesh: &'a Mesh,
        material: &'a Material,
        instances: Range<u32>,
        camera_bind_group: &'a wgpu::BindGroup,
    );

}

impl<'a, 'b> DrawModel<'b> for wgpu::RenderPass<'a>
where
    'b: 'a,
{
    fn draw_mesh(&mut self, mesh: &'b Mesh, material: &'b Material, camera_bind_group: &'b wgpu::BindGroup) {
        self.draw_mesh_instanced(mesh, material, 0..1, camera_bind_group);
    }

    fn draw_mesh_instanced(
        &mut self,
        mesh: &'b Mesh,
        material: &'b Material,
        instances: Range<u32>,
        camera_bind_group: &'b wgpu::BindGroup,
    ) {
        self.set_vertex_buffer(0, mesh.vertex_buffer.slice(..));
        self.set_index_buffer(mesh.index_buffer.slice(..), wgpu::IndexFormat::Uint32);
        self.set_bind_group(0, &material.bind_group, &[]);
        self.set_bind_group(1, camera_bind_group, &[]);
        self.draw_indexed(0..mesh.num_elements, 0, instances);
    }
}
```

Necesitamos cambiar el código de renderización para reflejar esto.

```rust
render_pass.set_vertex_buffer(1, self.instance_buffer.slice(..));

render_pass.set_pipeline(&self.render_pipeline);

let mesh = &self.obj_model.meshes[0];
let material = &self.obj_model.materials[mesh.material];
render_pass.draw_mesh_instanced(mesh, material, 0..self.instances.len() as u32, &self.camera_bind_group);
```

Con todo eso en su lugar, deberíamos obtener lo siguiente.

![cubes-correct.png](./cubes-correct.png)

## Renderizado del modelo completo

Ahora mismo, estamos especificando la malla y el material directamente. Esto es útil si queremos dibujar una malla con un material diferente. Tampoco estamos renderizando otras partes del modelo (si tuviéramos algunas). Vamos a crear un método para `DrawModel` que dibujará todas las partes del modelo con sus materiales respectivos.

```rust
pub trait DrawModel<'a> {
    // ...
    fn draw_model(&mut self, model: &'a Model, camera_bind_group: &'a wgpu::BindGroup);
    fn draw_model_instanced(
        &mut self,
        model: &'a Model,
        instances: Range<u32>,
        camera_bind_group: &'a wgpu::BindGroup,
    );
}

impl<'a, 'b> DrawModel<'b> for wgpu::RenderPass<'a>
where
    'b: 'a, {
    // ...
    fn draw_model(&mut self, model: &'b Model, camera_bind_group: &'b wgpu::BindGroup) {
        self.draw_model_instanced(model, 0..1, camera_bind_group);
    }

    fn draw_model_instanced(
        &mut self,
        model: &'b Model,
        instances: Range<u32>,
        camera_bind_group: &'b wgpu::BindGroup,
    ) {
        for mesh in &model.meshes {
            let material = &model.materials[mesh.material];
            self.draw_mesh_instanced(mesh, material, instances.clone(), camera_bind_group);
        }
    }
}
```

El código en `lib.rs` cambiará en consecuencia.

```rust
render_pass.set_vertex_buffer(1, self.instance_buffer.slice(..));
render_pass.set_pipeline(&self.render_pipeline);
render_pass.draw_model_instanced(&self.obj_model, 0..self.instances.len() as u32, &self.camera_bind_group);
```

## Demo

<WasmExample example="tutorial9_models"></WasmExample>

<AutoGithubLink/>
