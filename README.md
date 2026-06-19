2do Parcial - Parte Practica
Que se solicita:

El codigo tiene 10 errores. Recae en usted analizar que es un error dentro del codigo.
Los Alumnos tendran que forkear este repo como propio, hacer un issue desde Github con Comentarios refiriendo en que linea esta el error, y como se debe solucionar.
La respuesta sera con el link a ese Fork, y adentro deben estar los issues. Los profesores tenemos que poder ingresar al mismo. Recae en los alumnos asegurarse de que los profesores puedan ingresar.
Tambien pueden editar el Archivo Readme y poner los resultados dentro de sus propios forks.
https://github.com/ExBattou/SimpsonsApp

# Reporte de errores detectados en SimpsonsApp

## Resumen

Durante la revisión se detectaron problemas de configuración, arquitectura, consumo de API, null-safety y código inválido. 

---

# 1. URL completa dentro de @GET

## Archivo
`app/src/main/java/com/example/simpsonsapp/data/remote/EpisodeRemoteMediator.kt`

## Línea archivo
106

## Código encontrado

```kotlin
@GET("https://thesimpsonsapi.com/api/episodes")
```

## Problema

Retrofit no debe recibir una URL completa en cada endpoint cuando ya existe una `baseUrl`.

Esto dificulta:
- mantenimiento
- reutilización
- testing
- cambios de ambiente (dev/prod)

Además rompe el patrón esperado de Retrofit.

## Solución

### Reemplazar código por

```kotlin
@GET("episodes")
```

### Retrofit deberá tener la base URL completa

---

# 2. Base URL faltante en Retrofit

## Archivo
`app/src/main/java/com/example/simpsonsapp/di/DataModule.kt`

## Línea código
35

## Problema

Retrofit necesita una URL base válida.
Sin ella el cliente HTTP no puede construir de forma correcta las peticiones.

## Solución

Agregar:

```kotlin
Retrofit.Builder()
    .baseUrl("https://thesimpsonsapi.com/api/")
    .client(client)
    .addConverterFactory(GsonConverterFactory.create())
    .build()
```

---

# 3. Uso inseguro de episode!!

## Archivo
`app/src/main/java/com/example/simpsonsapp/detail/DetailScreen.kt`

## Líneas aproximadas
51-64

## Código encontrado

```kotlin
if (episode != null) {

    EpisodeCard(
        episode = episode!!,
        onClick = {},
        isDetailMode = true
    )
}
```

## Problema

El operador `!!` puede lanzar:

```text
NullPointerException
```

si en algún momento la condición deja de cumplirse o cambia el flujo.

Además Kotlin provee mecanismos más seguros.

## Solución

```kotlin
episode?.let { selectedEpisode ->

    EpisodeCard(
        episode = selectedEpisode,
        onClick = {},
        isDetailMode = true
    )
}
```

## Beneficios

- Null Safety
- Código más idiomático
- Evita crashes

---

# 4. Código inválido dentro de data class

## Archivo
`app/src/main/java/com/example/simpsonsapp/domain/model/Episode.kt`

## Líneas aproximadas
11-15

## Código encontrado

```kotlin
init {
    return Episode; // NO BORRAR
}
```

## Problema

Un bloque `init` no puede devolver valores ni contener `return`.
Esto produce error de compilación.

## Solución

Eliminar completamente:

```kotlin
init {
    return Episode
}
```

### Resultado esperado

```kotlin
data class Episode(
    val id: Int,
    val airdate: String,
    val episodeNumber: Int,
    val imagePath: String,
    val name: String,
    val season: Int,
    val synopsis: String
)
```

---

# 5. Inconsistencia en nombres del repositorio

## Archivos

- `EpisodeRepository.kt`
- `GetEpisodesUseCase.kt`
- `EpisodeRepositoryImpl.kt`

## Líneas aproximadas
8
13
11

## Problema

Se detectó coexistencia de nombres:

```kotlin
get_episodes()
```

y

```kotlin
getEpisodes()
```

## Impacto

Provoca:

- errores de compilación
- referencias rotas
- inconsistencia de estilo

## Solución

Unificar en camelCase:

```kotlin
fun getEpisodes()
```

### Repositorio

```kotlin
interface EpisodeRepository {
    fun getEpisodes()
}
```

### Caso de uso

```kotlin
class GetEpisodesUseCase(
    private val repository: EpisodeRepository
) {

    operator fun invoke() =
        repository.getEpisodes()
}
```

