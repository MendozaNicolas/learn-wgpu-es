# ¡Primera Versión Mayor! (22.0)

Aquí solo hay algunos cambios. Primero, todas las estructuras de
configuración relacionadas con shaders ahora tienen un campo `compilation_options`.
Por ahora simplemente lo estoy dejando como `Default::default()`, pero si
tienes necesidades específicas de compilación está disponible para ti.

Lo siguiente es que `RenderPipelineDescriptor` y `ComputePipelineDescriptor`
ahora tienen un campo `cache`. Esto te permite proporcionar un caché para
usar durante la compilación de shaders. Esto es realmente útil solo para
dispositivos Android ya que la mayoría del hardware de escritorio y los
controladores proporcionan caché. Lo he dejado como `None` por ahora.

`DeviceDescriptor` ahora tiene un campo `memory_hint`. Puedes usar esto para
pedirle a la GPU que priorice el rendimiento, el uso de memoria, o permitirte
solicitar un tamaño de bloque de memoria personalizado. Sin embargo, estas
son solo sugerencias y el hardware tiene la palabra final en cómo hacer las
cosas. Lo he dejado como `Default::default()` por ahora.
