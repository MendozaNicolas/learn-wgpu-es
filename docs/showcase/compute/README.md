# Ejemplo de Compute: Tangentes y Bitangentes

Esto resultó ser más difícil de lo que anticipé. El primer problema que encontré fue cierta corrupción de datos de vértices debido a que el shader estaba leyendo mis datos de vértices incorrectamente. Estaba usando la estructura `ModelVertex` que usé en el [tutorial de mapeo de normales](/intermediate/tutorial11-normals/).

```rust
#[repr(C)]
#[derive(Copy, Clone, Debug, bytemuck::Pod, bytemuck::Zeroable)]
pub struct ModelVertex {
    position: [f32; 3],
    tex_coords: [f32; 2],
    normal: [f32; 3],
    tangent: [f32; 3],
    bitangent: [f32; 3],
}
```

Esta estructura funciona perfectamente cuando se usa como búfer de vértices. Usarla como búfer de almacenamiento resultó ser menos conveniente. Mi código anterior usaba una estructura GLSL similar a mi `ModelVertex`.

```glsl
struct ModelVertex {
    vec3 position;
    vec2 tex_coords;
    vec3 normal;
    vec3 tangent;
    vec3 bitangent;
};
```

A primera vista, esto parece estar bien, pero los expertos en OpenGL probablemente verían un problema con la estructura. Nuestros campos no están alineados correctamente para soportar la alineación `std430` que requieren los búferes de almacenamiento. No entraré en detalle, pero puedes revisar la [demostración de alineación](../alignment) si quieres saber más. Para resumir, el `vec2` para `tex_coords` estaba arruinando la alineación de bytes, corrompiendo los datos de vértices resultando en lo siguiente:

![./corruption.png](./corruption.png)

Podría haber arreglado esto agregando un campo de relleno después de `tex_coords` en el lado de Rust, pero eso requeriría modificar el `VertexBufferLayout`. Terminé resolviendo este problema utilizando los componentes de los vectores directamente, lo que resultó en una estructura como esta:

```glsl
struct ModelVertex {
    float x; float y; float z;
    float uv; float uw;
    float nx; float ny; float nz;
    float tx; float ty; float tz;
    float bx; float by; float bz;
};
```

Dado que `std430` usará la alineación del elemento más grande de la estructura, usar todos los flotantes significa que la estructura se alineará a 4 bytes. Esta alineación coincide con la que `ModelVertex` usa en Rust. Fue un poco incómodo trabajar con esto, pero arregló el problema de corrupción.

El segundo problema me requirió replantear cómo estaba calculando la tangente y bitangente. El algoritmo anterior que estaba usando solo calculaba la tangente y bitangente para cada triángulo y configuraba todos los vértices en ese triángulo para usar la misma tangente y bitangente. Aunque esto está bien en un contexto de un solo hilo, el código se desmorona cuando se intenta calcular los triángulos en paralelo. La razón es que múltiples triángulos pueden compartir los mismos vértices. Esto significa que cuando vamos a guardar las tangentes resultantes, inevitablemente terminamos intentando escribir en el mismo vértice desde múltiples hilos diferentes, lo cual no está permitido. Puedes ver el problema con este método a continuación:

![./black_triangles.png](./black_triangles.png)

Esos triángulos negros fueron el resultado de múltiples hilos de GPU intentando modificar los mismos vértices. Observando los datos en Render Doc pude ver que las tangentes y bitangentes eran números basura como `NaN`.

![./render_doc_output.png](./render_doc_output.png)

Mientras que en la CPU podríamos introducir una primitiva de sincronización como un `Mutex` para arreglar este problema, que sepa no existe realmente algo así en la GPU. En su lugar, decidí modificar mi código para trabajar con cada vértice individualmente. Hay algunos obstáculos con eso, pero serán más fáciles de explicar en código. Comencemos con la función `main`.

```glsl
void main() {
    uint vertexIndex = gl_GlobalInvocationID.x;
    ModelVertex result = calcTangentBitangent(vertexIndex);
    dstVertices[vertexIndex] = result;
}
```

