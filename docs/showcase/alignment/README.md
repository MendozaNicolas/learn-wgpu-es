# Diseño de Memoria en WGSL

<div class="warn">

Esta página se está reelaborando actualmente. Quiero entender los temas un poco mejor, pero como la versión 0.12 ya está disponible, quiero lanzar lo que tengo por ahora.

</div>

## Alineación de búferes de vértices e índices

Los búferes de vértices requieren definir un `VertexBufferLayout`, por lo que la alineación de memoria es lo que le indiques a WebGPU que debe ser. Esto puede ser muy conveniente para reducir el uso de memoria en la GPU.

El búfer de índices utiliza la alineación del tipo primitivo que especifiques a través del `IndexFormat` que pasas a `RenderEncoder::set_index_buffer()`.

## Alineación de búferes Uniform y Storage

Los GPUs están diseñados para procesar miles de píxeles en paralelo. Para lograr esto, fue necesario hacer algunos sacrificios. El hardware gráfico prefiere que todos los bytes que pretendas procesar estén alineados por potencias de 2. Los detalles exactos de por qué ocurre esto están más allá de mi nivel de conocimiento, pero es importante saberlo para que puedas solucionar los problemas por los que tus shaders no funcionan correctamente.

<!-- The address of the position of an instance in memory has to be a multiple of its alignment. Normally alignment is the same as size. Exceptions are vec3, structs, and arrays. A vec3 is padded to be a vec4 which means it behaves as if it was a vec4 just that the last entry is not used. -->

Echemos un vistazo a la siguiente tabla:

---------------------------------------------------------------
| Type                   | Alignment in Bytes | Size in Bytes |
|------------------------|--------------------|---------------|
| scalar (i32, u32, f32) |                  4 |             4 |
| vec2&lt;T&gt;          |                  8 |             8 |
| vec3&lt;T&gt;          |             **16** |            12 |
| vec4&lt;T&gt;          |                 16 |            16 |

Puedes ver que para `vec3` la alineación es la siguiente potencia de 2 respecto al tamaño, 16. Esto puede sorprender a principiantes (e incluso a veteranos) ya que no es la más intuitiva. Esto se vuelve especialmente importante cuando comenzamos a distribuir structs. Tomemos el struct de luz del [tutorial de iluminación](../../intermediate/tutorial10-lighting/#seeing-the-light):

Puedes ver la tabla completa de alineaciones en la sección [4.3.7.1 de la especificación WGSL](https://www.w3.org/TR/WGSL/#alignment-and-size)

```wgsl
struct Light {
    position: vec3<f32>,
    color: vec3<f32>,
}
```

¿Cuál es entonces la alineación de este struct? Tu primera suposición sería que es la suma de las alineaciones de los campos individuales. Eso podría tener sentido si estuviéramos en el mundo de Rust, pero en el mundo de los shaders, es un poco más complicado. La alineación de un struct dado se da por la siguiente ecuación:

```
// S is the struct in question
// M is a member of the struct
AlignOf(S) = max(AlignOfMember(S, M1), ... , AlignOfMember(S, Mn))
```

Básicamente, la alineación del struct es el máximo de las alineaciones de los miembros del struct. Esto significa que:

```
AlignOf(Light) 
    = max(AlignOfMember(Light, position), AlignOfMember(Light, color))
    = max(16, 16)
    = 16
```

Por eso `LightUniform` tiene esos campos de relleno. WGPU no lo aceptará si los datos no están alineados correctamente.

## Cómo lidiar con problemas de alineación

En general, 16 es la alineación máxima que verás. En ese caso, podrías pensar que deberíamos ser capaces de hacer algo como lo siguiente:

```rust
#[repr(C, align(16))]
#[derive(Debug, Copy, Clone, bytemuck::Pod, bytemuck::Zeroable)]
struct LightUniform {
    position: [f32; 3],
    color: [f32; 3],
}
```

Pero esto no compilará. El [crate bytemuck](https://docs.rs/bytemuck/) no funciona con structs que tienen bytes de relleno implícitos. Rust no puede garantizar que la memoria entre los campos se haya inicializado correctamente. Esto me dio un error cuando lo intenté:

```
error[E0512]: cannot transmute between types of different sizes, or dependently-sized types
   --> code/intermediate/tutorial10-lighting/src/main.rs:246:8
    |
246 | struct LightUniform {
    |        ^^^^^^^^^^^^
    |
    = note: source type: `LightUniform` (256 bits)
    = note: target type: `_::{closure#0}::TypeWithoutPadding` (192 bits)
```

## Recursos adicionales

Si estás buscando más información, consulta el [artículo](https://gist.github.com/teoxoy/936891c16c2a3d1c3c5e7204ac6cd76c) de @teoxoy.