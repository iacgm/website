---
date: '2025-10-01'
draft: false
title: 'C? Reescribelo it in Brainfuck.'
summary: "C2BF: Un Compilador de C a Brainfuck."
aliases: ["/posts/c2bf"]
---

Haciendo las cosas peor

La gente siempre parece querer hacer las cosas bien, y cuando fracasa, tiende a culpar a sus herramientas. Así que no debería sorprender que los programadores, siendo algo parecidos a las personas (y generalmente malos en lo que hacen), tengan una larga tradición de desarrollar un celo casi religioso por editores, paradigmas, estilos de código y, por supuesto, lenguajes de programación. Las discusiones nunca terminan, y lo que uno predica, otro [lo considera dañino](https://en.wikipedia.org/wiki/Considered_harmful).

- "_La tipificación estática solo estorba_", dice una voz, "_Python me deja simplemente programar._"

- "_El verificador de préstamos no es **tan** malo una vez que te acostumbras_", responde otro, cuya camiseta dice "_Reescríbelo en Rust_".

- "_Diviértete con tus bucles y estado mutable_", se burla un tercero, "_vuelve cuando aprendas lo que es una mónada._"

Es exactamente como Boris Marshalov describía al Congreso Estadounidense:
"Un hombre se levanta para hablar y no dice nada. Nadie escucha—y luego todos están en desacuerdo."

Pero derepente, milagrosamente, desde los cielos, oímos las voces sabias y atronadoras de [Church y Turing](https://es.wikipedia.org/wiki/Tesis_de_Church-Turing) al unísono, trayendo claridad en medio del ruido:
"Simplemente no importa", dicen, "En buenas manos, todos son equivalentes."

Pero, ¿y si no eres el tipo de persona que quiere hacer las cosas bien? ¿Y si eres del tipo que quiere hacerlas mal? ¿Y si quieres hacer las cosas **_peor_**?

En ese caso, podrías tener algo en común con Urban Müller, quien en 1993[^Bohm], presumiblemente incapaz de aceptar la doctrina de Church-Turing, creó **Brainfuck**, un lenguaje tan atroz que ningún otro nombre podría hacerle justicia. Con sólo 8 instrucciones y sin soporte para variables, funciones, memoria de acceso aleatorio, tipos de datos, aritmética fraccional, gestión de memoria ni ningún tipo de flujo de control convencional (!), seguramente _esta_ abominación no podría compararse con (digamos) C, la "lingua franca" del mundo digital, usada en casi todos los dispositivos de los últimos 50 años.

[^Bohm]: Con un poco de ayuda de [Bohm](https://es.wikipedia.org/wiki/P%E2%80%B2%E2%80%B2).

"Hello World" en Brainfuck, para los no iniciados, se ve así:

```brainfuck
++++++++[>++++[>++>+++>+++>+<<<<-]>+>+>->>+[<]<-]>>.>---.+++++++..+++.>>.<-.<.+++.------.--------.>>+.>++.
```

Durante los últimos 30 años, Brainfuck se ha convertido en una especie de broma recurrente entre programadores. Les gusta señalarlo y quedarse mirando, diciendo con misticismo y asombro:
"_Brainfuck puede parecer un desastre, pero es Turing Completo: puedes programar cualquier cosa que quieras en Brainfuck._"
Pero no puedes programar cualquier cosa que quieras en Brainfuck, porque Brainfuck es difícil, y tú no eres tan listo.

Pero **_yo_** sí.

**_Yo_** sí puedo hacer casi cualquier cosa que quiera en Brainfuck.
Por ejemplo, considera [esta creación reciente mía en Brainfuck](/donut.bf), que muestra una animación de una dona girando:

![ASCII Donut](/donut.png#center)

Si esto te resulta familiar, es porque es una versión (ligeramente modificada) del famoso programa en C de Andy Sloane, [donut.c](https://www.a1k0n.net/2011/07/20/donut-math.html), traducido a Brainfuck.

No, no escribí yo mismo las 125,489 instrucciones de esto. Sino, pasé 6 semanas creando [un compilador de C a Brainfuck](https://github.com/iacgm/c2bf), y luego modifiqué donut.c para que utilice aritmética de punto fijo de muy baja precisión.

Mi compilador admite casi todas las características principales de C, incluyendo:

- Aritmética de enteros
- Variables locales y globales
- Bucles y sentencias if
- Arreglos (arrays)
- Funciones (y recursión)
- Punteros
- Punteros a función (!)

He aquí cómo lo hice.

## Semántica de Brainfuck

Antes de analizar el compilador en sí, vale la pena repasar la semántica de Brainfuck.

Comenzamos con una cinta infinita de celdas, cada una conteniendo el número cero. Tenemos un cabezal de cinta que apunta a la primera de estas celdas. Además, contamos con ocho instrucciones, cada una con su propio símbolo:

1. `+`: Incrementa la celda bajo el cabezal de cinta.
2. `-`: Decrementa la celda bajo el cabezal de cinta.
3. `>`: Mueve el cabezal de cinta a la derecha.
4. `<`: Mueve el cabezal de cinta a la izquierda.
5. `,`: Almacena el siguiente carácter de la entrada en la celda bajo el cabezal de cinta.
6. `.`: Imprime el valor almacenado en la celda bajo el cabezal de cinta.
7. `[`: Si el cabezal de cinta apunta a un cero, salta al `]` correspondiente.
8. `]`: Si el cabezal de cinta **no** apunta a un cero, salta al `[` correspondiente.

Esencialmente, los bloques `[`/`]` funcionan como bucles `while-no-cero`.

Eso es todo. En serio.

## Introducción a los Compiladores

Este compilador no es realmente muy diferente de cualquier otro, por lo que resulta una buena base para aprender los fundamentos de los compiladores en general. Todo lo que hace un compilador es traducir un lenguaje a otro, con algunos pasos intermedios. Esto se realiza en una serie de pasos de traducción, cada uno de los cuales nos lleva un poco más lejos en el camino desde el código fuente hasta el lenguaje objetivo. En mi compilador, las etapas son:

1. **Análisis Léxico**: Este paso convierte un flujo de caracteres en tokens. Es decir, divide nuestro código en palabras clave, identificadores, símbolos, etc.
2. **Análisis Sintáctico**: Este paso convierte un flujo de tokens en un Árbol de Sintaxis Abstracta (AST). Es decir, divide nuestro código en construcciones: expresiones, declaraciones, definiciones, etc.
3. **Generación de Código Intermedio**: En esta etapa, convertimos nuestro AST en una Representación Intermedia (IR). Es decir, un lenguaje que cierra la brecha entre C y Brainfuck. Debería ser algo fácil de traducir tanto hacia como desde este.
4. **Generación de Código BF**: En esta etapa, traducimos cada instrucción IR a Brainfuck.
5. **Optimización Local**: Este paso no es estrictamente necesario, pero para lograr velocidades de ejecución razonables, necesitaremos que nuestro intérprete de Brainfuck reconozca ciertos patrones comunes en Brainfuck que pueden acelerarse o incluso omitirse por completo.

## Análisis Léxico y Sintáctico

Este es, más o menos, un problema resuelto. Esta es la única parte de este proyecto en la que utilicé una biblioteca externa, Pest. Esto nos permite especificar nuestra gramática (en este caso, [una gramática C99](https://www.quut.com/c/ANSI-C-grammar-l-1999.html) ligeramente modificada[^C99]) en algo más parecido a [la Forma de Backus-Naur](https://es.wikipedia.org/wiki/Notaci%C3%B3n_de_Backus-Naur). Por ejemplo:

[^C99]: Recomendaría encarecidamente a cualquiera que no lo haya hecho que examine detenidamente la gramática de C, o que al menos consulte la referencia de Microsoft sobre C (o el Estándar de C, aunque esto no es para los débiles de corazón). C está lleno de peculiaridades interesantes que de otro modo podrías nunca encontrar, y si hay algo que construir un compilador te obliga a hacer, es a aprender tu lenguaje de verdad, de manera profunda.

```pest
if_stmt = { "if" ~ "(" ~ expr ~ ")" ~ stmt ~ ("else" ~ stmt)? }
```

Una vez que Pest ha construido un árbol de análisis, podemos construir nuestro AST directamente. Por ejemplo, nuestra estructura de declaración if se ve así:

```rust
enum Stmt {
  IfStmt(Expr, Box<Stmt>, Option<Box<Stmt>>),
  // ... otros tipos de declaraciones
}
```

Lo mismo puede hacerse para otras construcciones, esencialmente de manera mecánica.

### Generación de IR

Aquí es donde ocurre la verdadera magia. Pero antes de analizar el proceso de generación de código, es importante considerar cómo se verá el IR. Nuestro lenguaje IR debe tener en cuenta que Brainfuck no tiene memoria de acceso aleatorio, por lo que cada decisión sobre dónde almacenamos cada valor creará obstáculos tangibles y difíciles al traducir a Brainfuck.

Esto es un rompecabezas que te animo a considerar. Encontrar una solución para este problema fue lo que me inspiró a trabajar en este proyecto.

La idea es utilizar un IR basado en [pila](https://es.wikipedia.org/wiki/Pila_(inform%C3%A1tica)). Esto significa que nuestras instrucciones no tomarán argumentos, sino que operarán sobre una única pila. Así, al traducir a Brainfuck, (en su mayoría) no tendremos que recuperar argumentos de la memoria.

Por ejemplo, consideremos la sentencia:

```c
putchar('0' + 3 * 2);
```

Usando un IR basado en registros, obtendríamos algo como:

```asm
mov r0, 48     ; 48 es '0' en ASCII
mov r1, 3
mov r2, 2
mul r1, r1, r2 ; multiplica r1 y r2
add r0, r0, r1 ; suma r1 a r0
putchar r0     ; ... según la convención de E/S
```

Pero nuestro IR basado en pila no tiene argumentos (no constantes). En su lugar, cada argumento actúa sobre los resultados de los anteriores:

```asm
Push(48) ; la pila queda [48]
Push(3)  ; la pila queda [48, 3]
Push(2)  ; la pila queda [48, 3, 2]
Mul      ; la pila queda [48, 6]
Add      ; la pila queda [54]
PutChar  ; la pila queda []
```

Continuando con nuestro ejemplo de la sentencia `if`, el nodo AST `If(cond, then, else)` se convertiría en algo como:

```ir
; [IR para `cond`]
Branch(L1, L2) ; Salta a L1 si cond es verdadero, a L2 en caso contrario
Label(L1)
; [IR para `then`]
GoTo(L3)
Label(L2)
; [IR para `else`]
Label(L3)
```

Nótese que, aunque estas instrucciones toman argumentos, todos son constantes. Nunca tenemos que leer memoria aparte de los valores en la cima de la pila.

### Traducción a Brainfuck

El IR basado en pila es especialmente conveniente porque la semántica de Brainfuck básicamente nos proporciona una pila en forma de cinta. Podemos pensar que el cabezal de la cinta apunta a la cima de la pila, y las celdas a su izquierda contienen los valores de la pila. Debemos mantener las celdas a la derecha del cabezal vacías para usarlas según sea necesario. Con este enfoque, traducir nuestras primeras instrucciones es pan comido. Por ejemplo:

1. `Push(5)`: `>+++++` 
  Simplemente movemos el cabezal e incrementamos la nueva celda 5 veces.

2. `Add`: `[-<+>]<` 
  Un poco más complicado, pero no demasiado. Si `x` e `y` son nuestros argumentos, decrementamos `x` e incrementamos `y` repetidamente hasta que `x` sea cero. Luego, movemos el cabezal atrás una celda para liberar espacio en la pila.

3. `Store(3)`: `<<<[-]>>>[-<<<+>>>]<` 
  Para mover un valor de la cima de la pila a una posición más abajo, limpiamos esa ranura, sumamos el valor de la cima y movemos el cabezal de vuelta.

4. `GoTo`: Mmm... Aunque las operaciones aritméticas básicas pueden implementarse con traducciones directas[^translations], implementar los saltos necesarios para el flujo de control general (o para punteros a función) es más complicado.

[^translations]: "Directo" no significa "simple"; intenta implementar operaciones bit a bit sin multiplicación o división. Incluso las comparaciones de números sin signo son altamente complejas. Si te interesa, puedes consultar [estos recursos](https://esolangs.org/wiki/Brainfuck_algorithms) como inspiración.

Nuevamente, te animo a detenerte y reflexionar sobre esto. Es otro rompecabezas interesante.

Esta vez, la idea es usar una transformación de programa completo. Podemos envolver todo el programa en un bucle `[`/`]` y luego controlar cada bloque de código ensamblador con una sentencia condicional. De esta manera, solo necesitamos implementar un tipo simple de flujo de control en Brainfuck. Por ejemplo:

```c
a:
  // Bloque A
  goto c;
b:
  // Bloque B
  exit();
c:
  // Bloque C
  goto b;
```

... puede reescribirse como ...

```c
int etiqueta = 'a';
while (etiqueta != 0) {
  if (etiqueta == 'a') {
    // Bloque A
    etiqueta = 'c';
  }
  if (etiqueta == 'b') {
    // Bloque B
    etiqueta = 0;
  }
  if (etiqueta == 'c') {
    // Bloque C
    etiqueta = 'b';
  }
}
```

Implementar verificaciones de igualdad, bucles while-no-cero y sentencias condicionales en Brainfuck son problemas, si no fáciles, al menos manejables.

En cuanto a las lecturas y escrituras de punteros, no intentaré explicarlas, más allá de enlazar este excelente artículo.

### Optimización Local

Aunque los pasos anteriores nos permiten compilar programas Brainfuck funcionales, no están cerca de ser óptimos. Solo mirando un fragmento de nuestro código donut.bf, vemos todo tipo de código subóptimo:

```brainfuck
+<[[-]>-<]>[-<+>]<[-<[-]<>>+>+++++++++++++++[-<[->>+>+<<<]>>[-<<
+>>]>[-<<<+>>>]<<]<<<[->>>+>+<<<<]>>>>[-<<<<+>>>>]<>>[-]+>[-]+>[
-]+>[-]+>[-]+>[-]+>[-]+>[-]+>[-]+>[-]+>[-]+>[-]+>[-]+>[-]+>[-]+>
[-]+>[-]<[<]<[->>[>]<[--[++++[->]>]++<]>--<<[<]<]<[->+<]>>>[>]+>
[-]+>[-]+>[-]+>[-]+>[-]+>[-]+>[-]+>[-]+>[-]+>[-]+>[-]+>[-]+>[-]+
>[-]+>[-]+>[-]<[<]<[->>[>]<[--[++++[->]>]++<]>--<<[<]<]>>[>]++++
++++++++++++[-<[-<<<<<<<<<<<<<<<<+>>>>>>>>>>>>>>>>]>[-<+>]<]<[<]
<+>>[>]<[>+<----[[-]>-<]>[-<<[<]<[-<+>>+<]>[-<+>]>[>]>]<<[<]<[->
++<]>[-<+>]>>[>]<]<<<<[-]>[-<+>]<[->+>+<<]>>[-<<+>>]<>++++++++++
```

Por todas partes vemos secuencias como `<>` o `><`, que pueden eliminarse por completo. Peor aún, también encontramos código lento _pero óptimo_, como largas cadenas de instrucciones `>/<` y `+/-`, secciones `[-]` y bloques de "movimiento" (es decir, bloques que limpian una celda y suman o restan su valor de otras celdas, como `[-<+>]` o `[->>+>+<<<]`).

Estos patrones no son difíciles de detectar, y hacerlo nos permite saltarnos y ejecutar grandes bloques de código de una sola vez. Esto acelera masivamente nuestro código. Por ejemplo, sumar 100 a 100 normalmente tomaría about 600 instrucciones, pero se convierte en una sola instrucción.

Intenté renderizar un solo fotograma de donut.c sin estas optimizaciones, pero abandoné después de about 20 horas porque mi portátil comenzó a sobrecalentarse tanto que la pantalla dejó de responder. Un cálculo rápido mostraba que podría haber esperado about 6 horas más en mi máquina.

Con estas optimizaciones, podemos renderizar un fotograma en about 90 minutos.

Podemos llevar esto aún más lejos, porque un análisis de rendimiento revela que solo un pequeño número de instrucciones complejas consumen casi todo nuestro tiempo de ejecución. Después de usar un truco similar para detectar el código que genero para instrucciones complejas como operaciones bit a bit, multiplicaciones, divisiones y accesos a memoria (efectivamente descompilándolas de vuelta a IR en tiempo de ejecución), puedo renderizar un fotograma en about 12 minutos[^caveat].

Sí, este último truco es básicamente hacer trampa. 
No, realmente no me importa.

[^caveat]: Para poder hacer esto de forma segura, debemos asegurarnos de que siempre que veamos el código para una operación bit a bit (por ejemplo), realmente solo calcule un valor, sin efectos secundarios. Esto significa que debemos estar seguros de que todas las celdas de memoria que ese fragmento de código podría leer tienen los valores que esperamos. Podemos lograr esto haciendo que estas instrucciones limpien toda la memoria que necesiten antes de utilizar cualquier parte de ella.

## Limitaciones

Faltan algunas características, pero en su mayoría no serían difíciles de agregar. El factor limitante fue que, en este punto, el proyecto pasó de ser divertido y educativo a consumir mucho tiempo y volverse tedioso.

Algunas características importantes que faltan son:

- Soporte para funciones variádicas (incluyendo `printf`).
- Funciones de biblioteca estándar en general.
- Tipos de diferentes tamaños (aparte de arreglos). La mayoría de los intérpretes de Brainfuck usan celdas de 1 byte. El mío usa celdas de 2 bytes para que sean lo suficientemente grandes como para almacenar las etiquetas a las que saltar, y para tener suficiente precisión para implementar una aritmética de punto fijo utilizable.
- Sentencias `switch`. Sin excusas aquí, simplemente me volví un poco perezoso.
- Asignación de memoria / arreglos de longitud variable. Esta sería una adición complicada, ya que tendríamos que cambiar cómo accedemos a las variables globales, pero no estaría fuera de nuestro alcance.

## Proyectos Similares

Después de completar este proyecto e investigar un poco, descubrí que la idea para este proyecto, como cualquier buena idea, ha sido concebida varias veces antes. Sin embargo, aunque definitivamente elogio a cualquiera que asuma el importante trabajo de construir un compilador de X-a-Brainfuck, creo que mi enfoque distingue a este proyecto de otros.

Existen varios proyectos (como el descrito [aquí](https://www.quut.com/c/ANSI-C-grammar-l-1999.html)) que pueden traducir comandos simples e incluso bucles, pero no admiten cosas como llamadas a funciones. El artículo enlazado incluso llega a describir el truco de salto que usé, pero dice que sería demasiado difícil de implementar.

Los únicos que he encontrado que son tan completos como el mío son aquellos que funcionan, pero se parecen más a emuladores escritos en Brainfuck que a transpiladores. Es decir, en lugar de traducir el código directamente, actúan como máquinas virtuales, colocando instrucciones en memoria y luego ejecutando cada una independientemente, simulando memoria tradicional y registros. [ELVM](https://github.com/shinh/elvm) es un proyecto que se dirige a muchos lenguajes esotéricos, pero la implementación de BF es esencialmente un emulador. Gregor Richards tiene [otro proyecto](https://esolangs.org/wiki/C2BF) que es explícitamente un emulador.

No hay nada malo con los emuladores, pero siento que mi enfoque ofrece una traducción más "auténtica" (lo que sea que eso signifique), y la IR basada en pila es realmente el ingrediente secreto para eso. Probablemente también es mucho más rápido, ya que solo necesitamos ejecutar realmente cada instrucción, en lugar de cargar instrucciones desde memoria y lidiar con contadores de programa y registros, etc. Sin embargo, no los comparé, porque mi implementación está escrita en Rust, así que no importa lo lenta que sea, sigue siendo extremadamente rápida.

Así que adelante, prospera y pon Brainfuck en producción ;)