---

# 6. Import faltante de SimpsonsApi

## Archivo

`EpisodeRepositoryImpl.kt`

## Líneas aproximadas
11

## Problema

Se utiliza `SimpsonsApi` pero no estaba importado.

## Solución

```kotlin
import com.example.simpsonsapp.data.remote.SimpsonsApi
```

---

# 7. Error arquitectónico en MainScreen

## Archivo

`MainScreen.kt`

## Problema detectado

Se estaba utilizando un contenedor de desplazamiento horizontal (LazyRow) para la disposición de los elementos de una lista que, por diseño y modificadores, requiere una estructura de paginación vertical. Esto rompía la visualización de los elementos y afectaba el rendimiento.

## Solución recomendada

Usar:

```kotlin
LazyColumn
```

Ejemplo:

```kotlin
// Cambios en los Imports
- import androidx.compose.foundation.layout.fillMaxHeight
        - import androidx.compose.foundation.layout.width
        - import androidx.compose.foundation.lazy.LazyRow
        + import androidx.compose.foundation.lazy.LazyColumn

// Dentro de fun MainScreen( ... )
// // Main List
        Box(modifier = Modifier.fillMaxSize()) {
            -   LazyRow(
                +   LazyColumn(
                    state = listState,
                    modifier = Modifier.fillMaxSize()
                ) {
                    // ... Estructura de items ...

                    if (episodes.loadState.append is LoadState.Loading) {
                        item {
                            Box(
                                modifier = Modifier
                                        -                       .fillMaxHeight()
                                        -                       .width(100.dp)
                                        +                       .fillMaxWidth()
                                        +                       .padding(16.dp)
                            )
                        }
                    }
                }
        }
```

---

# 8. Configuración incorrecta del JDK

## Archivo

`gradle.properties`

## Líneas código

31, 32

## Problema

Gradle apuntaba a una ruta inválida.

```properties
# JDK 17
org.gradle.java.home=/opt/homebrew/Cellar/openjdk@17/17.0.15/libexec/openjdk.jdk/Contents/Home
```

## Síntoma

```text
Could not determine Java version
```

y

```text
JAVA_HOME is invalid
```
al ejecutar el código.

## Solución

Apuntar al JDK instalado.

Ejemplo:

```properties
# JDK 17 (use the Android Studio bundled JBR on Windows)
org.gradle.java.home=C:/Program Files/Android/Android Studio/jbr
```

o eliminar la propiedad y utilizar el JDK configurado en Android Studio.

---

# 9. Posible mezcla de responsabilidades

## Archivos involucrados

- EpisodeRemoteMediator.kt
- EpisodeRepository.kt

## Observación

Durante la revisión aparecieron varias referencias cruzadas entre:

- Paging
- Room
- Retrofit
- Casos de uso

Conviene verificar que:

### Repository

Sólo exponga:

```kotlin
Flow<PagingData<Episode>>
```

### ViewModel

Sólo gestione estado UI.

### RemoteMediator

Sólo sincronice:

```text
API ↔ Room
```

---

# 10. Código residual de ejemplo

## Archivo
`Episode.kt`

## Línea archivo
14

## Evidencia

La presencia de:

```kotlin
// NO BORRAR
```

junto a:

```kotlin
return Episode;
```

indica código temporal dejado accidentalmente.

## Recomendación

Eliminar:

- código de prueba
- comentarios temporales
- snippets de debugging

antes de entregar el proyecto.

---

# 11 / Bonus: Mal uso de Use Cases

## Archivos
`GetEpisodeDetailUseCase.kt`
`GetEpisodesBySeasonUseCase.kt`
`GetEpisodesUseCase.kt`
`GetSeasonsUseCase.kt`

## Problema

Los use cases tienen sentido si el objetivo es encapsular lógica de negocio, pero no deberían usarse como un contenedor genérico para cualquier llamada al repositorio.
Usarlo como wrapper vacío sí puede ser una mala decisión. Si no agrega lógica, no es necesario agregarlo.

## Recomendación

### Opción 1: dejar el use case y darle una responsabilidad real

Por ejemplo:
Decidir si usar lista completa o por temporada,
Aplicar filtros,
Ordenar o transformar resultados antes de mostrarlos.

### Opción 2: eliminar el use case si no agrega valor

Entonces la VM puede llamar directamente al repositorio.
Esto es válido si la app es pequeña y la separación no compensa la complejidad extra.

---