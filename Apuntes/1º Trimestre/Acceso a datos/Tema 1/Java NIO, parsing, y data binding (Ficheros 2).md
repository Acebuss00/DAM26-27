Acceso a datos

# Ficheros en Java II

Acceso aleatorio con java.io y NIO · Intercambio de datos · Parsers · Data binding**Objetivos**

- comparar acceso secuencial y aleatorio sobre un fichero binario;
- comprender por qué el acceso directo exige conocer o calcular posiciones en bytes;
- resolver después ese mismo problema con la API clásica y con el modelo NIO y comparar ambos enfoques;
- diseñar registros de tamaño fijo y calcular la posición exacta de cada campo;
- leer un XML con DOM y SAX, crear un XML con DOM y consultar nodos con XPath;
- realizar data binding XML ↔ objetos Java con Jakarta XML Binding (JAXB);
- realizar data binding JSON ↔ objetos Java con Jackson.

## Contenido

1. [Acceso secuencial y aleatorio](#secuencial-aleatorio)
2. [RandomAccessFile](#randomaccessfile)
3. [NIO: FileChannel y ByteBuffer paso a paso](#filechannel)
4. [Posiciones y tamaños](#posiciones)
5. [Registros de tamaño fijo](#registros)
6. [Patrones con RandomAccessFile](#operaciones-random)
7. [Intercambio de datos](#intercambio)
8. [Formato, parser y data binding](#formato-parser-binding)
9. [Formatos de intercambio](#formatos)
10. [XML: repaso necesario](#xml-repaso)
11. [DOM](#dom)
12. [SAX](#sax)
13. [StAX](#stax)
14. [DOM vs SAX vs StAX](#comparativa-parsers)
15. [XPath](#xpath)
16. [Bien formado, válido y esquemas](#validacion)
17. [Data binding](#databinding)
18. [JAXB / Jakarta XML Binding](#jaxb)
19. [JSON y Jackson](#json-jackson)
20. [Otras tecnologías](#otros-formatos)
21. [Referencia rápida](#referencia)

<a id="secuencial-aleatorio"></a>

## 1. Antes de elegir una clase: acceso secuencial y acceso aleatorio

Lo primero es separar el **problema** de la **herramienta**. Secuencial y aleatorio describen *cómo queremos acceder a los datos*; todavía no dicen qué clase Java utilizaremos.

**Acceso secuencial**inicio ─► dato 0 ─► dato 1 ─► dato 2 ─► ...

Procesamos los datos siguiendo su orden. Si queremos llegar a un dato lejano, normalmente pasamos por los anteriores.

**Acceso aleatorio o directo**inicio ─────────────► posición conocida

Conocemos o calculamos una posición del fichero y empezamos a trabajar directamente desde ese byte.

Por ejemplo, si un fichero contiene únicamente enteros almacenados consecutivamente y cada entero ocupa 4 bytes, el elemento de índice 5 empieza en:

posición = 5 × 4 = byte 20**Analogía: un libro con páginas numeradas**

En un recorrido secuencial pasaríamos páginas hasta llegar a la 120. En acceso directo abrimos la página 120 porque conocemos su posición. Java no busca mágicamente por contenido: **la posición tiene que conocerse o calcularse**.

**Acceso aleatorio no significa «buscar por nombre».** Saltar al byte 20 es posible porque conocemos su dirección. Para encontrar «Ana» necesitamos recorrer los datos, conocer previamente su posición o disponer de una estructura de índice.

### Qué herramientas estudiaremos

Una vez entendido el problema, utilizaremos dos formas de resolverlo en Java:

1. **Primero `RandomAccessFile`**, de `java.io`. Es la forma más directa de aprender la idea de puntero y posición.
2. **Después `FileChannel` + `ByteBuffer`**, del modelo NIO. Antes de comparar código explicaremos qué es un canal, qué es un buffer y cómo cambia de estado.

Solo después de conocer ambos modelos los pondremos uno al lado del otro. Así la comparación servirá para comprender, no para memorizar métodos desconocidos.

## 2. `RandomAccessFile`

`RandomAccessFile` pertenece a `java.io` y permite leer y escribir en cualquier posición de un fichero. Mantiene un **puntero interno** que indica dónde empezará la siguiente operación.

```
try (RandomAccessFile fichero = new RandomAccessFile("datos.dat", "rw")) {
    // lectura y escritura
}
```

| Modo | Uso |
| --- | --- |
| `"r"` | Solo lectura. |
| `"rw"` | Lectura y escritura. |
| `"rwd"` | Lectura/escritura intentando sincronizar también el contenido escrito con el almacenamiento. |
| `"rws"` | Como `"rwd"`, incluyendo además cambios de metadatos asociados. |

### El puntero del fichero

```
fichero.seek(100);              // siguiente operación desde el byte 100
long posicion = fichero.getFilePointer();
long longitud = fichero.length();
```

`seek()` trabaja con **desplazamientos en bytes** y la primera posición es 0. Leer o escribir hace avanzar el puntero el número de bytes consumidos.

**Por qué puede leer tipos directamente:** `RandomAccessFile` implementa las interfaces `DataInput` y `DataOutput`. Por eso, además de bytes, ofrece operaciones tipadas como `readInt()`, `writeLong()`, `readDouble()` o `writeChar()`.

### Primer acceso completo: leer un entero concreto

Si el fichero contiene enteros consecutivos y queremos el de índice 5, primero calculamos su byte inicial y después movemos el puntero:

```
long indice = 5;
long posicion = indice * (long) Integer.BYTES; // 5 × 4 = byte 20

fichero.seek(posicion);                        // saltamos directamente al byte 20
int valor = fichero.readInt();                 // leemos los 4 bytes del int
```

Este ejemplo resume el modelo de `RandomAccessFile`: **calcular posición → mover puntero → leer o escribir un tipo**.

<a id="filechannel"></a>

## 3. Acceso aleatorio con NIO: construir el modelo antes de comparar código

Ya conocemos una solución: `RandomAccessFile` mantiene un puntero, lo movemos a una posición y después leemos o escribimos un tipo. NIO resuelve el mismo problema con otro modelo. Antes de ver código completo necesitamos entender sus dos piezas.

### 3.1. Primera pieza: el canal

Un **canal** es una conexión por la que pueden transferirse datos. Cuando esa conexión está asociada a un fichero utilizamos `FileChannel`. El canal puede leer bytes del fichero, escribir bytes en él y mantener una posición dentro del fichero.

fichero ⇄ FileChannel

Para abrirlo usaremos una ruta `Path` de NIO.2. Las constantes de `StandardOpenOption` indican qué operaciones permitimos:

| Opción | Significado |
| --- | --- |
| `READ` | Permitimos leer. |
| `WRITE` | Permitimos escribir. |
| `CREATE` | Crea el fichero si no existe. |
| `TRUNCATE_EXISTING` | Vacía un fichero existente al abrirlo para escritura. |

```
// NIO.2: Path representa dónde está el fichero.
Path ruta = Path.of("datos.bin");

// NIO: FileChannel abre una conexión con ese fichero.
try (FileChannel canal = FileChannel.open(
        ruta,
        StandardOpenOption.READ,
        StandardOpenOption.WRITE)) {

    // A partir de aquí podemos mover bytes.
}
```

**Aquí se mezclan NIO y NIO.2 de forma normal.** `Path` y las opciones de apertura pertenecen a la API moderna de sistema de archivos; `FileChannel` pertenece al modelo de canales. No son tecnologías incompatibles: están diseñadas para trabajar juntas.

### 3.2. Segunda pieza: el buffer

El canal no nos entrega directamente un `int` o un `double`. Transfiere **bytes**. Necesitamos una zona de memoria donde recibirlos o desde la que enviarlos: un `ByteBuffer`.

fichero ⇄ FileChannel ⇄ ByteBuffer ⇄ programa

Si queremos leer exactamente un `int`, necesitamos espacio para sus 4 bytes:

```
ByteBuffer buffer = ByteBuffer.allocate(Integer.BYTES); // 4 bytes
```

El buffer mantiene tres valores básicos:

| Valor | Pregunta que responde | Al hacer `allocate(4)` |
| --- | --- | --- |
| `capacity` | ¿Cuánto cabe como máximo? | 4 |
| `position` | ¿Dónde está la siguiente operación? | 0 |
| `limit` | ¿Hasta dónde se puede trabajar en esta fase? | 4 |

recién creado:  
[ \_ \_ \_ \_ ]  
↑ position = 0 limit = 4 ↑ capacity = 4

### 3.3. El cambio de fase: primero llenamos, después consumimos

Supongamos que el canal ha cargado los cuatro bytes de un entero dentro del buffer. Al escribir bytes en el buffer, `position` avanza hasta 4:

después de recibir 4 bytes:  
[ B0 B1 B2 B3 ]  
 ↑ position = 4

Ahora queremos leer esos mismos bytes desde el principio. Para cambiar el buffer de la fase de **llenado** a la fase de **consumo** hacemos:

```
buffer.flip();
```

Después de `flip()`, el límite se coloca donde terminó el llenado y la posición vuelve al principio:

[ B0 B1 B2 B3 ]  
↑ position = 0 limit = 4 ↑

Ahora sí podemos interpretar los cuatro bytes como un entero:

```
int valor = buffer.getInt();
```

**Analogía de la bandeja**

Mientras el canal llena la bandeja, el marcador avanza dejando sitio para más datos. Cuando decimos `flip()` estamos diciendo: «ya no voy a meter más; ahora quiero empezar a sacar lo que he recibido».

### 3.4. Leer el entero de índice 5 paso a paso

Ahora que conocemos las piezas, podemos montar la operación completa. Supongamos un fichero formado solo por enteros de 4 bytes.

```
long indice = 5;
long posicion = indice * (long) Integer.BYTES; // 5 × 4 = byte 20

// 1. Reservamos exactamente los 4 bytes que ocupa un int.
ByteBuffer buffer = ByteBuffer.allocate(Integer.BYTES);

// 2. Situamos el canal en el byte calculado.
canal.position(posicion);

// 3. Llenamos el buffer. Repetimos porque una lectura puede ser parcial.
while (buffer.hasRemaining()) {
    int leidos = canal.read(buffer);

    if (leidos == -1) {
        throw new EOFException("El fichero termina antes de completar el int");
    }
}

// 4. Dejamos de llenar y preparamos el buffer para consumirlo.
buffer.flip();

// 5. Interpretamos los cuatro bytes como un int.
int valor = buffer.getInt();
```

| Momento | position | limit | Qué está pasando |
| --- | --- | --- | --- |
| `allocate(4)` | 0 | 4 | Hay cuatro huecos libres. |
| tras leer 4 bytes | 4 | 4 | El buffer está lleno. |
| `flip()` | 0 | 4 | Empezamos a consumir desde el principio. |
| `getInt()` | 4 | 4 | Los cuatro bytes ya se han interpretado. |

### 3.5. Ahora sí: comparar la misma lectura

| `RandomAccessFile` | `FileChannel` + `ByteBuffer` |
| --- | --- |
| ``` long posicion = indice * (long) Integer.BYTES;  fichero.seek(posicion); int valor = fichero.readInt(); ``` | ``` long posicion = indice * (long) Integer.BYTES;  ByteBuffer buffer = ByteBuffer.allocate(Integer.BYTES); canal.position(posicion);  while (buffer.hasRemaining()) {     if (canal.read(buffer) == -1) {         throw new EOFException();     } }  buffer.flip(); int valor = buffer.getInt(); ``` |

La fórmula y la posición física son exactamente las mismas. Lo que cambia es **quién interpreta los bytes**: `RandomAccessFile` ofrece directamente `readInt()`; en NIO los bytes pasan por el buffer y `ByteBuffer.getInt()` los interpreta.

### 3.6. Escribir un entero con NIO

Para escribir ocurre el recorrido inverso: primero colocamos el valor en el buffer y después el canal consume sus bytes.

```
ByteBuffer buffer = ByteBuffer.allocate(Integer.BYTES);

// 1. El programa coloca el int dentro de la bandeja.
buffer.putInt(25);

// 2. Cambiamos de "llenar el buffer" a "dejar que el canal lo consuma".
buffer.flip();

// 3. Elegimos el punto del fichero donde escribir.
canal.position(posicion);

// 4. El canal vacía el contenido válido del buffer en el fichero.
while (buffer.hasRemaining()) {
    canal.write(buffer);
}
```

lectura: fichero ─► canal ─► buffer ─► programa  
escritura: programa ─► buffer ─► canal ─► fichero

### 3.7. Dos formas de indicar la posición con `FileChannel`

Una vez entendido el canal, podemos ver una capacidad que no tenía sentido presentar antes. Podemos cambiar su posición actual:

```
canal.position(100);
canal.read(buffer);
```

O indicar una posición directamente en la operación:

```
canal.read(buffer, 100);   // lee desde el byte 100 sin cambiar la posición actual del canal
canal.write(buffer, 100);  // escribe desde el byte 100 sin cambiarla
```

**Una operación puede ser parcial.** Una llamada a `read()` o `write()` no debe interpretarse como «siempre transfiere todo». Cuando necesitamos completar una estructura de tamaño conocido comprobamos lo que queda en el buffer y repetimos si es necesario.

### 3.8. Comparación final, ahora que conocemos los dos modelos

| Aspecto | `RandomAccessFile` | `FileChannel` |
| --- | --- | --- |
| Familia | `java.io` | NIO |
| Modelo mental | Puntero + métodos tipados | Canal + buffer de bytes |
| Cambiar posición | `seek(pos)` | `position(pos)` |
| Leer un `int` | `readInt()` | `read()` → `flip()` → `getInt()` |
| Operación posicional sin cambiar el marcador | No tiene un método equivalente directo | `read(buffer, pos)` / `write(buffer, pos)` |

**Orden de bytes:** tanto los métodos tipados de `RandomAccessFile` como `ByteBuffer` utilizan por defecto orden *big-endian*. Solo necesitamos cambiarlo si el formato externo que estamos leyendo especifica otro orden.

<a id="posiciones"></a>

## 4. La fórmula fundamental del acceso aleatorio

La fórmula no depende de que utilicemos `RandomAccessFile` o `FileChannel`. Si todos los elementos tienen el mismo tamaño, la posición de un elemento se calcula multiplicando su **índice lógico** por el número de bytes que ocupa.

posición física = índice lógico × tamaño del elemento

| Tipo | Bytes | Posición del elemento *i* |
| --- | --- | --- |
| `byte` | 1 | `i * 1L` |
| `boolean` con `writeBoolean()` | 1 | `i * 1L` |
| `short` / `char` | 2 | `i * 2L` |
| `int` / `float` | 4 | `i * 4L` |
| `long` / `double` | 8 | `i * 8L` |

Java ofrece constantes que evitan utilizar números mágicos:

```
Integer.BYTES     // 4
Double.BYTES      // 8
Long.BYTES        // 8
Character.BYTES   // 2
```

### Ejemplo con `int`

```
long indice = 5;
long posicion = indice * Integer.BYTES;  // 20

fichero.seek(posicion);
int valor = fichero.readInt();
```

### El mismo índice con `double`

```
long indice = 5;
long posicion = indice * Double.BYTES;   // 40

fichero.seek(posicion);
double valor = fichero.readDouble();
```

**Atención con las cadenas:** `writeUTF()` escribe primero un prefijo de 2 bytes con la longitud y después texto en *modified UTF-8*, por lo que el número de bytes depende del contenido. Si cada registro puede ocupar un tamaño distinto, ya no podemos localizar el registro *i* únicamente con `i × tamaño`.

<a id="registros"></a>

## 5. Registros de tamaño fijo

Un fichero de acceso aleatorio resulta especialmente sencillo cuando todos los registros ocupan exactamente lo mismo. Entonces no necesitamos guardar la posición de cada registro: podemos **calcularla**.

| id: 4 bytes | nombre: 40 bytes | notaMedia: 8 bytes | = 52 bytes

| Campo | Representación | Tamaño |
| --- | --- | --- |
| `id` | `int` | 4 bytes |
| `nombre` | 20 caracteres fijos escritos con `writeChar()` | 40 bytes |
| `notaMedia` | `double` | 8 bytes |
| Tamaño de registro | | 52 bytes |

**¿Por qué 20 caracteres ocupan 40 bytes aquí?** Porque este formato propio escribe cada unidad `char` con `writeChar()`, que utiliza 2 bytes. No estamos almacenando el nombre como texto UTF-8. El tamaño fijo es una decisión del formato del fichero.

```
static final int NUM_CARACTERES_NOMBRE = 20;
static final int TAM_NOMBRE = NUM_CARACTERES_NOMBRE * Character.BYTES;
static final int TAM_REGISTRO = Integer.BYTES + TAM_NOMBRE + Double.BYTES;

long posicionRegistro = indice * (long) TAM_REGISTRO;
```

**El formato no pertenece a `RandomAccessFile`.**

Estos 52 bytes describen el *layout* que hemos decidido para cada registro. Podemos escribirlos con `RandomAccessFile` o con `FileChannel` + `ByteBuffer`; lo importante es respetar el mismo tamaño, orden de campos y orden de bytes.

### Escribir y leer una cadena de longitud fija

```
static void escribirCadenaFija(RandomAccessFile fichero, String texto,
                               int longitud) throws IOException {
    String ajustado = String.format("%-" + longitud + "s", texto);
    ajustado = ajustado.substring(0, longitud);

    for (int i = 0; i < longitud; i++) {
        fichero.writeChar(ajustado.charAt(i));
    }
}

static String leerCadenaFija(RandomAccessFile fichero,
                             int longitud) throws IOException {
    StringBuilder resultado = new StringBuilder(longitud);

    for (int i = 0; i < longitud; i++) {
        resultado.append(fichero.readChar());
    }

    return resultado.toString().stripTrailing();
}
```

### Modificar únicamente la nota media

**Con `RandomAccessFile`:**

```
// Posición del registro + desplazamiento del campo dentro del registro.
long posicionNota = indice * (long) TAM_REGISTRO
        + Integer.BYTES
        + TAM_NOMBRE;

fichero.seek(posicionNota);       // saltamos directamente al campo
fichero.writeDouble(nuevaNota);   // sustituimos solo sus 8 bytes
```

**La misma modificación con `FileChannel`:**

```
long posicionNota = indice * (long) TAM_REGISTRO
        + Integer.BYTES
        + TAM_NOMBRE;

// Preparamos exactamente los 8 bytes del double.
ByteBuffer buffer = ByteBuffer.allocate(Double.BYTES);
buffer.putDouble(nuevaNota);
buffer.flip();

// Nos situamos en el mismo desplazamiento calculado antes.
canal.position(posicionNota);

while (buffer.hasRemaining()) {
    canal.write(buffer);
}
```

**La fórmula no cambia al cambiar de API.** La posición de `notaMedia` depende del formato de nuestro registro, no de `RandomAccessFile` ni de `FileChannel`. Lo que cambia es la forma de transportar e interpretar los 8 bytes.

El registro completo no se reescribe. El programa calcula el desplazamiento exacto del campo y modifica únicamente los 8 bytes del `double`.

## 6. Operaciones y patrones con `RandomAccessFile`

| Método / constructor | Funcionalidad |
| --- | --- |
| `RandomAccessFile(String, String)` | Abre el fichero en el modo indicado. |
| `seek(long pos)` | Mueve el puntero a una posición absoluta en bytes. |
| `getFilePointer()` | Devuelve la posición actual del puntero. |
| `length()` | Devuelve el tamaño del fichero en bytes. |
| `setLength(long)` | Amplía o trunca el fichero. |
| `read()`, `read(byte[])` | Lee bytes desde la posición actual. |
| `readInt()`, `readLong()`, `readDouble()`, `readChar()`, `readUTF()` | Lee valores tipados en binario. |
| `write(...)` | Escribe bytes desde la posición actual. |
| `writeInt()`, `writeLong()`, `writeDouble()`, `writeChar()`, `writeUTF()` | Escribe valores tipados en binario. |
| `close()` | Cierra el fichero. Habitualmente se gestiona con try-with-resources. |

### Patrón: comparar lectura secuencial y aleatoria

```
// Acceso aleatorio al entero de índice i
try (RandomAccessFile fichero = new RandomAccessFile("datos.bin", "r")) {
    long posicion = i * (long) Integer.BYTES;
    fichero.seek(posicion);
    int valor = fichero.readInt();
}
```

Para comparar tiempos de forma más precisa puede utilizarse `System.nanoTime()` antes y después de cada operación. Una única medición no constituye un benchmark riguroso, pero permite visualizar la diferencia conceptual entre recorrer y posicionarse.

### Patrón: insertar un entero desplazando los posteriores

Insertar no es lo mismo que sobrescribir. Si queremos conservar el elemento existente en esa posición, hay que ampliar el fichero y desplazar los elementos hacia la derecha empezando desde el final.

```
long numeroElementos = fichero.length() / Integer.BYTES;

if (posicion < 0 || posicion > numeroElementos) {
    throw new IllegalArgumentException("Posición no válida");
}

fichero.setLength(fichero.length() + Integer.BYTES);

for (long i = numeroElementos; i > posicion; i--) {
    fichero.seek((i - 1) * Integer.BYTES);
    int valor = fichero.readInt();

    fichero.seek(i * Integer.BYTES);
    fichero.writeInt(valor);
}

fichero.seek(posicion * Integer.BYTES);
fichero.writeInt(nuevoValor);
```

### Patrón: modificar un byte en su misma posición

```
while (fichero.getFilePointer() < fichero.length()) {
    long posicion = fichero.getFilePointer();
    int dato = fichero.read();

    if (dato == 'e') {
        fichero.seek(posicion);
        fichero.write('E');
    }
}
```

**Por qué este ejemplo funciona:** en ASCII y UTF-8 los caracteres `e` y `E` ocupan un byte. No debe generalizarse esta técnica a sustituciones arbitrarias en texto UTF-8, porque otros caracteres pueden ocupar diferente número de bytes.

<a id="intercambio"></a>

## 7. Intercambio de datos entre aplicaciones

El **intercambio de datos** consiste en transferir información entre sistemas, plataformas, procesos o aplicaciones. Puede producirse mediante ficheros, APIs, mensajería, streaming u otros mecanismos.

Aplicación A ── XML / JSON / CSV / binario ──► Aplicación B

Para que el receptor pueda interpretar los datos, ambos extremos necesitan compartir un **contrato de representación**: un formato conocido y reglas sobre el significado de la información.

**Almacenamiento**

Dónde permanecen los datos: fichero, base de datos, objeto en memoria...

**Representación**

Cómo se codifican y estructuran: XML, JSON, CSV, Protobuf...

**Transporte**

Cómo viajan: fichero, HTTP, socket, cola de mensajes, streaming...

<a id="formato-parser-binding"></a>

## 8. Formato, parser y data binding: tres conceptos diferentes

Estos conceptos están relacionados, pero resuelven problemas distintos. Entender su separación evita confundir DOM, SAX, JAXB o Jackson como si fueran herramientas equivalentes.

### 7.1. Formato de datos

Un **formato** define **cómo se representa la información**: su sintaxis, estructura y reglas. La misma información puede representarse con formatos diferentes.

**XML**

```
<maquina nombre="S0001">
    <tipo>Server</tipo>
    <fabricante>HP</fabricante>
</maquina>
```

**JSON**

```
{
  "nombre": "S0001",
  "tipo": "Server",
  "fabricante": "HP"
}
```

Los datos conceptuales son los mismos; cambia su representación.

### 7.2. ¿Qué es un parser?

Un **parser** o analizador sintáctico es el componente que **lee una representación y reconoce su estructura de acuerdo con las reglas del formato**. No se limita a leer caracteres: identifica qué significan dentro de la gramática del documento.

texto XML ──► PARSER ──► elementos, atributos, texto, jerarquía...

Por ejemplo, ante:

```
<Maquina Nombre="S0001">
    <Tipo>Server</Tipo>
</Maquina>
```

un parser XML reconoce que `Maquina` es un elemento, que `Nombre` es un atributo, que `Tipo` es un elemento hijo y que `Server` es contenido textual.

Distintos parsers pueden exponer esa estructura de maneras diferentes:

| Parser | Qué entrega al programa |
| --- | --- |
| **DOM** | Un árbol completo de objetos `Document`, `Element`, `Node`... |
| **SAX** | Eventos: “empieza elemento”, “hay texto”, “termina elemento”... |
| **StAX** | Un cursor/eventos que el propio programa solicita secuencialmente. |

**Para qué sirve un parser:** cuando necesitamos trabajar con la **estructura del documento**, recorrerla, buscar nodos, reaccionar a eventos o construir una representación propia a partir de ella.

### 7.3. ¿Qué es data binding?

El **data binding** establece una correspondencia entre los datos representados en un formato y las **clases y objetos del programa**.

XML / JSON ◄────────► objetos Java  
representación modelo de dominio

Con parsing directo podemos terminar trabajando con:

```
Element maquina = ...;
String nombre = maquina.getAttribute("Nombre");
```

Con data binding queremos trabajar directamente con:

```
Maquina maquina = ...;
System.out.println(maquina.getNombre());
```

La biblioteca se encarga de relacionar elementos, atributos o propiedades del documento con campos y propiedades de nuestras clases.

### 7.4. Cómo se relacionan

DATOS  
│  
▼  
FORMATO (XML / JSON / ...)  
│  
├────────► PARSING ───────► estructura / eventos  
│ DOM · SAX · StAX  
│  
└────────► DATA BINDING ──► objetos Java  
 JAXB · Jackson**No son alternativas exactamente equivalentes.** El data binding también necesita interpretar el formato; la biblioteca realiza internamente el parsing necesario y añade después la correspondencia con clases Java. DOM/SAX/StAX exponen el documento; JAXB/Jackson intentan entregarnos directamente objetos del modelo.

<a id="formatos"></a>

## 9. Panorama de formatos de intercambio

No existe un único formato adecuado para todos los problemas. En este bloque se programarán principalmente XML y JSON; los demás deben reconocerse y relacionarse con sus casos de uso.

| Formato | Tipo | Uso habitual |
| --- | --- | --- |
| **XML** | Texto jerárquico | Integración, intercambio estructurado, validación con esquemas, sistemas legacy. |
| **JSON** | Texto estructurado | APIs REST, web, configuración e intercambio entre servicios. |
| **CSV** | Texto tabular | Importación/exportación y compatibilidad ofimática. |
| **YAML** | Texto jerárquico | Configuración de aplicaciones e infraestructura. |
| **Protobuf** | Binario con esquema | RPC, gRPC y comunicación eficiente entre servicios. |
| **Parquet** | Binario columnar | Analítica, Data Lakes y grandes volúmenes de datos. |

<a id="xml-repaso"></a>

## 10. XML

XML ya no se aborda como lenguaje nuevo. Solo necesitamos recordar la estructura imprescindible para trabajar después con DOM, SAX, XPath y data binding.

```
<Hardware>
    <Maquina Nombre="S0001">
        <Info>
            <Tipo>Server</Tipo>
            <Fabricante>HP</Fabricante>
        </Info>
        <Config>
            <OS>Red Hat Enterprise</OS>
            <IP>192.168.20.105</IP>
        </Config>
    </Maquina>
</Hardware>
```

| Concepto | Ejemplo |
| --- | --- |
| Elemento raíz | `Hardware` |
| Elemento | `Maquina`, `Info`, `Tipo`... |
| Atributo | `Nombre="S0001"` |
| Texto | `Server`, `HP`... |
| Jerarquía | `Hardware → Maquina → Info → Tipo` |

<a id="dom"></a>

## 11. DOM: cargar XML como un árbol de objetos

**DOM (Document Object Model)** analiza el documento XML completo y construye en memoria un árbol de objetos. Después podemos recorrerlo en cualquier dirección, buscar nodos, leer atributos o modificar la estructura.

XML ──► DocumentBuilder ──► Document  
 └── Element  
 ├── Attr  
 ├── Text  
 └── Element ...

### Leer `equipos.xml`

```
DocumentBuilderFactory factory = DocumentBuilderFactory.newInstance();
DocumentBuilder builder = factory.newDocumentBuilder();

Document documento = builder.parse(new File("equipos.xml"));
documento.getDocumentElement().normalize();

NodeList maquinas = documento.getElementsByTagName("Maquina");

for (int i = 0; i < maquinas.getLength(); i++) {
    Element maquina = (Element) maquinas.item(i);

    String nombre = maquina.getAttribute("Nombre");
    String tipo = maquina.getElementsByTagName("Tipo")
                         .item(0).getTextContent();
    String fabricante = maquina.getElementsByTagName("Fabricante")
                               .item(0).getTextContent();

    System.out.println(nombre + " - " + tipo + " - " + fabricante);
}
```

### Operaciones habituales: DOM

| Clase / método | Uso |
| --- | --- |
| `DocumentBuilderFactory.newInstance()` | Crea la factoría para obtener un parser DOM. |
| `newDocumentBuilder()` | Obtiene un `DocumentBuilder`. |
| `parse(File)` | Analiza el XML y devuelve un `Document`. |
| `newDocument()` | Crea un documento DOM vacío. |
| `getDocumentElement()` | Obtiene el elemento raíz. |
| `getElementsByTagName(String)` | Obtiene todos los elementos descendientes con ese nombre. |
| `getChildNodes()` | Devuelve los nodos hijos. |
| `NodeList.getLength()` / `item(int)` | Recorre una colección de nodos. |
| `Element.getAttribute(String)` | Obtiene el valor de un atributo. |
| `getTextContent()` | Obtiene el texto contenido en un nodo y sus descendientes. |
| `createElement()`, `createTextNode()` | Crea elementos y nodos de texto. |
| `setAttribute()` | Añade o modifica un atributo. |
| `appendChild()` | Añade un nodo hijo. |

### Crear un XML con DOM

```
DocumentBuilderFactory factory = DocumentBuilderFactory.newInstance();
DocumentBuilder builder = factory.newDocumentBuilder();
Document documento = builder.newDocument();

Element hardware = documento.createElement("Hardware");
documento.appendChild(hardware);

Element maquina = documento.createElement("Maquina");
maquina.setAttribute("Nombre", "S0001");
hardware.appendChild(maquina);

Element info = documento.createElement("Info");
maquina.appendChild(info);

Element tipo = documento.createElement("Tipo");
tipo.appendChild(documento.createTextNode("Server"));
info.appendChild(tipo);

Element fabricante = documento.createElement("Fabricante");
fabricante.appendChild(documento.createTextNode("HP"));
info.appendChild(fabricante);
```

Para guardar el árbol DOM en un fichero se utiliza un transformador:

```
Transformer transformer = TransformerFactory.newInstance().newTransformer();
transformer.setOutputProperty(OutputKeys.INDENT, "yes");

transformer.transform(
        new DOMSource(documento),
        new StreamResult(new File("equipos_generado.xml"))
);
```

<a id="sax"></a>

## 12. SAX: parsing basado en eventos

SAX procesa el XML secuencialmente. No construye un árbol completo en memoria. El parser avanza por el documento y **llama a métodos de nuestro handler** cuando encuentra eventos relevantes.

<Maquina> ──► startElement()  
texto ──► characters()  
</Maquina> ──► endElement()

Este modelo se denomina **push**: es el parser quien controla el recorrido y “empuja” los eventos hacia nuestro código.

### Ejemplo de lectura

```
class EquiposHandler extends DefaultHandler {

    private StringBuilder texto = new StringBuilder();

    @Override
    public void startElement(String uri, String localName,
                             String qName, Attributes attributes) {
        texto.setLength(0);

        if (qName.equals("Maquina")) {
            System.out.println("Máquina: " + attributes.getValue("Nombre"));
        }
    }

    @Override
    public void characters(char[] ch, int start, int length) {
        texto.append(ch, start, length);
    }

    @Override
    public void endElement(String uri, String localName, String qName) {
        String contenido = texto.toString().trim();

        if (qName.equals("Tipo") || qName.equals("Fabricante")
                || qName.equals("OS") || qName.equals("IP")) {
            System.out.println("  " + qName + ": " + contenido);
        }
    }
}
```

```
SAXParserFactory factory = SAXParserFactory.newInstance();
SAXParser parser = factory.newSAXParser();

parser.parse(new File("equipos.xml"), new EquiposHandler());
```

**Detalle importante:** `characters()` puede ejecutarse varias veces para un mismo texto. Por eso es más seguro acumular los caracteres en un `StringBuilder` y procesarlos al recibir `endElement()`.

### Operaciones habituales: SAX

| Elemento | Uso |
| --- | --- |
| `SAXParserFactory.newInstance()` | Crea la factoría SAX. |
| `newSAXParser()` | Obtiene el parser. |
| `parse(File, DefaultHandler)` | Procesa el documento notificando eventos al handler. |
| `startDocument()` / `endDocument()` | Inicio y final del documento. |
| `startElement(...)` | Inicio de un elemento y acceso a sus atributos. |
| `characters(...)` | Fragmento de contenido textual. |
| `endElement(...)` | Fin de un elemento. |
| `Attributes.getValue(String)` | Obtiene un atributo del elemento actual. |

<a id="stax"></a>

## 13. StAX: parsing secuencial controlado por el programa

StAX también procesa el XML como un flujo, pero utiliza un modelo **pull**. Es nuestro código quien pide al parser el siguiente evento.

**SAX · push**

El parser avanza y llama a nuestros métodos.

**StAX · pull**

Nuestro código llama a `next()` para avanzar.

```
XMLInputFactory factory = XMLInputFactory.newFactory();

try (FileInputStream entrada = new FileInputStream("equipos.xml")) {
    XMLStreamReader reader = factory.createXMLStreamReader(entrada);

    while (reader.hasNext()) {
        int evento = reader.next();

        if (evento == XMLStreamConstants.START_ELEMENT
                && reader.getLocalName().equals("Maquina")) {
            System.out.println(reader.getAttributeValue(null, "Nombre"));
        }
    }

    reader.close();
}
```

StAX debe conocerse como tercera estrategia, aunque en este bloque el trabajo práctico principal se centra en DOM y SAX.

<a id="comparativa-parsers"></a>

## 14. DOM vs SAX vs StAX

| Aspecto | DOM | SAX | StAX |
| --- | --- | --- | --- |
| **Modelo** | Árbol de objetos | Eventos push | Eventos/cursor pull |
| **Carga todo en memoria** | Sí | No | No |
| **Navegar hacia atrás** | Sí | No | No |
| **Modificar estructura** | Muy cómodo | No construye estructura | Procesamiento de flujo |
| **Documentos grandes** | Mayor consumo | Adecuado | Adecuado |
| **Control del recorrido** | Programa | Parser | Programa |

**Criterio práctico:** DOM es muy cómodo cuando necesitamos navegar o modificar el árbol; SAX y StAX son apropiados cuando queremos procesar secuencialmente sin cargar todo el documento.

<a id="xpath"></a>

## 15. XPath: localizar información dentro del árbol XML

XPath es un lenguaje de expresiones para seleccionar nodos de un documento XML. Resulta especialmente cómodo cuando ya tenemos un DOM y queremos localizar una parte concreta sin recorrer manualmente todos sus hijos.

| Expresión | Significado |
| --- | --- |
| `/Hardware/Maquina` | Todos los elementos `Maquina` hijos de `Hardware`. |
| `//Maquina` | Elementos `Maquina` en cualquier profundidad. |
| `//Maquina/@Nombre` | Atributos `Nombre` de las máquinas. |
| `//Maquina/Info/Tipo` | Tipos de todas las máquinas. |
| `//Maquina[@Nombre='S0001']` | Máquina cuyo atributo tiene ese valor. |
| `//Maquina[Config/OS='CentOS']` | Máquinas cuyo sistema operativo es CentOS. |

### Obtener los nombres de las máquinas

```
XPath xpath = XPathFactory.newInstance().newXPath();

NodeList nombres = (NodeList) xpath.evaluate(
        "//Maquina/@Nombre",
        documento,
        XPathConstants.NODESET
);

for (int i = 0; i < nombres.getLength(); i++) {
    System.out.println(nombres.item(i).getNodeValue());
}
```

### Operaciones habituales: XPath

| Método / constante | Uso |
| --- | --- |
| `XPathFactory.newInstance()` | Crea la factoría. |
| `newXPath()` | Crea el evaluador XPath. |
| `evaluate(expresion, objeto)` | Evalúa una expresión y devuelve texto por defecto. |
| `evaluate(..., XPathConstants.NODESET)` | Devuelve un conjunto de nodos. |
| `XPathConstants.STRING`, `NUMBER`, `BOOLEAN`, `NODE`, `NODESET` | Indican el tipo de resultado esperado. |

<a id="validacion"></a>

## 16. XML bien formado, XML válido y esquemas

Son conceptos distintos:

**Bien formado**

Cumple las reglas sintácticas de XML: un único elemento raíz, etiquetas correctamente cerradas y anidadas, atributos correctamente escritos...

**Válido**

Además cumple un conjunto de reglas de contenido definido por un esquema, por ejemplo DTD o XSD.

**XSD** puede definir elementos, atributos, orden, cardinalidades y tipos de datos. En este bloque interesa comprender para qué sirve la validación; no se profundiza en la creación de esquemas complejos.

**XML Catalogs:** permiten resolver identificadores y referencias externas (por ejemplo DTD o XSD) hacia copias locales o ubicaciones controladas. Son útiles para evitar dependencias de red y controlar qué recursos externos utiliza un procesamiento XML.**XML procedente de fuentes no confiables:** en aplicaciones reales debe configurarse el parser para controlar entidades externas y DTDs y evitar vulnerabilidades como XXE. Si la aplicación necesita DTD/XSD externos, la política debe configurarse de forma explícita en lugar de aceptarlos indiscriminadamente.

<a id="databinding"></a>

## 17. Data binding: trabajar con objetos en lugar de nodos

El objetivo del data binding es que la representación externa se corresponda con un **modelo de clases**. El programa deja de centrarse en etiquetas o propiedades del documento y trabaja con objetos de dominio.

equipos.xml equipos.json  
 │ │  
 └──────────────┬────────────────────┘  
 ▼  
 modelo Java  
 Hardware  
 │  
 List<Maquina>  
 / \  
 Info Config

La biblioteca necesita conocer la correspondencia entre la estructura externa y las clases. Esa correspondencia puede deducirse por convenciones o declararse con anotaciones.

| Estrategia | El código trabaja principalmente con... |
| --- | --- |
| **Parsing DOM** | `Document`, `Node`, `Element`, atributos y texto. |
| **Parsing SAX** | Eventos y estado acumulado durante el recorrido. |
| **Data binding** | `Hardware`, `Maquina`, `Info`, `Config`. |

El data binding es especialmente útil cuando el documento representa claramente entidades del dominio y queremos leerlas, modificarlas y volverlas a serializar sin programar manualmente cada navegación del documento.

<a id="jaxb"></a>

## 18. XML ↔ objetos Java con Jakarta XML Binding (JAXB)

Jakarta XML Binding automatiza la correspondencia entre documentos XML y objetos Java. En Java moderno se utiliza como dependencia externa mediante paquetes `jakarta.xml.bind`.

### Dependencias Maven para Java 25

```
<dependency>
    <groupId>jakarta.xml.bind</groupId>
    <artifactId>jakarta.xml.bind-api</artifactId>
    <version>4.0.5</version>
</dependency>

<dependency>
    <groupId>org.glassfish.jaxb</groupId>
    <artifactId>jaxb-runtime</artifactId>
    <version>4.0.9</version>
</dependency>
```

### Modelo común

El fichero `equipos.xml` puede representarse mediante cuatro clases: `Hardware`, `Maquina`, `Info` y `Config`. Para simplificar el binding usamos acceso a campos con `@XmlAccessorType(XmlAccessType.FIELD)`.

```
@XmlRootElement(name = "Hardware")
@XmlAccessorType(XmlAccessType.FIELD)
public class Hardware {

    @XmlElement(name = "Maquina")
    private List<Maquina> maquinas = new ArrayList<>();

    public Hardware() { }

    // getters y setters
}
```

```
@XmlAccessorType(XmlAccessType.FIELD)
public class Maquina {

    @XmlAttribute(name = "Nombre")
    private String nombre;

    @XmlElement(name = "Info")
    private Info info;

    @XmlElement(name = "Config")
    private Config config;

    public Maquina() { }

    // getters y setters
}
```

```
@XmlAccessorType(XmlAccessType.FIELD)
public class Info {
    @XmlElement(name = "Tipo")
    private String tipo;

    @XmlElement(name = "Fabricante")
    private String fabricante;

    public Info() { }
    // getters y setters
}

@XmlAccessorType(XmlAccessType.FIELD)
public class Config {
    @XmlElement(name = "OS")
    private String os;

    @XmlElement(name = "IP")
    private String ip;

    public Config() { }
    // getters y setters
}
```

### Unmarshalling: XML → objetos

equipos.xml ── unmarshalling ──► Hardware → List<Maquina>

```
JAXBContext context = JAXBContext.newInstance(Hardware.class);
Unmarshaller unmarshaller = context.createUnmarshaller();

Hardware hardware = (Hardware) unmarshaller.unmarshal(
        new File("equipos.xml")
);

for (Maquina maquina : hardware.getMaquinas()) {
    System.out.println(maquina.getNombre());
}
```

### Marshalling: objetos → XML

Hardware ── marshalling ──► equipos\_salida.xml

```
JAXBContext context = JAXBContext.newInstance(Hardware.class);
Marshaller marshaller = context.createMarshaller();

marshaller.setProperty(
        Marshaller.JAXB_FORMATTED_OUTPUT,
        Boolean.TRUE
);

marshaller.marshal(
        hardware,
        new File("equipos_salida.xml")
);
```

### Anotaciones y operaciones habituales

| Elemento | Uso |
| --- | --- |
| `@XmlRootElement` | Identifica la clase correspondiente al elemento raíz. |
| `@XmlAccessorType(XmlAccessType.FIELD)` | Indica que el binding se realiza directamente sobre los campos. |
| `@XmlElement` | Relaciona un campo con un elemento XML. |
| `@XmlAttribute` | Relaciona un campo con un atributo XML. |
| `@XmlElementWrapper` | Representa un elemento contenedor alrededor de una colección. |
| `@XmlTransient` | Excluye un campo del binding XML. |
| `JAXBContext.newInstance(...)` | Prepara el contexto para las clases indicadas. |
| `createMarshaller()` | Crea el componente objetos → XML. |
| `createUnmarshaller()` | Crea el componente XML → objetos. |
| `marshal(...)` | Serializa el objeto como XML. |
| `unmarshal(...)` | Deserializa XML a objetos. |

<a id="json-jackson"></a>

## 19. JSON y data binding con Jackson

JSON representa objetos mediante pares nombre-valor y colecciones mediante arrays. Para esta unidad interesa especialmente utilizar el **mismo modelo Java** empleado con XML y leer/escribir JSON con Jackson.

```
{
  "maquinas" : [ {
    "nombre" : "S0001",
    "info" : {
      "tipo" : "Server",
      "fabricante" : "HP"
    },
    "config" : {
      "os" : "Red Hat Enterprise",
      "ip" : "192.168.20.105"
    }
  } ]
}
```

### Dependencia Maven

```
<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
    <version>2.22.2</version>
</dependency>
```

### JSON → objetos Java

```
ObjectMapper mapper = new ObjectMapper();

Hardware hardware = mapper.readValue(
        new File("equipos.json"),
        Hardware.class
);

for (Maquina maquina : hardware.getMaquinas()) {
    System.out.println(maquina.getNombre());
}
```

### Objetos Java → JSON

```
ObjectMapper mapper = new ObjectMapper();

mapper.writerWithDefaultPrettyPrinter()
      .writeValue(new File("equipos_salida.json"), hardware);
```

### Operaciones habituales: Jackson 2.x

| Método / anotación | Uso |
| --- | --- |
| `new ObjectMapper()` | Crea el objeto principal de binding JSON. |
| `readValue(File, Clase.class)` | Convierte JSON en un objeto Java. |
| `writeValue(File, objeto)` | Escribe un objeto como JSON. |
| `writeValueAsString(objeto)` | Devuelve la representación JSON como `String`. |
| `writerWithDefaultPrettyPrinter()` | Configura una salida legible con indentación. |
| `@JsonProperty("nombreExterno")` | Asocia una propiedad JSON con un nombre distinto al campo Java. |
| `@JsonIgnore` | Excluye una propiedad del binding. |

### Cuando el contrato JSON exige nombres exactos

Si el JSON externo utiliza nombres como `"Maquina"` o `"@Nombre"`, el modelo puede adaptarse con anotaciones sin cambiar los nombres internos de Java:

```
@JsonProperty("Maquina")
private List<Maquina> maquinas;

@JsonProperty("@Nombre")
private String nombre;
```

**Idea importante:** el modelo Java representa el dominio. JAXB y Jackson son capas de adaptación entre ese modelo y dos formatos externos diferentes.

<a id="otros-formatos"></a>

## 20. Otras tecnologías que conviene reconocer

No forman parte del desarrollo práctico de esta guía, pero deben identificarse y asociarse con el problema que resuelven.

| Tecnología | Para qué se utiliza |
| --- | --- |
| **XML Catalogs** | Resolver de forma controlada referencias externas XML como DTD/XSD, frecuentemente hacia copias locales. |
| **Protocol Buffers (Protobuf)** | Serialización binaria compacta y tipada; muy habitual junto a gRPC y microservicios. |
| **Apache Avro** | Serialización basada en esquemas con evolución de versiones; frecuente en plataformas de datos y mensajería. |
| **Apache Thrift** | Definición de estructuras y servicios RPC interoperables entre lenguajes. |
| **MessagePack** | Representación binaria compacta de estructuras similares a JSON. |
| **CBOR** | Formato binario estandarizado y compacto, útil en protocolos y dispositivos con recursos limitados. |
| **Parquet** | Formato columnar para análisis de grandes volúmenes de datos con lectura selectiva y compresión. |
| **ORC** | Formato columnar optimizado para cargas analíticas, especialmente en ecosistemas Hadoop. |
| **Apache Arrow** | Representación columnar en memoria orientada al intercambio y procesamiento eficiente entre procesos y lenguajes. |

<a id="referencia"></a>

## 21. Referencia rápida: qué elegir

| Necesidad | Herramienta |
| --- | --- |
| Ir directamente al elemento *i* de un fichero binario de tamaño fijo | `RandomAccessFile` o `FileChannel` + cálculo de posición |
| Acceso posicional con NIO, buffers y operaciones absolutas | `FileChannel` + `ByteBuffer` |
| Modificar un campo de un registro sin reescribir el resto | `seek()` / `position()` + escritura en la posición calculada |
| Cargar un XML, navegarlo y modificarlo cómodamente | DOM |
| Procesar un XML secuencialmente mediante callbacks | SAX |
| Procesar XML secuencialmente controlando cuándo avanzar | StAX |
| Localizar nodos concretos dentro de un XML | XPath |
| Convertir XML ↔ objetos Java | Jakarta XML Binding (JAXB) |
| Convertir JSON ↔ objetos Java | Jackson `ObjectMapper` |

### Excepciones que aparecerán con frecuencia

| Excepción | Contexto |
| --- | --- |
| `IOException` | Errores generales de entrada/salida. |
| `EOFException` | Intento de leer un dato completo más allá del final del fichero. |
| `ParserConfigurationException` | No se puede crear/configurar el parser DOM/SAX solicitado. |
| `SAXException` | Error durante el parsing XML. |
| `XPathExpressionException` | Expresión XPath inválida o error al evaluarla. |
| `JAXBException` | Errores de marshalling/unmarshalling XML. |
| `JsonProcessingException` / `IOException` | Errores de procesamiento o E/S con Jackson. |

## Resumen esencial

| **Acceso aleatorio** | Acceder directamente a una posición física conocida o calculada en bytes. |
| --- | --- |
| **`RandomAccessFile`** | `seek()` + métodos tipados como `readInt()`/`writeDouble()`. |
| **`FileChannel`** | `position()` y operaciones posicionales; trabaja con `ByteBuffer`. |
| **Registro fijo** | `posición = índice × tamañoRegistro`; los campos se localizan sumando desplazamientos. |
| **Formato** | Define cómo se representan los datos. |
| **Parser** | Interpreta la sintaxis y estructura del formato. |
| **DOM / SAX / StAX** | Tres modelos XML: árbol, eventos push y lectura pull. |
| **XPath** | Selecciona nodos dentro de un árbol XML. |
| **Data binding** | Relaciona representaciones externas con objetos Java. |
| **JAXB / Jackson** | XML ↔ objetos Java / JSON ↔ objetos Java. |
