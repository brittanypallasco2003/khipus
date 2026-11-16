# CLAUDE.md

Este archivo ofrece orientación a Claude Code (claude.ai/code) para trabajar con el código de este repositorio.

## Proyecto

Khipus: multiproyecto Gradle en fase inicial (Java 25, Spring Boot 4.1.2, Spring Cloud 2025.1.4). Group `com.khipus`, versión `0.0.1-SNAPSHOT`. Gradle wrapper 9.8.2.

## Repositorios

Este directorio es el **repo raíz** (paraguas): contiene el build compartido (`settings.gradle`, `build-logic/`, wrapper) y referencia cada módulo como **submódulo git**:

| Ruta                  | Repo                                                       |
|-----------------------|------------------------------------------------------------|
| `libs`                | `https://github.com/brittanypallasco2003/khipus-libs.git`       |
| `services/accounting` | `https://github.com/brittanypallasco2003/khipus-accounting.git` |
| `services/invoicing`  | `https://github.com/brittanypallasco2003/khipus-invoicing.git`  |

- Clonar todo: `git clone --recurse-submodules <url-raíz>`; en un clon existente, `git submodule update --init --recursive`.
- Un cambio dentro de un submódulo se commitea y se sube **en el submódulo primero**; después, en el raíz, `git add <ruta>` + commit para mover el puntero. Si se sube el raíz sin haber subido el submódulo, los demás no podrán hacer checkout de ese commit.
- Los submódulos no se compilan solos: dependen de `build-logic/` y `settings.gradle` del raíz. Construir siempre desde el raíz.
- Para añadir un servicio nuevo como repo propio: `git submodule add <url> services/<nombre>` (settings lo descubre solo, ver Arquitectura).

## Comandos

Ejecutar desde la raíz del repo con el wrapper raíz (`./gradlew`). `libs/` y los servicios traen su propio `gradlew` sobrante de Spring Initializr (en los servicios ni siquiera existe `gradle/wrapper/`); usar siempre el del raíz.

```bash
./gradlew build                      # compila + prueba todos los proyectos
./gradlew :libs:test                 # tests de la librería commons
./gradlew :libs:test --tests 'com.khipus.commons.KhipusCommonsApplicationTests'
./gradlew :<service>:bootRun         # ejecuta un servicio (nombre del servicio = su directorio bajo services/)
./gradlew :<service>:bootBuildImage  # imagen OCI con buildpacks (requiere Docker)
./gradlew :<service>:nativeCompile   # imagen nativa GraalVM (requiere GraalVM 25+ y que el servicio aplique khipus.native-conventions)
./gradlew :<service>:nativeTest      # ejecuta los tests como imagen nativa
```

Native Build Tools ignora los toolchains de Gradle: la compilación nativa usa el JDK que ejecuta Gradle.

## Arquitectura

- `build-logic/`: build incluido (`pluginManagement { includeBuild('build-logic') }`) con **convention plugins** en `src/main/groovy/`. Es el único sitio donde se fijan las versiones de los plugins externos (en `build-logic/build.gradle`) y la de Spring Cloud (en `khipus.spring-conventions`). Los plugins disponibles son:
  - `khipus.java-conventions`: `java`, group/version, toolchain Java 25 y JUnit Platform.
  - `khipus.spring-conventions`: lo anterior + dependency-management con los BOM de Spring Boot y Spring Cloud, **sin** el plugin de Boot (jar plano, para librerías).
  - `khipus.spring-boot-conventions`: lo anterior + `org.springframework.boot` (bootJar/bootRun). `bootJar` se salta si el proyecto no tiene código en `src/main` (servicio recién creado).
  - `khipus.native-conventions`: GraalVM Native Build Tools (1.1.14, la versión que gestiona Spring Boot 4.1.2). Pensado para servicios, no para `libs`.
- Los subproyectos aplican los convention plugins por id y **nunca declaran versiones de plugins, `group`, `version`, toolchain ni `repositories {}`** (todo eso lo ponen las convenciones). El `build.gradle` raíz está vacío a propósito; no se usan `allprojects`/`subprojects`.
- `settings.gradle`: conecta `build-logic`, centraliza los repositorios (`FAIL_ON_PROJECT_REPOS`: declarar `repositories {}` en un `build.gradle` rompe el build), incluye `libs` y **descubre automáticamente cada directorio bajo `services/` como subproyecto** (nombre del proyecto = nombre del directorio). Si un directorio de servicio no tiene `build.gradle`, la evaluación de settings **genera** uno que aplica `khipus.spring-boot-conventions`. Para añadir un servicio, crear `services/<nombre>/` (o añadirlo como submódulo); no registrarlo manualmente.
- `libs/` (`:libs`): "Khipus Commons", la librería compartida. Incluye GraphQL, WebFlux, WebMVC, springdoc-openapi, OpenFeign y Lombok. Aplica solo `khipus.spring-conventions`: produce un jar plano, sin clase `main`. Para `@SpringBootTest` existe `KhipusCommonsTestApplication` en `src/test`, y `application.properties` vive en `src/test/resources` (una librería no debe imponer configuración a los servicios). Sus dependencias son `implementation`: si un servicio necesita ver esas APIs al compilar, aplicar `java-library` en `libs` y pasarlas a `api`.
- `services/`: `accounting` e `invoicing`, generados con Spring Initializr (Flyway + PostgreSQL, GraphQL, sesiones en Redis, Lombok, devtools). Solo tienen la clase `*Application` y su test de contexto.

## Estado actual / pendientes

- Los tests `contextLoads` de `accounting` e `invoicing` fallan (`DataSourceBeanCreationException`): no hay `spring.datasource.url` ni base de datos de prueba. `./gradlew build` falla por eso; compilar y empaquetar sí funciona (`./gradlew assemble`).
- Un servicio aplica `khipus.spring-boot-conventions`, no `khipus.spring-conventions` (sin el plugin de Boot no existen `bootJar`/`bootRun` ni la configuración `developmentOnly`).
- Los servicios usan el paquete y group `com.kipus` (sin "h"); el resto del proyecto usa `com.khipus`.
- Git: el repo raíz (rama `master`, sin commits ni remoto) aún no registra los submódulos; `libs`, `accounting` e `invoicing` son repos anidados con un commit en `main` ya subido a su remoto. `.gitmodules` está escrito a mano y tiene una entrada errónea (`third_submodule` → `path/to/third_submodule` en vez de `libs`).
- Si un servicio combina springdoc + `khipus.native-conventions` sin actuator, la compilación AOT avisa de que falta `org.springframework.boot.actuate.autoconfigure.web.server.ManagementPortType` (springdoc usa `@ConditionalOnManagementPort`). Es inocuo; añadir `spring-boot-starter-actuator` al servicio debería eliminarlo (no verificado).
