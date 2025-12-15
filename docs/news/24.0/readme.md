# Versión 24.0

No incluí la versión 23.0, ¡ya que he estado ocupado con el trabajo y un bebé! No ha habido
muchos cambios entre la versión 22.0 y 24.0, al menos en lo que respecta a este
tutorial.

## Inferencia de punto de entrada

Si un shader tiene solo una función etiquetada con `@vertex` para
shaders de vértices, o `@fragment` para shaders de fragmento, entonces Wgpu
no necesitas especificar el punto de entrada al crear un pipeline de
renderizado. Esto significa que si deseas especificar el punto de entrada,
necesitas envolverlo en una opción.

```rust
device.create_render_pipeline(&wgpu::RenderPipelineDescriptor {
        label: Some(&format!("{:?}", shader)),
        layout: Some(layout),
        vertex: wgpu::VertexState {
            module: &shader,
            entry_point: Some("vs_main"), // Updated
            buffers: vertex_layouts,
            compilation_options: Default::default(),
        },
        fragment: Some(wgpu::FragmentState {
            module: &shader,
            entry_point: Some("fs_main"), // Updated
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
        // ...
    })
```

Lo mismo aplica para pipelines de cómputo.

```rust
let equirect_to_cubemap =
    device.create_compute_pipeline(&wgpu::ComputePipelineDescriptor {
        label: Some("equirect_to_cubemap"),
        layout: Some(&pipeline_layout),
        module: &module,
        entry_point: Some("compute_equirect_to_cubemap"), // Updated
        compilation_options: Default::default(),
        cache: None,
    });
```

## Otros cambios

- `ImageCopyTexture` ha sido renombrado a `TexelCopyTextureInfo`
- `ImageDataLayout` ha sido renombrado a `TexelCopyBufferLayout`
- `ImageCopyBuffer` ha sido renombrado a `TexelCopyBufferInfo`
- `wgpu::Instance::new()` ahora toma una referencia a `&wgpu::InstanceDescriptor`
- `wgpu::SurfaceError::Other` ahora existe

## Hacer funcionar WASM

No estoy seguro si es algo específico de la versión `24.0`, pero tuve
que agregar algo de código a `Cargo.toml` para que `webpack` manejara
WASM correctamente.

```toml
# Esto debe ir en el Cargo.toml en el directorio raíz
[profile.release]
strip = true
```

Si sabes por qué esto es requerido, avísame.

