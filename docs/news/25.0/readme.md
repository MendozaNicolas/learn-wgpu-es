# ¡Versión 25.0!

Al igual que en la versión 24.0, no ha cambiado mucho en el tutorial. Si
quieres ver las notas de parche completas, puedes consultarlas
[aquí](https://github.com/gfx-rs/wgpu/releases/tag/v25.0.0)

Sin embargo, dos cosas sí han cambiado:

1. `requestDevice` ahora toma un parámetro en lugar de 2.
y el trace se ha movido a `DeviceDescriptor`. Aquí hay
un fragmento de código:

```rust
let (device, queue) = adapter.request_device(
    &wgpu::DeviceDescriptor {
        required_features: wgpu::Features::empty(),
        required_limits: if cfg!(target_arch = "wasm32") {
            wgpu::Limits::downlevel_webgl2_defaults()
        } else {
            wgpu::Limits::default()
        },
        label: None,
        memory_hints: Default::default(),
        trace: wgpu::Trace::Off, // NEW!
    },
    // REMOVED
).await.unwrap();
```

2. `Device::poll()` toma `PollType` en lugar de `Maintain`:

```
device.poll(wgpu::PollType::wait_indefinitely()).unwrap();
```

¡Eso es prácticamente todo! Como siempre, siéntete libre de crear un issue/PR
en el repositorio si me falta algo.