Usamos `gl_GlobalInvocationID.x` para obtener el índice del vértice para el cual queremos calcular las tangentes. Opté por poner el cálculo actual en su propio método. Veamos eso.

```glsl
ModelVertex calcTangentBitangent(uint vertexIndex) {
    ModelVertex v = srcVertices[vertexIndex];

    vec3 tangent = vec3(0);
    vec3 bitangent = vec3(0);
    uint trianglesIncluded = 0;

    // Find the triangles that use v
    //  * Loop over every triangle (i + 3)
    for (uint i = 0; i < numIndices; i += 3) {
        uint index0 = indices[i];
        uint index1 = indices[i+1];
        uint index2 = indices[i+2];

        // Only perform the calculation if one of the indices
        // matches our vertexIndex
        if (index0 == vertexIndex || index1 == vertexIndex || index2 == vertexIndex) {
            ModelVertex v0 = srcVertices[index0];
            ModelVertex v1 = srcVertices[index1];
            ModelVertex v2 = srcVertices[index2];

            vec3 pos0 = getPos(v0);
            vec3 pos1 = getPos(v1);
            vec3 pos2 = getPos(v2);

            vec2 uv0 = getUV(v0);
            vec2 uv1 = getUV(v1);
            vec2 uv2 = getUV(v2);

            vec3 delta_pos1 = pos1 - pos0;
            vec3 delta_pos2 = pos2 - pos0;

            vec2 delta_uv1 = uv1 - uv0;
            vec2 delta_uv2 = uv2 - uv0;

            float r = 1.0 / (delta_uv1.x * delta_uv2.y - delta_uv1.y * delta_uv2.x);
            tangent += (delta_pos1 * delta_uv2.y - delta_pos2 * delta_uv1.y) * r;
            bitangent += (delta_pos2 * delta_uv1.x - delta_pos1 * delta_uv2.x) * r;
            trianglesIncluded += 1;
        }

    }

    // Average the tangent and bitangents
    if (trianglesIncluded > 0) {
        tangent /= trianglesIncluded;
        bitangent /= trianglesIncluded;
        tangent = normalize(tangent);
        bitangent = normalize(bitangent);
    }

    // Save the results
    v.tx = tangent.x;
    v.ty = tangent.y;
    v.tz = tangent.z;
    v.bx = bitangent.x;
    v.by = bitangent.y;
    v.bz = bitangent.z;

    return v;
}
```

## Posibles Mejoras

Hacer un bucle sobre cada triángulo para cada vértice probablemente levante algunas banderas rojas para algunos de ustedes. En un contexto de un solo hilo, este algoritmo terminaría siendo O(N*M). Como estamos utilizando el alto número de hilos disponibles en nuestra GPU, esto es menos problemático, pero aún significa que nuestra GPU está quemando más ciclos de los que necesita.

Una forma en que se me ocurrió posiblemente mejorar el rendimiento es almacenar el índice de cada triángulo en una estructura como un mapa hash con el índice del vértice como claves. Aquí hay un pseudocódigo:

```rust
for t in 0..indices.len() / 3 {
    triangle_map[indices[t * 3]].push(t);
    triangle_map.push((indices[t * 3 + 1], t);
    triangle_map.push((indices[t * 3 + 2], t);
}
```

Luego necesitaríamos aplanar esta estructura para pasarla a la GPU. También necesitaríamos un segundo arreglo para indexar el primero.

```rust
for (i, (_v, t_list)) in triangle_map.iter().enumerate() {
    triangle_map_indices.push(TriangleMapIndex {
        start: i,
        len: t_list.len(),
    });
    flat_triangle_map.extend(t_list);
}
```

Finalmente decidí en contra de este método ya que era más complicado y no he tenido tiempo para comparar si es más rápido que el método simple.

## Resultados

Las tangentes y bitangentes ahora se calculan correctamente en la GPU.

![./results.png](./results.png)

<AutoGithubLink/>