# Ordenamiento en la GPU

Trabajar con datos ordenados hace que la mayoría de los algoritmos sean más fáciles de usar, por lo
que tiene sentido que queramos poder ordenar nuestros datos de GPU en
la GPU. Tenemos que repensar cómo abordamos el ordenamiento ya que la forma en que lo hacemos
no es la misma que con los algoritmos de ordenamiento tradicionales, que no están diseñados con
el poder de la computación paralela en mente. ¡Afortunadamente hay algunos algoritmos que sí
funcionan bien con la GPU!

## Odd-Even Sort (aka. Brick Sort)

Este ordenamiento funciona iterando sobre pares de elementos, comparándolos e
intercambiándolos si uno es mayor que el otro. Considera el siguiente
arreglo:

```rust
[3, 7, 1, 5, 0, 4, 2, 6]
```

Primero hacemos el pase impar. Esto significa que consideramos pares de elementos desde
el índice 1 (no 0) en adelante. Entonces para los datos anteriores, los pares serían los siguientes.

```rust
[[7, 1], [5, 0], [4, 2]]
```

Saltamos el primer y último término ya que no tienen otro número
junto a ellos que no esté ya emparejado. Luego intercambiamos el número mayor
con el número menor, lo que significa que nuestros datos se ven así:

```rust
[3, 1, 7, 0, 5, 2, 4]
```

Lo siguiente es el pase par. Es lo mismo que el pase impar, pero empezamos en el índice
0 en lugar de 1. Esto nos da los siguientes pares.

```rust
[[3, 1], [7, 0], [2, 4]]
```

Lo que al intercambiar nos da lo siguiente:

```rust
[1, 3, 0, 7, 2, 4]
```

Este proceso se repite hasta que el arreglo esté ordenado. Aquí están los datos en cada
iteración después de esto:

```rust
[1, 0, 3, 2, 7, 4] // odd
[0, 1, 2, 3, 4, 7] // even
```

## ¿Cuándo dejamos de ordenar?

