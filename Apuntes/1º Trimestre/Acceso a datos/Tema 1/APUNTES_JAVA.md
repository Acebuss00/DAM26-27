Acceso a datos

# Ficheros en Java I

java.io · NIO · NIO.2 · Acceso secuencial
**Idea de partida**

Un fichero permite almacenar datos fuera de la memoria del programa. Puede utilizarse para conservar información entre ejecuciones, pero en una aplicación real también aparece en tareas de configuración, registro de actividad, intercambio de datos, generación de documentos o integración con otros sistemas.

## Contenido

1. [Para qué se utilizan los ficheros](#para-que)
2. [Uso real en aplicaciones](#uso-real)
3. [java.io, NIO y NIO.2](#apis)
4. [Texto, binario y codificación](#tipos)
5. [Rutas: File, Path y Files](#file)
6. [Streams: el modelo de java.io](#streams)
7. [Flujos de bytes](#bytes)
8. [Flujos de caracteres y Charset](#caracteres)
9. [Búferes de java.io y wrappers](#buffers)
10. [NIO: Channel y Buffer](#nio)
11. [Cierre de recursos y excepciones](#recursos)
12. [Tipos primitivos en binario](#data)
13. [Serialización de objetos](#objetos)
14. [Acceso secuencial y aleatorio](#random)
15. [Cómo elegir la API adecuada](#elegir)
16. [Más formas de leer y escribir con java.io](#formas-java-io)
17. [Comparativa práctica entre APIs](#comparativa-practica)

## 1. ¿Para qué utilizamos ficheros?

Durante la ejecución de un programa trabajamos normalmente con datos almacenados en memoria:

```
String nombre = "Ana";
int edad = 24;
```

La memoria de trabajo no es el lugar adecuado para conservar permanentemente esos datos. Cuando necesitamos que la información permanezca disponible después de terminar el programa, debemos almacenarla en un sistema persistente.

Programa ── escribe ──► Fichero
Programa ◄── lee ───── Fichero

Un **fichero** es una secuencia de datos almacenada en un sistema de archivos. Puede contener texto, imágenes, audio, documentos o cualquier otro tipo de información. El programa interpreta esos datos según el formato utilizado.

## 2. ¿Dónde aparecen los ficheros en aplicaciones reales?

Aunque una aplicación utilice una base de datos, normalmente seguirá trabajando con numerosos ficheros. No todos sirven para almacenar directamente los datos principales de la aplicación.

### Configuración

Permiten cambiar el comportamiento de una aplicación sin modificar su código.

`application.properties`
`application.yml`
`logback.xml`

### Proyecto y construcción

Las herramientas de desarrollo leen ficheros que describen dependencias, tareas y configuración del proyecto.

`pom.xml`
`build.gradle`
`Dockerfile`

### Logs

Registran actividad, [avisos](https://aulavirtual33.educa.madrid.org/ies.islasfilipinas.madrid/mod/forum/view.php?id=33587 "Avisos") y errores para poder diagnosticar qué ha ocurrido en una aplicación.

`aplicacion.log`
`errores.log`

### Importación y exportación

Sirven para intercambiar información entre aplicaciones o con usuarios.

`clientes.csv`
`pedidos.json`
`facturas.xml`

### Procesamiento por lotes

Es frecuente recibir periódicamente ficheros, procesarlos y generar otros nuevos o cargar sus datos en una base de datos.

### Documentos y recursos

Las aplicaciones almacenan, reciben o generan imágenes, PDF, hojas de cálculo, plantillas, certificados y otros recursos.

`factura.pdf` · `logo.png`
**Base de datos y fichero no son conceptos opuestos.**

Un gestor de bases de datos almacena internamente información en ficheros de datos, índices, registros de transacciones y ficheros de configuración. Una aplicación Java, sin embargo, normalmente accede al gestor mediante JDBC y SQL, no modificando directamente esos ficheros.

**Importante:** CSV, JSON y XML son **formatos de representación de datos**. Pueden almacenarse en ficheros, enviarse por red o guardarse dentro de otros sistemas.

## 3. Las tres familias de entrada/salida que vamos a utilizar

Antes de hablar de clases concretas necesitamos situar **las tres etapas de la entrada/salida de Java**. No son tres mundos aislados ni tres versiones que se sustituyan una a otra. Java fue añadiendo herramientas y hoy pueden aparecer juntas en el mismo programa.

Java 1.0 ──► **java.io**
Java 1.4 ──► **NIO** añade canales y buffers
Java 7 ──► **NIO.2** amplía NIO con una API moderna de sistema de archivos

### 1. `java.io`: la API clásica

Es la primera familia que apareció y sigue siendo perfectamente válida. En esta API encontraremos dos ideas distintas:

- `File` representa una **ruta** y además ofrece algunas operaciones sobre el fichero o directorio al que apunta.
- Los **streams** permiten leer y escribir el **contenido** de forma secuencial.

java.io
├── File → representa una ruta y permite algunas operaciones
└── Streams → hacen circular el contenido

Más adelante veremos también `RandomAccessFile`, que permite saltar a posiciones concretas de un fichero.

### 2. NIO: ¿qué aporta cuando trabajamos con ficheros?

`java.io` sigue siendo una API válida y, para muchas tareas, es la opción más sencilla. Su modelo de **streams** encaja muy bien cuando queremos leer o escribir datos de forma secuencial.

Cuando trabajamos con **ficheros**, NIO no se utiliza principalmente para evitar que el programa se bloquee. Una operación con `FileChannel` **puede seguir siendo bloqueante**. Lo que NIO aporta aquí es otro modelo de trabajo que nos da más control sobre **bloques de bytes, posiciones y transferencias**.

fichero ⇄ FileChannel ⇄ ByteBuffer ⇄ programa

Por eso no necesitamos NIO para leer unas líneas de texto solo porque el fichero sea grande o porque vayamos a trabajar con varios ficheros. Para muchas operaciones, `BufferedReader`, los streams clásicos o los métodos de `Files` expresan mejor lo que queremos hacer. `FileChannel` empieza a tener sentido cuando necesitamos **control explícito sobre cómo y dónde se mueven los datos**.

**Contexto: ¿y la E/S no bloqueante?**

NIO también se diseñó para otros escenarios, especialmente **comunicaciones de red**. Allí sí es muy importante poder usar canales no bloqueantes y mecanismos como `Selector` para gestionar muchas conexiones sin dedicar necesariamente un hilo esperando a cada una. Esa capacidad explica parte de la importancia de NIO, pero **no es la razón principal por la que utilizamos `FileChannel` con ficheros**.

java.io con ficheros → streams → lectura/escritura secuencial sencilla
NIO con ficheros → FileChannel + ByteBuffer → bloques, posiciones y mayor control
NIO en red → canales no bloqueantes + Selector → muchas conexiones

### 3. NIO.2: una API moderna para el sistema de archivos

Java 7 amplió NIO con lo que habitualmente llamamos **NIO.2**. En el trabajo cotidiano con ficheros, sus dos piezas más importantes son:

- `Path`: representa una **ruta**;
- `Files`: agrupa operaciones sobre las rutas: comprobar, crear, copiar, mover, borrar, abrir, leer o escribir.

**NIO.2 no significa que exista un paquete `java.nio2`.** Es el nombre habitual de la ampliación incorporada en Java 7. Sus clases de ficheros están principalmente en `java.nio.file`.

### Ahora sí: tres preguntas que nos ayudarán a elegir

Una vez situadas las tres familias, podemos separar problemas que al principio parecen el mismo:

| Pregunta | `java.io` | NIO / NIO.2 |
| --- | --- | --- |
| **¿Cómo represento dónde está?** | `File` | `Path` (NIO.2) |
| **¿Cómo hago operaciones sobre el fichero o directorio?** | Métodos de `File` para operaciones básicas | `Files` (NIO.2) |
| **¿Cómo leo o escribo el contenido?** | Streams | Canal + buffer cuando necesitamos ese modelo de NIO |

**Una forma sencilla de verlo**

En `java.io`, `File` reúne dos ideas: **representa dónde está un fichero o directorio** y además ofrece algunas operaciones sobre él. NIO.2 separa mejor esas responsabilidades:

Path → dónde está
Files → qué quiero hacer con esa ruta

Para trabajar con el contenido también cambia el modelo:

java.io → stream: los datos van pasando como un flujo
NIO → channel + buffer: el canal transporta bloques y el buffer es la zona de memoria donde trabaja el programa

**Idea clave:** NIO.2 separa responsabilidades que en `File` estaban más agrupadas.

**Orden de aprendizaje de esta guía:** primero veremos cómo se representan las rutas; después aprenderemos el modelo de streams de `java.io`; y solo cuando ese modelo esté claro introduciremos `FileChannel` y `ByteBuffer`.

## 4. Todo fichero acaba siendo una secuencia de bytes

El sistema de almacenamiento guarda bytes. La diferencia entre un fichero de texto, una imagen o un PDF está en **cómo se interpretan esos bytes**.

### Fichero de texto

Los bytes se interpretan como caracteres mediante una **codificación**, por ejemplo UTF-8.

```
Ana;24;8.5
Luis;31;7.2
```

### Fichero binario

Los bytes se interpretan siguiendo el formato definido por el programa o por una especificación.

PDF · JPG · ZIP · DAT · MP3
**No conviene identificar “binario” con “no portable”.** Un fichero binario puede ser completamente interoperable si su formato está bien definido. PNG, PDF o ZIP son ejemplos claros.

## 5. Representar una ruta: de `File` a `Path` + `Files`

En el apartado anterior hemos visto que Java tiene dos formas habituales de representar una ubicación del sistema de archivos. Ahora vamos a estudiarlas en orden histórico: primero `File`, de `java.io`, y después `Path` + `Files`, incorporados con NIO.2.

### 5.1. La API clásica: `File`

`java.io.File` representa una **ruta** que puede apuntar a un fichero o a un directorio. El nombre puede resultar engañoso: el objeto no contiene los bytes del fichero ni abre una conexión con él.

```
// Solo construimos un objeto que representa esta ruta.
File fichero = new File("datos/alumnos.txt");

// Ahora preguntamos al sistema de archivos por esa ruta.
if (fichero.exists()) {
    System.out.println("Nombre: " + fichero.getName());
    System.out.println("Tamaño: " + fichero.length() + " bytes");
}
```

**Primera idea importante:** `new File(...)` **no crea físicamente el fichero**. Solo crea en memoria un objeto que representa una ruta. Tampoco hacemos `close()` sobre un `File`, porque no es un recurso abierto.

Una peculiaridad de esta API es que el mismo objeto mezcla dos responsabilidades:

- representa la ruta;
- ofrece operaciones básicas como `exists()`, `delete()`, `mkdir()` o `listFiles()`.

### 5.2. NIO.2 separa las responsabilidades: `Path` y `Files`

Con NIO.2 el diseño es más explícito. `Path` representa la ruta; `Files` contiene las operaciones que actúan sobre esa ruta.

java.io: File = ruta + algunas operaciones

NIO.2: Path = ruta + Files = operaciones

Primero construimos la ruta:

```
// Path describe la ubicación; todavía no accedemos al disco.
Path ruta = Path.of("datos", "alumnos.txt");
```

Usar varios argumentos evita escribir manualmente separadores como `/` o `\`. Igual que ocurría con `File`, crear un `Path` no crea ningún fichero físico.

Después utilizamos `Files` cuando queremos realizar una operación real:

```
Path ruta = Path.of("datos", "alumnos.txt");

// Files consulta el sistema de archivos.
if (Files.exists(ruta)) {
    System.out.println("Tamaño: " + Files.size(ruta) + " bytes");
}
```

### La misma idea con las dos APIs

| Necesidad | `java.io.File` | NIO.2 |
| --- | --- | --- |
| Representar la ruta | `new File("datos/alumnos.txt")` | `Path.of("datos", "alumnos.txt")` |
| ¿Existe? | `fichero.exists()` | `Files.exists(ruta)` |
| ¿Es directorio? | `fichero.isDirectory()` | `Files.isDirectory(ruta)` |
| Tamaño | `fichero.length()` | `Files.size(ruta)` |
| Eliminar | `fichero.delete()` | `Files.delete(ruta)` |

**No son equivalentes al cien por cien.** NIO.2 ofrece muchas más operaciones, opciones y excepciones específicas. La tabla anterior solo sirve para reconocer la correspondencia conceptual entre la API clásica y la moderna.

### Trabajar con la propia ruta mediante `Path`

Hay operaciones que no modifican el disco: simplemente transforman o consultan el objeto que representa la ruta.

| Qué queremos saber o construir | Método | Ejemplo mental |
| --- | --- | --- |
| Nombre final | `getFileName()` | `alumnos.txt` |
| Directorio padre | `getParent()` | `datos` |
| Convertir a absoluta | `toAbsolutePath()` | Parte del directorio de trabajo |
| Añadir un tramo | `resolve()` | `datos` + `alumnos.txt` |
| Calcular una ruta relativa | `relativize()` | Cómo ir de una ruta a otra |
| Simplificar `.` y `..` | `normalize()` | Transformación sintáctica; no necesita acceder al fichero real |

### Operaciones habituales con `Files`

```
Path origen = Path.of("datos", "alumnos.txt");
Path carpetaCopias = Path.of("datos", "copias");

// 1. Creamos la carpeta si todavía no existe.
Files.createDirectories(carpetaCopias);

// 2. Construimos la ruta del fichero destino.
Path destino = carpetaCopias.resolve("alumnos.txt");

// 3. Copiamos el fichero real.
Files.copy(origen, destino);
```

| Necesidad | Métodos habituales |
| --- | --- |
| Comprobar | `exists()`, `isRegularFile()`, `isDirectory()` |
| Crear | `createFile()`, `createDirectory()`, `createDirectories()` |
| Copiar o mover | `copy()`, `move()` |
| Eliminar | `delete()`, `deleteIfExists()` |
| Texto completo | `readString()`, `writeString()` |
| Abrir lectores/escritores | `newBufferedReader()`, `newBufferedWriter()` |
| Metadatos | `size()`, `getLastModifiedTime()`, `readAttributes()` |

**Cuidado con los métodos que cargan todo en memoria.** `Files.readString()` es excelente para un fichero pequeño o moderado, pero crea todo el contenido en memoria. Si el fichero es grande, más adelante veremos formas de procesarlo progresivamente.

### Recorrer un directorio

```
Path directorio = Path.of("datos");

try (DirectoryStream<Path> entradas = Files.newDirectoryStream(directorio)) {
    for (Path entrada : entradas) {
        System.out.println(entrada.getFileName());
    }
}
```

### Convertir entre `File` y `Path`

```
File fichero = new File("datos/alumnos.txt");
Path ruta = fichero.toPath();

File otraVezFile = ruta.toFile();
```

En código nuevo suele resultar más expresivo utilizar **`Path` + `Files`** para el sistema de archivos, pero `File` sigue apareciendo en código existente y en APIs que todavía lo reciben.

## 6. Streams: el modelo fundamental de `java.io`

Para leer o escribir datos, `java.io` utiliza **flujos** (*streams*). Un stream representa una secuencia de datos que entra o sale del programa.

**Entrada**
Fichero ──► programa
El programa lee datos.
**Salida**
Programa ──► fichero
El programa escribe datos.

### Las cuatro clases base

|  | Bytes | Caracteres |
| --- | --- | --- |
| **Entrada** | `InputStream` | `Reader` |
| **Salida** | `OutputStream` | `Writer` |

### Métodos comunes de las clases base

| Clase | Método / constructor | Utilidad |
| --- | --- | --- |
| `InputStream` | `read()` | Lee un byte. Devuelve un valor 0–255 o `-1` al llegar al final. |
| `InputStream` | `read(byte[] b)` | Lee varios bytes y los guarda en el array. Devuelve cuántos bytes se han leído. |
| `InputStream` | `read(byte[] b, int off, int len)` | Lee hasta `len` bytes y los guarda a partir de la posición `off`. |
| `InputStream` | `skip(long n)` | Intenta saltar `n` bytes del flujo. |
| `InputStream` | `close()` | Cierra el flujo y libera los recursos asociados. |
| `OutputStream` | `write(int b)` | Escribe un byte. |
| `OutputStream` | `write(byte[] b)` | Escribe todos los bytes del array. |
| `OutputStream` | `write(byte[] b, int off, int len)` | Escribe una parte del array. |
| `OutputStream` | `flush()` | Fuerza la salida de datos pendientes hacia el destino. |
| `OutputStream` | `close()` | Cierra el flujo; el cierre también fuerza la salida pendiente. |
| `Reader` | `read()` | Lee un carácter. Devuelve el carácter como `int` o `-1` al final. |
| `Reader` | `read(char[] cbuf)` | Lee varios caracteres en un array. |
| `Reader` | `close()` | Cierra el flujo de caracteres. |
| `Writer` | `write(String str)` | Escribe una cadena de caracteres. |
| `Writer` | `write(char[] cbuf)` | Escribe un array de caracteres. |
| `Writer` | `flush()` | Fuerza la escritura de los caracteres pendientes. |
| `Writer` | `close()` | Cierra el flujo de salida de caracteres. |

**No confundir:** los *streams* de `java.io` son flujos de entrada/salida. No son lo mismo que la API `java.util.stream` utilizada para procesar colecciones y secuencias de datos.

## 7. Flujos de bytes: `InputStream` y `OutputStream`

Los flujos de bytes trabajan con información binaria. Son adecuados para imágenes, PDF, ZIP, audio y, en general, cualquier contenido que deba tratarse como bytes.

### Leer bytes con `FileInputStream`

```
try (FileInputStream entrada = new FileInputStream("datos.bin")) {
    int dato;

    while ((dato = entrada.read()) != -1) {
        System.out.println(dato);
    }
}
```

`read()` devuelve un `int`: los valores 0–255 representan el byte leído y `-1` indica el final del flujo.

### Constructores y operaciones habituales de `FileInputStream`

| Clase | Método / constructor | Utilidad |
| --- | --- | --- |
| `FileInputStream` | `FileInputStream(String name)` | Abre para lectura el fichero indicado por su ruta. |
| `FileInputStream` | `FileInputStream(File file)` | Abre para lectura el fichero representado por un objeto `File`. |
| `FileInputStream` | `read()` | Lee un byte. |
| `FileInputStream` | `read(byte[] b)` | Lee un bloque de bytes. |
| `FileInputStream` | `read(byte[] b, int off, int len)` | Lee hasta `len` bytes en una parte del array. |
| `FileInputStream` | `close()` | Cierra el flujo de entrada. |

### Escribir bytes con `FileOutputStream`

```
try (FileOutputStream salida = new FileOutputStream("datos.bin")) {
    salida.write(65);
    salida.write(66);
    salida.write(67);
}
```

Por defecto, abrir un `FileOutputStream` sobre un fichero existente reemplaza su contenido. Para añadir al final:

```
new FileOutputStream("aplicacion.log", true)
```

### Constructores y operaciones habituales de `FileOutputStream`

| Clase | Método / constructor | Utilidad |
| --- | --- | --- |
| `FileOutputStream` | `FileOutputStream(String name)` | Abre el fichero para escritura; si existe, reemplaza su contenido. |
| `FileOutputStream` | `FileOutputStream(String name, boolean append)` | Si `append` es `true`, añade los datos al final. |
| `FileOutputStream` | `FileOutputStream(File file)` | Abre para escritura el fichero representado por un objeto `File`. |
| `FileOutputStream` | `write(int b)` | Escribe un byte. |
| `FileOutputStream` | `write(byte[] b)` | Escribe un bloque completo de bytes. |
| `FileOutputStream` | `write(byte[] b, int off, int len)` | Escribe una parte del array; es la forma usada al copiar por bloques. |
| `FileOutputStream` | `flush()` | Fuerza la escritura de los datos pendientes. |
| `FileOutputStream` | `close()` | Cierra el flujo de salida. |

### Copiar un fichero por bloques

```
try (
    FileInputStream entrada = new FileInputStream("foto.jpg");
    FileOutputStream salida = new FileOutputStream("copia.jpg")
) {
    byte[] buffer = new byte[8192];
    int bytesLeidos;

    while ((bytesLeidos = entrada.read(buffer)) != -1) {
        salida.write(buffer, 0, bytesLeidos);
    }
}
```

El último bloque puede contener menos bytes que el tamaño total del array. Por eso se escribe únicamente desde la posición 0 hasta `bytesLeidos`.

## 8. Flujos de caracteres: `Reader` y `Writer`

Cuando el contenido es texto, resulta más natural trabajar con caracteres que con bytes. Las familias `Reader` y `Writer` realizan esa abstracción.

### Lectura de texto

```
try (FileReader lector = new FileReader(
        "alumnos.txt", StandardCharsets.UTF_8)) {

    int caracter;

    while ((caracter = lector.read()) != -1) {
        System.out.print((char) caracter);
    }
}
```

### Operaciones habituales de lectura de texto

| Clase | Método / constructor | Utilidad |
| --- | --- | --- |
| `FileReader` | `FileReader(String fileName)` | Abre un fichero de texto para lectura. |
| `FileReader` | `FileReader(String fileName, Charset charset)` | Abre un fichero indicando explícitamente la codificación. |
| `FileReader` | `read()` | Lee un carácter. |
| `FileReader` | `read(char[] cbuf)` | Lee un bloque de caracteres. |
| `BufferedReader` | `BufferedReader(Reader in)` | Añade un búfer a otro `Reader`. |
| `BufferedReader` | `readLine()` | Lee una línea completa sin incluir el salto de línea; devuelve `null` al final. |
| `BufferedReader` | `read()` | Lee un carácter utilizando el búfer. |
| `BufferedReader` | `close()` | Cierra el lector y el recurso que envuelve. |

Para texto real, normalmente interesa leer por líneas utilizando `BufferedReader`:

```
try (BufferedReader lector = new BufferedReader(
        new FileReader("alumnos.txt", StandardCharsets.UTF_8))) {

    String linea;

    while ((linea = lector.readLine()) != null) {
        System.out.println(linea);
    }
}
```

### Escritura de texto

```
try (BufferedWriter escritor = new BufferedWriter(
        new FileWriter("alumnos.txt", StandardCharsets.UTF_8))) {

    escritor.write("Ana;24;8.5");
    escritor.newLine();
    escritor.write("Luis;31;7.2");
}
```

### Operaciones habituales de escritura de texto

| Clase | Método / constructor | Utilidad |
| --- | --- | --- |
| `FileWriter` | `FileWriter(String fileName)` | Abre un fichero de texto para escritura y reemplaza su contenido si existe. |
| `FileWriter` | `FileWriter(String fileName, boolean append)` | Permite añadir texto al final cuando `append` es `true`. |
| `FileWriter` | `FileWriter(String fileName, Charset charset)` | Permite indicar explícitamente la codificación. |
| `BufferedWriter` | `BufferedWriter(Writer out)` | Añade un búfer a otro `Writer`. |
| `BufferedWriter` | `write(String str)` | Escribe una cadena. |
| `BufferedWriter` | `newLine()` | Escribe el separador de línea adecuado para el sistema. |
| `BufferedWriter` | `flush()` | Fuerza la escritura de los caracteres pendientes. |
| `BufferedWriter` | `close()` | Cierra el escritor y el recurso que envuelve. |

### Codificación

Un carácter debe convertirse a bytes para almacenarse. La codificación determina esa conversión. En código actual conviene especificar explícitamente una codificación conocida, habitualmente UTF-8.

caracteres ── codificación UTF-8 ──► bytes ──► fichero
**La codificación también forma parte de NIO.**

`Charset` y `StandardCharsets` pertenecen a `java.nio.charset`. Esto recuerda que NIO no se limita a canales y buffers: también incluye las abstracciones utilizadas para convertir entre bytes y caracteres.

**Puente entre ambos mundos:** `InputStreamReader` convierte bytes de un `InputStream` en caracteres; `OutputStreamWriter` realiza la operación inversa.

### Puentes entre bytes y caracteres

| Clase | Método / constructor | Utilidad |
| --- | --- | --- |
| `InputStreamReader` | `InputStreamReader(InputStream in, Charset charset)` | Decodifica los bytes de un `InputStream` y los ofrece como caracteres. |
| `InputStreamReader` | `read() / read(char[] cbuf)` | Lee caracteres ya decodificados. |
| `OutputStreamWriter` | `OutputStreamWriter(OutputStream out, Charset charset)` | Codifica caracteres y los escribe sobre un `OutputStream`. |
| `OutputStreamWriter` | `write(String str)` | Escribe texto aplicando la codificación configurada. |
| `OutputStreamWriter` | `flush() / close()` | Fuerza la salida pendiente o cierra el escritor. |

### Combinar streams: cada clase añade una responsabilidad

Una de las ideas más importantes de `java.io` es que los flujos pueden **combinarse**. En lugar de buscar una única clase que haga todo, se encadenan varias clases y cada una se ocupa de una tarea concreta.

Lectura de un fichero de texto
Fichero (bytes)
↓
**FileInputStream** · obtiene bytes del fichero
↓
**InputStreamReader** · decodifica bytes → caracteres
↓
**BufferedReader** · añade búfer y `readLine()`
↓
Programa

```
try (BufferedReader lector = new BufferedReader(
        new InputStreamReader(
            new FileInputStream("datos.txt"),
            StandardCharsets.UTF_8))) {

    String linea;

    while ((linea = lector.readLine()) != null) {
        System.out.println(linea);
    }
}
```

La expresión se construye de dentro hacia fuera, pero los datos recorren la cadena desde el fichero hasta el programa:

| Capa | Responsabilidad | Tipo que entrega |
| --- | --- | --- |
| `FileInputStream` | Conecta con el fichero y lee sus bytes. | `InputStream` · bytes |
| `InputStreamReader` | Interpreta esos bytes con un `Charset` y produce caracteres. | `Reader` · caracteres |
| `BufferedReader` | Añade almacenamiento intermedio y operaciones cómodas como `readLine()`. | `Reader` · caracteres |

Escritura de un fichero de texto
Programa
↓
**BufferedWriter** · agrupa caracteres
↓
**OutputStreamWriter** · codifica caracteres → bytes
↓
**FileOutputStream** · escribe los bytes en el fichero
↓
Fichero (bytes)

```
try (BufferedWriter escritor = new BufferedWriter(
        new OutputStreamWriter(
            new FileOutputStream("datos.txt"),
            StandardCharsets.UTF_8))) {

    escritor.write("José");
    escritor.newLine();
}
```

**Regla para saber qué clases se pueden combinar:** hay que mirar el tipo que recibe el constructor. Por ejemplo, `BufferedReader` recibe un `Reader` e `InputStreamReader` es un `Reader`; por eso pueden encadenarse. En cambio, un `FileInputStream` es un `InputStream`, por lo que no puede pasarse directamente a `BufferedReader`: hace falta el puente `InputStreamReader`.

### Conversión de codificación

`InputStreamReader` y `OutputStreamWriter` son también las clases clave cuando necesitamos controlar la codificación. La primera **decodifica** bytes utilizando un juego de caracteres; la segunda **codifica** caracteres para producir bytes.

bytes ISO-8859-1 ──► InputStreamReader ──► caracteres Java ──► OutputStreamWriter ──► bytes UTF-8

Por tanto, también podemos usar ambas clases para **recodificar un fichero de texto**, leyendo con una codificación y escribiendo con otra:

```
try (
    BufferedReader lector = new BufferedReader(
        new InputStreamReader(
            new FileInputStream("origen.txt"),
            StandardCharsets.ISO_8859_1));

    BufferedWriter escritor = new BufferedWriter(
        new OutputStreamWriter(
            new FileOutputStream("destino.txt"),
            StandardCharsets.UTF_8))
) {
    String linea;

    while ((linea = lector.readLine()) != null) {
        escritor.write(linea);
        escritor.newLine();
    }
}
```

**Terminología correcta:** la clase de salida se llama `OutputStreamWriter`, no `OutputStreamReader`. La pareja es `InputStreamReader` / `OutputStreamWriter`.

## 9. Búferes de `java.io` y composición de streams

En este apartado hablamos del **buffering de los streams de `java.io`**. Un búfer es una zona temporal de memoria que permite agrupar operaciones de entrada/salida. En lugar de acceder continuamente al origen o destino para cantidades muy pequeñas de datos, se trabaja con bloques.

Fichero ──► FileInputStream ──► BufferedInputStream ──► Programa

Una característica importante de `java.io` es que las clases se pueden **envolver** unas dentro de otras para añadir funcionalidades.

```
BufferedInputStream entrada = new BufferedInputStream(
    new FileInputStream("datos.bin")
);
```

Aquí las responsabilidades son distintas:

- `FileInputStream`: conecta el programa con el fichero.
- `BufferedInputStream`: añade almacenamiento intermedio.

### Operaciones habituales de los flujos con búfer

| Clase | Método / constructor | Utilidad |
| --- | --- | --- |
| `BufferedInputStream` | `BufferedInputStream(InputStream in)` | Añade un búfer a un flujo de entrada de bytes. |
| `BufferedInputStream` | `read() / read(byte[] b)` | Lee bytes utilizando el almacenamiento intermedio. |
| `BufferedInputStream` | `mark(int readlimit)` | Marca una posición para poder volver a ella posteriormente. |
| `BufferedInputStream` | `reset()` | Vuelve a la última posición marcada, si sigue siendo válida. |
| `BufferedOutputStream` | `BufferedOutputStream(OutputStream out)` | Añade un búfer a un flujo de salida de bytes. |
| `BufferedOutputStream` | `write(int b) / write(byte[] b)` | Escribe datos primero en el búfer. |
| `BufferedOutputStream` | `flush()` | Vacía el búfer y fuerza el envío de los datos al destino. |
| `BufferedOutputStream` | `close()` | Cierra el flujo y escribe previamente los datos pendientes. |

**No confundir con `java.nio.Buffer`.**

`BufferedInputStream`, `BufferedReader`, etc. son **wrappers de streams** que gestionan internamente almacenamiento intermedio. En NIO, en cambio, un `ByteBuffer` es un objeto explícito cuyo estado (`position`, `limit`, `capacity`) controla el propio programa. El siguiente apartado desarrolla esa diferencia.

## 10. NIO aplicado a ficheros: entender `FileChannel` + `ByteBuffer`

Antes de aprender la mecánica fijamos el criterio de elección: **NIO no sustituye a `java.io`**. Con ficheros lo utilizaremos cuando necesitemos un control más explícito sobre **bloques de bytes, posiciones o transferencias**. Si una API de más alto nivel expresa directamente la operación, normalmente será más clara.

**Con ficheros, NIO no significa «no bloqueante».**

Una lectura con `FileChannel` puede bloquear mientras se completa. No elegimos `FileChannel` porque tengamos muchos ficheros que revisar, sino porque necesitamos **más control sobre cómo accedemos a sus datos**.

**Con `java.io`**
fichero ──► stream ──► programa

Consumimos o producimos datos como un flujo. Podemos añadir capas para leer líneas, usar búfer, interpretar tipos primitivos, etc.

**Con NIO aplicado a ficheros**
fichero ⇄ FileChannel ⇄ ByteBuffer ⇄ programa

El canal transfiere bloques de bytes y el programa controla explícitamente el estado del `ByteBuffer` y la posición del canal cuando lo necesita.

**La red es otro contexto.**

NIO también ofrece canales de red no bloqueantes y `Selector`, muy útiles para gestionar muchas conexiones. Lo mencionamos para entender el alcance de NIO, pero aquí estudiamos su aplicación a **ficheros**: `FileChannel` y `ByteBuffer`.

**La frase que debes poder explicar**

`FileChannel` **no devuelve directamente un `int`, una línea o un objeto**. Transfiere bytes. Esos bytes se colocan en un `ByteBuffer`, y después el programa decide cómo interpretarlos.

### 1. El canal: por dónde se mueven los bytes

Un `Channel` representa una conexión capaz de realizar operaciones de entrada/salida. Como aquí trabajamos con un fichero, utilizaremos su especialización `FileChannel`.

Al abrirlo indicamos qué operaciones permitimos mediante constantes como `READ`, `WRITE`, `CREATE` o `TRUNCATE_EXISTING`. Estas opciones pertenecen a `StandardOpenOption`, de la API de ficheros de NIO.2.

```
Path ruta = Path.of("datos.bin");   // NIO.2: identifica dónde está el fichero

try (FileChannel canal = FileChannel.open(
        ruta,
        StandardOpenOption.READ)) { // NIO: abre el canal de lectura

    // Aquí trabajaremos con el canal.
}
```

Observa ya una mezcla importante: `Path` y `StandardOpenOption` pertenecen a la API moderna de ficheros de NIO.2; `FileChannel` pertenece al modelo NIO. Se diseñaron para trabajar juntos.

### 2. El buffer: la bandeja de trabajo

```
ByteBuffer buffer = ByteBuffer.allocate(8); // reserva espacio para 8 bytes
```

Un `ByteBuffer` tiene memoria para los datos y, además, mantiene tres valores que indican qué parte de esa memoria estamos utilizando:

| Propiedad | Pregunta que responde | Con `allocate(8)` |
| --- | --- | --- |
| `capacity` | ¿Cuánto cabe como máximo? | 8 |
| `position` | ¿Dónde será la siguiente lectura o escritura? | 0 al empezar |
| `limit` | ¿Hasta dónde puedo trabajar en esta fase? | 8 al empezar |

Al crear el buffer:
[ _ _ _ _ _ _ _ _ ]
 ↑ position = 0 limit = 8 ↑
capacity = 8

### 3. La operación clave: llenar → girar → consumir → reutilizar

Imagina que el canal lee 5 bytes del fichero y los coloca en el buffer. Mientras **entra información en el buffer**, su `position` avanza:

después de leer 5 bytes:
[ A B C D E _ _ _ ]
 ↑ position = 5 limit = 8 ↑

Ahora queremos **sacar** esos cinco bytes. Pero `position` está en 5, preparado para seguir llenando. Necesitamos cambiar el buffer de «modo llenar» a «modo consumir»:

```
buffer.flip(); // ahora vamos a LEER del buffer lo que antes hemos ESCRITO en él
```

`flip()` hace conceptualmente dos cosas:

```
limit = position; // solo son válidos los 5 bytes que hemos cargado
position = 0;      // empezaremos a consumir desde el primero
```

después de flip():
[ A B C D E | _ _ _ ]
 ↑ position = 0 limit = 5 ↑

Cuando terminemos de consumir esos datos podemos preparar todo el espacio para volver a llenarlo:

```
buffer.clear(); // position = 0, limit = capacity; NO borra físicamente los bytes
```

| Método | Traducción mental | Cuándo |
| --- | --- | --- |
| `flip()` | «Ya he llenado; ahora quiero consumir». | Después de meter datos en el buffer y antes de sacarlos. |
| `clear()` | «Ya he terminado; puedo reutilizar toda la bandeja». | Cuando no necesitamos conservar datos pendientes. |
| `rewind()` | «Quiero volver a leer los mismos datos desde el principio». | Relectura sin cambiar el límite. |
| `compact()` | «Conservo lo que queda y libero espacio detrás». | Cuando una operación no ha consumido todo el contenido. |

### 4. Ejemplo completo: copiar una imagen por bloques

Ahora el patrón anterior aparece dentro de un programa real. Lee primero los comentarios: describen el ciclo del buffer.

```
Path origen = Path.of("foto.jpg");
Path destino = Path.of("copia.jpg");

try (
    // Canal por el que ENTRAN bytes desde el fichero origen.
    FileChannel entrada = FileChannel.open(origen, StandardOpenOption.READ);

    // Canal por el que SALEN bytes hacia el fichero destino.
    FileChannel salida = FileChannel.open(
        destino,
        StandardOpenOption.CREATE,
        StandardOpenOption.WRITE,
        StandardOpenOption.TRUNCATE_EXISTING)
) {
    // Nuestra "bandeja": puede contener hasta 8192 bytes cada vez.
    ByteBuffer buffer = ByteBuffer.allocate(8192);

    // 1. entrada.read(buffer) LLENA el buffer.
    //    La position del buffer avanza con los bytes recibidos.
    while (entrada.read(buffer) != -1) {

        // 2. Dejamos de llenar y preparamos el buffer para CONSUMIRLO.
        buffer.flip();

        // 3. Vaciamos TODO el contenido válido hacia el canal de salida.
        //    write() puede escribir menos bytes de los disponibles en una llamada.
        while (buffer.hasRemaining()) {
            salida.write(buffer);
        }

        // 4. Ya hemos consumido el bloque. Reutilizamos todo el buffer.
        buffer.clear();
    }
}
```

read(buffer) → **LLENAR**
↓
flip() → **CAMBIAR DE FASE**
↓
write(buffer) → **CONSUMIR**
↓
clear() → **REUTILIZAR**

### 5. Seguir el estado del buffer en una iteración

| Momento | position | limit | capacity | Qué significa |
| --- | --- | --- | --- | --- |
| Tras `allocate(8192)` | 0 | 8192 | 8192 | Todo el espacio está disponible. |
| Supongamos que `read()` carga 3000 bytes | 3000 | 8192 | 8192 | Hay 3000 bytes válidos al principio. |
| Tras `flip()` | 0 | 3000 | 8192 | Solo podremos consumir esos 3000 bytes. |
| Tras escribirlos todos | 3000 | 3000 | 8192 | No quedan bytes pendientes. |
| Tras `clear()` | 0 | 8192 | 8192 | Preparado para el siguiente bloque. |

### 6. ¿Por qué no hacemos siempre esto?

Porque si nuestra intención es simplemente «copiar un fichero», NIO.2 ya ofrece una operación de alto nivel:

```
Files.copy(origen, destino, StandardCopyOption.REPLACE_EXISTING);
```

El ejemplo con `FileChannel` no es «mejor» por ser más largo ni NIO es automáticamente «más rápido». Nos sirve para entender y controlar **cómo se transportan los bytes por bloques**. Elegiremos esa capa cuando necesitemos posiciones, mapeo de memoria, bloqueos, transferencias entre canales u otro control específico; si no necesitamos ese control, una API de más alto nivel suele expresar mejor nuestra intención.

### 7. Bytes no son caracteres

`ByteBuffer` contiene bytes. Para texto UTF-8 no debemos hacer simplemente `(char) buffer.get()`: algunos caracteres se representan mediante varios bytes. Para texto suele ser más sencillo utilizar `Reader`/`Writer`, `Files.newBufferedReader()` o una conversión explícita mediante `Charset`.

**NIO es más amplio que lo que veremos aquí.** En esta unidad nos interesa sobre todo `FileChannel`, `ByteBuffer`, posiciones y codificaciones. NIO también introdujo canales de red que pueden trabajar en modo no bloqueante y mecanismos como `Selector` para gestionar muchos canales de forma eficiente. Esa necesidad de escalabilidad ayuda a entender **por qué apareció NIO**, aunque no necesitemos desarrollar ahora esa parte para trabajar con ficheros.

## 11. Cierre de recursos y excepciones

Abrir un stream implica utilizar recursos del sistema operativo. Deben liberarse correctamente aunque se produzca un error durante la operación.

### Try-with-resources

La forma recomendada de gestionar estos recursos es **try-with-resources**:

```
try (BufferedReader lector = new BufferedReader(
        new FileReader("datos.txt", StandardCharsets.UTF_8))) {

    String linea;

    while ((linea = lector.readLine()) != null) {
        System.out.println(linea);
    }

} catch (IOException e) {
    System.err.println("Error al leer el fichero: " + e.getMessage());
}
```

Al terminar el bloque `try`, el recurso se cierra automáticamente, tanto si la ejecución termina normalmente como si se produce una excepción.

### IOException

Las operaciones de entrada/salida dependen del sistema de archivos y pueden fallar por causas externas al código:

- el fichero no existe;
- la ruta es incorrecta;
- faltan permisos;
- el dispositivo no está disponible;
- se produce un error durante la lectura o escritura.

**`IOException` es una checked exception.** Debe capturarse o propagarse. `FileNotFoundException` es una excepción más específica que hereda de `IOException`.

### Excepciones habituales de `java.io`

| Excepción | Significado habitual |
| --- | --- |
| `IOException` | Excepción general de entrada/salida. |
| `FileNotFoundException` | El fichero no existe o no puede abrirse. |
| `EOFException` | Se alcanza el final del fichero de forma inesperada en una lectura binaria estructurada. |
| `UTFDataFormatException` | Error en el formato UTF utilizado por operaciones como `readUTF()`. |
| `InterruptedIOException` | La operación de E/S ha sido interrumpida. |
| `NotSerializableException` | Se intenta serializar un objeto que no cumple los requisitos de serialización. |

## 12. Guardar tipos primitivos en binario

`DataOutputStream` y `DataInputStream` permiten escribir y leer directamente valores primitivos y cadenas en formato binario.

### Escritura

```
try (DataOutputStream salida = new DataOutputStream(
        new BufferedOutputStream(
            new FileOutputStream("alumno.dat")))) {

    salida.writeUTF("Ana");
    salida.writeInt(24);
    salida.writeDouble(8.5);
    salida.writeBoolean(true);
}
```

### Lectura

```
try (DataInputStream entrada = new DataInputStream(
        new BufferedInputStream(
            new FileInputStream("alumno.dat")))) {

    String nombre = entrada.readUTF();
    int edad = entrada.readInt();
    double nota = entrada.readDouble();
    boolean aprobado = entrada.readBoolean();
}
```

### Métodos más útiles de `DataInputStream` y `DataOutputStream`

| Clase | Método / constructor | Utilidad |
| --- | --- | --- |
| `DataInputStream` | `DataInputStream(InputStream in)` | Añade lectura de tipos Java sobre otro flujo de entrada. |
| `DataInputStream` | `readBoolean() / readByte() / readChar()` | Lee los tipos indicados. |
| `DataInputStream` | `readInt() / readLong()` | Lee valores enteros binarios. |
| `DataInputStream` | `readFloat() / readDouble()` | Lee valores reales binarios. |
| `DataInputStream` | `readUTF()` | Lee una cadena escrita mediante `writeUTF()`. |
| `DataInputStream` | `readFully(byte[] b)` | Intenta llenar completamente el array; si no hay suficientes bytes, lanza `EOFException`. |
| `DataOutputStream` | `DataOutputStream(OutputStream out)` | Añade escritura de tipos Java sobre otro flujo de salida. |
| `DataOutputStream` | `writeBoolean() / writeByte() / writeChar()` | Escribe los tipos indicados. |
| `DataOutputStream` | `writeInt() / writeLong()` | Escribe valores enteros en binario. |
| `DataOutputStream` | `writeFloat() / writeDouble()` | Escribe valores reales en binario. |
| `DataOutputStream` | `writeUTF(String s)` | Escribe una cadena en el formato que posteriormente entiende `readUTF()`. |
| `DataOutputStream` | `flush() / close()` | Fuerza la salida pendiente o cierra el flujo. |

**Regla fundamental:** los datos deben leerse en el mismo orden y con los mismos tipos con los que fueron escritos. El fichero no contiene automáticamente nombres de campos que permitan reconstruir el significado de cada valor.

## 13. Serialización de objetos

La serialización clásica de Java permite transformar el estado de un objeto en una secuencia de bytes para almacenarlo y recuperarlo posteriormente.

Objeto Java ── serialización ──► bytes ──► fichero

### Clase serializable

```
public class Alumno implements Serializable {

    private static final long serialVersionUID = 1L;

    private String nombre;
    private int edad;
    private double nota;

    // constructor, getters, setters...
}
```

### Escribir un objeto

```
Alumno alumno = new Alumno("Ana", 24, 8.5);

try (ObjectOutputStream salida = new ObjectOutputStream(
        new FileOutputStream("alumno.dat"))) {

    salida.writeObject(alumno);
}
```

### Leer un objeto

```
try (ObjectInputStream entrada = new ObjectInputStream(
        new FileInputStream("alumno.dat"))) {

    Alumno alumno = (Alumno) entrada.readObject();
}
```

### Operaciones habituales de serialización

| Clase | Método / constructor | Utilidad |
| --- | --- | --- |
| `ObjectOutputStream` | `ObjectOutputStream(OutputStream out)` | Crea un flujo capaz de serializar objetos sobre otro flujo de salida. |
| `ObjectOutputStream` | `writeObject(Object obj)` | Serializa y escribe un objeto. |
| `ObjectOutputStream` | `flush()` | Fuerza la escritura de los datos pendientes. |
| `ObjectOutputStream` | `close()` | Cierra el flujo de objetos y el flujo que envuelve. |
| `ObjectInputStream` | `ObjectInputStream(InputStream in)` | Crea un flujo capaz de deserializar objetos desde otro flujo de entrada. |
| `ObjectInputStream` | `readObject()` | Lee y reconstruye un objeto; devuelve una referencia de tipo `Object`. |
| `ObjectInputStream` | `close()` | Cierra el flujo de objetos y el flujo que envuelve. |

`transient` permite excluir un atributo de la serialización:

```
private transient String tokenSesion;
```

**Contexto:** la serialización nativa de Java es útil para comprender persistencia de objetos y aparece en código existente, pero no suele ser la primera opción para intercambio de datos entre aplicaciones. Para ello son habituales formatos definidos de forma explícita, como JSON, XML o formatos binarios específicos.

## 14. Acceso secuencial y acceso aleatorio: ¿por qué lo necesitamos?

Hasta ahora hemos leído o escrito normalmente avanzando desde el principio hacia el final. Ese patrón es **acceso secuencial** y resulta adecuado cuando necesitamos recorrer todos los datos.

dato 0 ──► dato 1 ──► dato 2 ──► dato 3 ──► ...

Pero pensemos en un fichero grande con miles de registros. Si conocemos qué registro queremos modificar, no tiene sentido leer todos los anteriores. En **acceso aleatorio** conocemos o calculamos una posición en bytes y saltamos directamente hasta ella.

inicio ─────────────────────► byte 4000
**La idea que hace posible el salto directo**

Si todos los registros ocupan lo mismo, podemos calcular dónde empieza cualquiera de ellos:

posición = númeroDeRegistro × tamañoDelRegistro

Si cada registro ocupa 40 bytes, el registro 0 empieza en el byte 0, el registro 1 en el 40, el registro 2 en el 80 y el registro 100 en el 4000.

`RandomAccessFile` mantiene un **puntero** que indica la posición actual dentro del fichero. Con `seek()` movemos ese puntero; la siguiente lectura o escritura comenzará desde allí.

```
try (RandomAccessFile fichero = new RandomAccessFile("alumnos.dat", "rw")) {

    long posicion = 100L * 40; // registro 100, 40 bytes por registro
    fichero.seek(posicion);

    int edad = fichero.readInt();
}
```

El ejemplo muestra la idea, no un formato completo de registro. En la práctica tendremos que conocer exactamente cuántos bytes ocupa cada campo para calcular correctamente el tamaño total.

### `RandomAccessFile` · java.io

```
fichero.seek(100);
fichero.writeInt(25);
```

Especialmente cómodo cuando conocemos la posición y queremos leer o escribir **tipos primitivos** directamente mediante métodos como `readInt()` o `writeDouble()`.

### `FileChannel` · NIO

```
canal.position(100);
canal.read(buffer);
```

También permite trabajar con posiciones, pero lo hace dentro del modelo NIO: `Channel` + `ByteBuffer`, apropiado cuando necesitamos control sobre bloques y transferencias.

**¿Por qué son tan importantes los registros de tamaño fijo?**

Porque permiten calcular directamente dónde empieza cada registro. Si cada registro ocupa un tamaño distinto, ya no podemos aplicar una multiplicación sencilla: necesitaremos recorrer información previa o mantener algún tipo de índice.

**El acceso aleatorio no significa buscar por valor.**

Java puede saltar al byte 4000 porque nosotros le damos esa posición. Para encontrar «el alumno Ana» necesitamos conocer previamente dónde está, mantener un índice o recorrer los datos.

## 15. Cómo elegir la API adecuada

La elección correcta no consiste en buscar «la clase más moderna», sino la abstracción que mejor expresa la operación.

| Necesidad | Herramienta recomendable | Por qué |
| --- | --- | --- |
| Representar una ruta | `Path` | API moderna de rutas. |
| Crear, copiar, mover, borrar, consultar | `Files` | Operaciones directas y expresivas. |
| Leer todo un texto pequeño/moderado | `Files.readString()` | La intención queda clara; carga todo en memoria. |
| Procesar texto por líneas | `Files.newBufferedReader()` / `BufferedReader` | No es necesario cargar el fichero completo. |
| Leer/escribir bytes como flujo | `InputStream` / `OutputStream` | Modelo sencillo y compatible con muchas APIs. |
| Tipos primitivos en formato binario propio | `DataInputStream` / `DataOutputStream` | Métodos tipados directos. |
| Control de bloques, posiciones o transferencias | `FileChannel` + `ByteBuffer` | Modelo NIO de menor nivel y mayor control. |
| Acceso directo a registros de posición conocida con operaciones tipadas | `RandomAccessFile` | Permite usar `seek()` y métodos tipados; encaja especialmente bien con registros de tamaño fijo. |
| Compatibilidad con una API antigua que exige `File` | `Path.toFile()` | Permite mantener `Path` como representación principal. |
| Gestionar muchas conexiones de red | NIO no bloqueante + `Selector` | Contexto de red: permite atender muchos canales sin dedicar necesariamente un hilo bloqueado a cada conexión. |

### Árbol de decisión rápido

```
¿Solo quiero trabajar con la ruta o el sistema de archivos?
├─ Sí → Path + Files
│
└─ No: quiero procesar contenido
   │
   ├─ Texto
   │  ├─ entero y pequeño/moderado → Files.readString / writeString
   │  └─ por líneas                → BufferedReader / BufferedWriter
   │
   └─ Binario
      ├─ flujo de bytes            → InputStream / OutputStream
      ├─ tipos primitivos          → DataInputStream / DataOutputStream
      └─ control de posición/bloques
         ├─ métodos tipados        → RandomAccessFile
         └─ modelo NIO             → FileChannel + ByteBuffer
```

## 16. `java.io` es una familia: varias formas de leer y escribir

Hasta ahora hemos utilizado algunas combinaciones concretas, pero `java.io` contiene muchas clases que se pueden **combinar**. No significa que haya una única forma correcta de leer o escribir un fichero. Cada clase aporta una responsabilidad distinta: trabajar con bytes, con caracteres, añadir búfer, leer líneas, interpretar tipos o facilitar la escritura de texto.

**Objetivo de este apartado**

No necesitas memorizar todas las combinaciones. Debes reconocerlas cuando aparezcan en código ajeno y entender **qué aporta cada capa**. Si ves una solución distinta a la utilizada en clase, no significa que sea otra API completamente diferente.

### A. Formas habituales de leer texto

| Clase / combinación | Qué aporta | Cuándo puede resultar cómoda |
| --- | --- | --- |
| `FileReader` | Lee caracteres desde un fichero. | Lectura sencilla de caracteres; suele envolverse con otra clase si queremos más comodidad. |
| `BufferedReader` | Añade búfer y `readLine()`. | Procesar texto línea a línea. |
| `Scanner` | Divide la entrada en elementos y puede interpretar `int`, `double`, etc. | Entrada textual estructurada y cómoda de analizar. |
| `InputStreamReader` + `InputStream` | Convierte bytes en caracteres usando una codificación. | Cuando partimos de un flujo de bytes y necesitamos interpretarlo como texto. |

Por ejemplo, estas dos construcciones no representan dos conceptos opuestos: la segunda simplemente añade una capa que permite leer líneas con comodidad.

```
Reader lector = new FileReader("datos.txt", StandardCharsets.UTF_8);

BufferedReader buffer = new BufferedReader(
    new FileReader("datos.txt", StandardCharsets.UTF_8)
);
```

### B. Formas habituales de escribir texto

| Clase / combinación | Qué aporta | Cuándo puede resultar cómoda |
| --- | --- | --- |
| `FileWriter` | Escribe caracteres en un fichero. | Escritura textual sencilla. |
| `BufferedWriter` | Añade búfer y operaciones como `newLine()`. | Escribir texto progresivamente de forma eficiente. |
| `PrintWriter` | Ofrece `print()`, `println()` y `printf()`. | Generar texto con una sintaxis cómoda y familiar. |
| `OutputStreamWriter` + `OutputStream` | Convierte caracteres en bytes usando una codificación. | Cuando partimos de un flujo de bytes y queremos escribir texto con un `Charset` concreto. |

### C. La misma idea también aparece en binario

| Necesidad | Clases habituales | Qué añaden |
| --- | --- | --- |
| Bytes sin interpretar | `FileInputStream` / `FileOutputStream` | Flujo básico de bytes. |
| Añadir almacenamiento intermedio | `BufferedInputStream` / `BufferedOutputStream` | Búfer gestionado internamente. |
| Tipos primitivos | `DataInputStream` / `DataOutputStream` | Métodos como `readInt()`, `writeDouble()`... |
| Objetos serializables | `ObjectInputStream` / `ObjectOutputStream` | Serialización nativa de objetos Java. |

**No memorices constructores: identifica responsabilidades.**

Cuando encuentres una cadena como `BufferedReader → InputStreamReader → FileInputStream`, léela por capas: el fichero proporciona bytes, `InputStreamReader` los convierte en caracteres y `BufferedReader` añade búfer y lectura por líneas. Esa composición es una de las ideas centrales de `java.io`.

## 17. Comparativa práctica: el mismo problema visto desde varias APIs

Después de ver que incluso dentro de `java.io` existen varias combinaciones, comparamos ahora familias distintas. Esta sección no pretende que memorices varias formas de hacer lo mismo: su objetivo es que veas **qué responsabilidad asume cada nivel de abstracción y por qué elegiríamos uno u otro**.

### A. Leer un fichero de texto línea a línea

**Con `java.io`** construimos explícitamente la cadena de clases:

```
try (BufferedReader lector = new BufferedReader(
        new FileReader("alumnos.txt", StandardCharsets.UTF_8))) {

    String linea;
    while ((linea = lector.readLine()) != null) {
        System.out.println(linea);
    }
}
```

**Con NIO.2** usamos `Path` para la ruta y `Files` para abrir el lector:

```
Path ruta = Path.of("alumnos.txt");

try (BufferedReader lector = Files.newBufferedReader(
        ruta, StandardCharsets.UTF_8)) {

    String linea;
    while ((linea = lector.readLine()) != null) {
        System.out.println(linea);
    }
}
```

**Qué debes observar:** NIO.2 no ha «eliminado» `BufferedReader`. Nos da una API moderna para localizar y abrir el fichero, y seguimos utilizando una clase de `java.io` para consumir texto por líneas.

### B. Leer un texto pequeño completo

```
String contenido = Files.readString(
    Path.of("alumnos.txt"),
    StandardCharsets.UTF_8
);
```

Aquí subimos de nivel: expresamos directamente «lee este fichero como texto». Es cómodo, pero el contenido completo queda en memoria. Por eso no es la elección automática para ficheros muy grandes.

### C. Copiar una imagen: tres formas, tres niveles

**Opción 1 · NIO.2: expresar directamente la intención**

```
Files.copy(
    Path.of("foto.jpg"),
    Path.of("copia.jpg"),
    StandardCopyOption.REPLACE_EXISTING
);
```

**Opción 2 · `java.io`: nosotros controlamos el bloque intermedio**

```
try (
    InputStream entrada = new FileInputStream("foto.jpg");
    OutputStream salida = new FileOutputStream("copia.jpg")
) {
    byte[] bloque = new byte[8192];
    int leidos;

    while ((leidos = entrada.read(bloque)) != -1) {
        salida.write(bloque, 0, leidos);
    }
}
```

**Opción 3 · NIO: canal + buffer explícito**

```
try (
    FileChannel entrada = FileChannel.open(Path.of("foto.jpg"), StandardOpenOption.READ);
    FileChannel salida = FileChannel.open(
        Path.of("copia.jpg"),
        StandardOpenOption.CREATE,
        StandardOpenOption.WRITE,
        StandardOpenOption.TRUNCATE_EXISTING)
) {
    ByteBuffer buffer = ByteBuffer.allocate(8192);

    while (entrada.read(buffer) != -1) { // llenar
        buffer.flip();                    // preparar para consumir

        while (buffer.hasRemaining()) {
            salida.write(buffer);         // consumir
        }

        buffer.clear();                   // preparar para volver a llenar
    }
}
```

| Solución | Qué escribimos nosotros | Qué aprendemos / controlamos |
| --- | --- | --- |
| `Files.copy()` | La intención: copiar. | Máxima claridad; pocos detalles internos. |
| `InputStream` / `OutputStream` | Bucle y `byte[]`. | Modelo secuencial de streams. |
| `FileChannel` / `ByteBuffer` | Bucle y estado del buffer. | Modelo NIO y control explícito sobre bloques/posición. |

**No hay una competición entre APIs.** Si la operación es «copiar», `Files.copy()` es normalmente la forma más expresiva. Estudiamos las otras formas porque enseñan modelos de E/S que necesitaremos en problemas donde sí haga falta ese control.

## Resumen esencial

|  |  |
| --- | --- |
| **`Path`** | Representa una ruta. Crear el objeto no crea el fichero. |
| **`Files`** | Opera sobre rutas: crear, copiar, mover, leer, escribir, consultar atributos... |
| **`File`** | API clásica de rutas; útil para código existente y compatibilidad. |
| **`InputStream / OutputStream`** | Modelo `java.io` para bytes. |
| **`Reader / Writer`** | Modelo de caracteres; la codificación conecta caracteres con bytes. |
| **Buffered...** | Wrappers de `java.io` que añaden almacenamiento intermedio y operaciones cómodas. |
| **`Scanner / PrintWriter`** | Alternativas cómodas para analizar entrada textual y producir texto; forman parte de la amplitud de la API clásica. |
| **`RandomAccessFile`** | Acceso directo a posiciones conocidas con operaciones tipadas; especialmente útil con registros de tamaño fijo. |
| **`FileChannel`** | Canal NIO para leer, escribir y posicionarse en un fichero. |
| **`ByteBuffer`** | Buffer NIO explícito con `position`, `limit` y `capacity`. |
| **`flip()` / `clear()`** | `flip()` prepara para consumir datos; `clear()` prepara para reutilizar el buffer y no borra físicamente su contenido. |
| **Data...** | Lee/escribe tipos primitivos en formato binario. |
| **Try-with-resources** | Cierra automáticamente streams, canales y otros recursos `AutoCloseable`. |

**Continuación**

La Guía II profundiza en el **acceso aleatorio**, compara `RandomAccessFile` con `FileChannel` y aplica estas ideas a registros binarios de tamaño fijo.

Ficheros en Java · Bloque 1 · java.io, NIO y NIO.2