La mayoría de los algoritmos de ordenamiento no verifican manualmente que el arreglo esté ordenado
después de cada iteración. Afortunadamente [la investigación muestra](https://en.wikipedia.org/wiki/Odd%E2%80%93even_sort#cite_note-6)
que el número máximo de iteraciones para completar este algoritmo es el
mismo que el número de elementos, es decir, N. Esto significa que dado un arreglo de
tamaño `N = 8`

```rust
[7, 6, 5, 4, 3, 2, 1, 0]
```

Tomará 8 pases ordenar estos datos.

```rust
[7, 5, 6, 3, 4, 1, 2, 0] // odd
[5, 7, 3, 6, 1, 4, 0, 2] // even
[5, 3, 7, 1, 6, 0, 4, 2] // odd
[3, 5, 1, 7, 0, 6, 2, 4] // even
[3, 1, 5, 0, 7, 2, 6, 4] // odd
[1, 3, 0, 5, 2, 7, 4, 6] // even
[1, 0, 3, 2, 5, 4, 7, 6] // odd
[0, 1, 2, 3, 4, 5, 6, 7] // even
```

Esto siempre funcionará, independientemente de los datos que estemos ordenando. Es un
poco ineficiente cuando tus datos están casi ordenados, así que querrás
tener eso en mente.

## Portando odd-even sort a WGSL

El odd-even sort es especial porque cada pase es trivial de paralelizar.
Cada par de elementos se considera independientemente de todos los otros elementos.
Eso significa que podemos dedicar un solo hilo a cada par que queramos
comparar. ¡Veamos el sombreador!

```wgsl
@group(0)
@binding(0)
var<storage, read_write> data: array<u32>;

@compute
@workgroup_size(64, 1, 1)
fn odd_even_sort(
    @builtin(global_invocation_id)
    gid: vec3<u32>,
) {
    // ...
}
```

Esto funciona de manera muy similar al código de introducción. Configuramos nuestro bindgroup
con un `array<u32>` que es `read_write` ya que este ordenamiento funciona sin necesitar
un arreglo adicional para hacer la salida. También solo necesitamos el `global_invocation_id`
incorporado para indexar correctamente nuestro `data`. Ahora vamos a empezar a ver el código dentro
de `odd_even_sort`.

```wgsl
    let num_items = arrayLength(&data);
    let pair_index = gid.x;
```

Primero obtenemos el índice del par en el que el hilo actual está trabajando.

```wgsl
    // odd
    var a = pair_index * 2u + 1;
    var b = a + 1u;

    if a < num_items && b < num_items && data[a] > data[b] {
        let temp = data[a];
        data[a] = data[b];
        data[b] = temp;
    }
```

Para esta parte del código, primero obtenemos los índices de los elementos que queremos comparar.
Si los índices están dentro de los límites y los valores están fuera de orden, intercambiamos los valores. También podemos
hacer el pase par para que podamos reducir a la mitad el número de veces que llamamos a este sombreador.

```wgsl
    // even
    a = pair_index * 2u;
    b = a + 1u;

    if a < num_items && b < num_items && data[a] > data[b] {
        let temp = data[a];
        data[a] = data[b];
        data[b] = temp;
    }
```

Parece que hemos terminado con el código del sombreador, pero técnicamente
hay un error en nuestro código. No tiene nada que ver con la lógica de nuestro código, tiene
que ver con la naturaleza de la programación paralela en general: condiciones de carrera.

## Condiciones de carrera y barreras

![example of race conditions featuring puppies](./race-condition-puppies.jpg)

Una condición de carrera ocurre cuando dos o más hilos intentan operar en la
misma ubicación en memoria. Si hiciéramos un paso para cada llamada al sombreador,
estaríamos bien, pero como hacemos dos pases, necesitamos asegurarnos de que
los hilos no se interfieran entre sí. Hacemos esto usando barreras.

Una barrera hace que el hilo actual espere a que otros hilos terminen
antes de continuar. Hay dos tipos de barreras.

Un `workgroupBarrier` hará que todos los hilos en el grupo de trabajo esperen
hasta que todos los otros hilos en el grupo de trabajo hayan llegado a la barrera. También
sincronizará todas las variables atómicas y los datos almacenados en el espacio de
dirección del grupo de trabajo.

<div class="note">

En WGSL, "address space" (espacio de dirección) determina cómo se puede acceder a un cierto trozo de datos.
Los datos en el espacio de dirección `workgroup` solo son accesibles por hilos dentro del
mismo grupo de trabajo. Muchos de los espacios de dirección son implícitos, como el
espacio de dirección `function`. Los espacios de dirección `uniform` y `storage` son
significativos ya que corresponden a búferes uniformes y búferes de almacenamiento
respectivamente.

</div>

Un `storageBarrier` hará que la GPU sincronice todos los cambios en los búferes de almacenamiento.
Dado que nuestros datos están en un búfer de almacenamiento, esta es la barrera que necesitamos
para asegurar que todo se mantenga en sincronización. Agrega la siguiente línea entre los pases impar y
par.

```wgsl
    storageBarrier();
```

Con eso la función `odd_even_sort` se ve así:

```wgsl
@compute
@workgroup_size(64, 1, 1)
fn odd_even_sort(
    @builtin(global_invocation_id)
    gid: vec3<u32>,
) {
    let num_items = arrayLength(&data);
    let pair_index = gid.x;

    // odd
    var a = pair_index * 2u + 1;
    var b = a + 1u;

    if a < num_items && b < num_items && data[a] > data[b] {
        let temp = data[a];
        data[a] = data[b];
        data[b] = temp;
    }

    storageBarrier();

    // even
    a = pair_index * 2u;
    b = a + 1u;

    if a < num_items && b < num_items && data[a] > data[b] {
        let temp = data[a];
        data[a] = data[b];
        data[b] = temp;
    }
}
```

## Llamando al sombreador

La mayoría del código es igual al código de introducción, con la
excepción de crear solo un búfer de almacenamiento y el siguiente
código para llamar al sombreador:

```rust
    let num_items_per_workgroup = 128; // 64 threads, 2 items per thread
    let num_dispatches = (input_data.len() / num_items_per_workgroup) as u32
        + (input_data.len() % num_items_per_workgroup > 0) as u32;
    // We do 2 passes in the shader so we only need to do half the passes
    let num_passes = input_data.len().div_ceil(2);

    {
        let mut pass = encoder.begin_compute_pass(&Default::default());
        pass.set_pipeline(&pipeline);
        pass.set_bind_group(0, &bind_group, &[]);

        for _ in 0..num_passes {
            pass.dispatch_workgroups(num_dispatches, 1, 1);
        }
    }
```

Con esto, tus datos deberían estar ordenados. Ahora puedes usarlos para cualquier propósito
que necesites, como ordenar objetos transparentes por su coordenada z, u ordenar
objetos por la celda a la que pertenecen en una cuadrícula para detección y resolución de colisiones.
Usaremos ordenamiento para implementar algunos algoritmos diferentes en otras partes de
esta guía.

## Conclusión

El ordenamiento es uno de los pilares del desarrollo de software y ahora que podemos ordenar
nuestros datos de GPU sin enviarlos en un viaje de ida y vuelta a la GPU. Usaremos esto
mucho en el resto de esta guía.

¡Gracias por leer esto, y un agradecimiento especial a estos patrocinadores!

* Filip
* Lions Heart
* Jani Turkia
* Julius Liu
* 折登 樹
* Aron Granberg
* Ian Gowen
* Bernard Llanos
* David Laban
* IC

<WasmExample example="compute" noCanvas="true" autoLoad="true"></WasmExample>

<AutoGithubLink path="/compute/src/"/>
