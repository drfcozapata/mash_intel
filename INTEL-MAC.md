# Mash para Mac Intel (x86_64)

Este documento resume los cambios de la rama `intel-support` de este fork, que
permiten **compilar y ejecutar Mash en Macs con procesador Intel**. La versión
oficial de Mash solo publica un DMG para Apple silicon (arm64) y paquetes `.deb`
para Linux, y `packaging/build.sh` se negaba a ejecutarse fuera de arm64.

> **Estado: experimental y no oficial.** No hay DMG ni release para Intel: la
> app se compila desde el código fuente. No está firmada con Developer ID ni
> notarizada, y el auto-update está desactivado. Mash sigue siendo el proyecto de
> [LuisAlejandro/mash](https://github.com/LuisAlejandro/mash); este fork solo
> añade el soporte de arquitectura descrito aquí.

- Versión base: **0.3.0** (`VERSION` no se modifica).
- Probado el 2026-11-05 en un MacBook Pro con Intel Core i9 y macOS 26.
- Mínimo declarado por la app: macOS 14.

## Qué cambia respecto a la versión oficial

| Área | Versión oficial (arm64) | Esta rama en Intel |
| --- | --- | --- |
| Arquitectura del build | `macos-arm64` | `macos-amd64` (y arm64 sigue intacto) |
| Distribución | DMG firmado y notarizado | `dist/Mash.app` firmada ad-hoc, compilada localmente |
| Auto-update | Activo | **Desactivado** (no existe canal de releases para Intel) |
| Notificaciones | `terminal-notifier` arm64 del plugin | `terminal-notifier` 2.0.0 x86_64 fijado en el manifiesto |
| `sentrux` | Disponible | **No disponible** (su upstream solo publica arm64) |
| `latexindent` | Disponible | **No disponible** (su upstream solo publica arm64) |
| Dictado de voz | `transcribe-cpp` con Metal | `transcribe-cpp` solo CPU (`darwin-x64-cpu`) |

## Cambios por área

### 1. Target `macos-amd64` en el manifiesto

- `packaging/assets.json`: nuevo target `macos-amd64` en `targets` y
  `supportedTargets`, y artefactos x86_64 con `url`, `sha256`, `archive` y
  `member` para: `herdr`, `node`, `python`, `lazygit`, `lazydocker`, `pet`,
  `viu`, `atuin`, `gh`, `starship`, `rust`, `go`, `rust-analyzer`, `marksman`,
  `texlab` y `cua-driver`. Ghostty añade `frontends.macos-amd64: swift`.
- Los `sha256` se contrastaron con los checksums publicados por cada proveedor
  (API de GitHub, `SHASUMS256.txt` de Node, `go.dev`, `.sha256` de Rust). `pet` y
  `terminal-notifier` (releases antiguas) no publican digest.
- `marksman` reutiliza el mismo binario que arm64: es universal (x86_64 + arm64).
- `herdr` usa el binario de release `herdr-macos-x86_64`, que solo enlaza
  librerías del sistema; en arm64 se sigue usando el bottle de Homebrew.
- `packaging/pins.txt` se regeneró con `assets.py pins` (añade `terminal-notifier`).
- `packaging/assets.py`: comando nuevo `applies <componente> <target>`. Devuelve
  `0` si el componente aplica al target, `4` si el manifiesto lo excluye de ese
  target y `1` si es desconocido.

### 2. Build (`packaging/build.sh`, `packaging/carry.sh`)

- `build.sh` acepta `arm64` y `x86_64`, y los targets `macos-arm64` y
  `macos-amd64`.
- Herdr: si la URL no es del registro de Homebrew (`ghcr.io`), se descarga el
  binario con `mash_fetch_pinned` en lugar de extraerlo de un bottle.
- Zig: la URL de ayuda depende de la arquitectura y, si no existe
  `/Volumes/Documents/Software/tools/zig-0.16.0/zig`, se usa
  `~/Development/tools/zig-0.16.0/zig`.
- Ghostty (`mash_build_ghostty`), **solo en `macos-amd64`**: se compila con
  `-Dxcframework-target=native` (la mitad del trabajo de Zig frente al universal)
  y `xcodebuild` recibe `ARCHS=x86_64 ONLY_ACTIVE_ARCH=YES`. Hace falta fijar la
  arquitectura porque el proyecto Xcode solo fija `ONLY_ACTIVE_ARCH` en Debug, y
  un Release intentaría enlazar también arm64 contra un framework x86_64.

### 3. Notificaciones

El plugin `herdr-terminal-notifier` trae su `terminal-notifier` compilado solo
para arm64. Se añadió el componente `terminal-notifier` 2.0.0 (release oficial de
`julienXX/terminal-notifier`, x86_64) al manifiesto, y `mash_build_plugins`
sustituye el ejecutable del `HerdrNotify.app` cuando el manifiesto fija uno para
el target. Es la misma versión (2.0.0) que usa el plugin en arm64; la firma
ad-hoc se rehace como antes.

### 4. Auto-update desactivado

- `build.sh` escribe `Contents/Resources/updates-disabled` cuando el target no es
  `macos-arm64`.
- `packaging/helpers/mash-update-notice` respeta ese marcador: no consulta la API
  de GitHub ni ofrece descargas. Es un único interruptor para la comprobación
  automática (la lanza `herdr-pane`) y la manual (`Cmd+Shift+U`, menú "Check for
  Updates…"); la manual muestra "Updates are disabled in this build".
- `packaging/plugins/mash-update-notice/notice.sh` pinta ese estado.
- `packaging/helpers/mash-update-processes.c` buscaba solo la porción arm64 de un
  ejecutable Mach-O; ahora usa la del host (`__arm64__` o x86_64).
- `mash-update-verify` sigue nombrando `Mash-<v>-mac-arm64.dmg`, pero es
  inalcanzable porque el aviso de actualización nunca se muestra.

### 5. Herramientas no disponibles en Intel y arreglo del asistente

`sentrux` y `latexindent` solo publican binario arm64 en macOS, así que el
manifiesto las excluye de `macos-amd64` con `targets`. Eso destapó un fallo: la
página de herramientas las marca por defecto y el asistente de primer arranque
trataba cualquier salida no cero de `mash-tool-install` como error fatal, lo que
habría detenido el paso "Install pinned tools" en Intel.

- `packaging/helpers/mash-tool-install`: sale con código **4** ("no disponible en
  esta plataforma") cuando `assets.py applies` lo indica; cualquier otro fallo
  sigue saliendo con 1.
- `packaging/wizard/main.rs`: el código 4 es un aviso no fatal
  (`ToolOutcome::Unavailable`). El resumen lo nombra, por ejemplo:
  `3 installed, 0 provided by the system, not available for this platform: latexindent, sentrux`.

Efecto para el usuario:

- Sin `sentrux` se pierde el *gate* de análisis estructural y su servidor MCP. Los
  skills de planificación, trabajo y QA lo contemplan y registran
  `SENTRUX: unavailable`. La entrada `sentrux` de `mcp.json` apunta a un binario
  inexistente, igual que si el usuario desmarcara la herramienta.
- Sin `latexindent` no se formatea LaTeX al guardar en Fresh; `texlab` funciona.
  Alternativa: `brew install latexindent`.

### 6. Pruebas

- `packaging/tests/test_assets.py`: `macos-amd64` entra en `ALL_TARGETS`; nuevas
  pruebas de que Intel nunca resuelve un artefacto arm64, de las exclusiones y del
  comando `applies`.
- `packaging/tests/test_mash_update_notice.sh`: casos del marcador (automático y
  manual). Se comprobó que fallan contra el helper original.
- `packaging/helpers/tests/test-mash-tool-install.sh`: casos del código 4 y del
  código 1; el bloque con red real entiende los componentes excluidos.
- `packaging/wizard/main.rs`: 3 pruebas unitarias de `tool_outcome` y
  `tools_summary`.

## Cómo compilar

Requisitos fuera del repositorio (rutas por defecto que espera `build.sh`):

| Requisito | Detalle |
| --- | --- |
| Xcode completo | Probado con Xcode 26.6 y su Metal Toolchain (`xcodebuild -downloadComponent MetalToolchain`) |
| Zig 0.16.0 | En `~/Development/tools/zig-0.16.0/zig` o en `MASH_ZIG`. Versión exacta obligatoria |
| Fork de Ghostty | `LuisAlejandro/ghostty` en el commit `6bbdef7`, en `~/Development/ghostty` o en `MASH_GHOSTTY_SRC` |
| Skills | `LuisAlejandro/mash-programming-skills` 0.6.0, en `~/Development/mash-programming-skills` o en `MASH_SKILLS_SRC` |
| `git-lfs` | Algunos plugins guardan imágenes en LFS; con `filter.lfs.required=true` en `~/.gitconfig`, git falla sin él |
| Toolchains | Rust y Go se descargan fijados por el propio build |

```sh
caffeinate -dimsu bash packaging/build.sh        # genera dist/Mash.app
RUST_TEST_THREADS=4 make test                    # ver nota abajo
```

- Duración orientativa en un i9-9980HK: ~45 min en frío; `make test`, ~5 min.
- `build.sh` borra `dist/.build` al empezar (vuelve a descargar toolchains) pero
  conserva `dist/.cargo-target` si falla, así que un reintento recompila poco.
- `build.sh` aplica de forma idempotente los parches de menú a la copia del fork
  de Ghostty (`mash-ghostty-patch-menu`), por lo que esa copia queda modificada.
- Homebrew ya no publica bottles para Intel: instalar herramientas (por ejemplo
  `git-lfs`) compila desde fuente.

## Verificación realizada

- `make test`: pasan 64 suites, 0 fallos; 4 se omiten por no haber `herdr` ni Pi
  en el PATH.
- `build.sh` termina con `exit 0`. En `dist/Mash.app` los 47 binarios Mach-O
  incluyen x86_64 (los únicos con arm64 son prebuilds de paquetes npm que traen
  además su gemelo x64), la firma ad-hoc verifica con `--deep --strict`, y herdr,
  node, python, gh, lazygit, starship, atuin, viu y fresh ejecutan con las
  versiones fijadas.
- Primer arranque real: el asistente completó Homebrew, npm, Python y CUA Driver,
  y el paso de herramientas fijadas terminó con
  `3 installed … not available for this platform: latexindent, sentrux`, sin
  abortar.

## Limitaciones y puntos sin verificar

- **Pruebas con timeout:** con los hilos por defecto, 2 tests de
  `provider_page` (`catalog_flow_keeps_credential_provider_separate_from_fallback_model`
  y `custom_api_reaches_the_stdin_bridge_and_compact_mouse_target`) fallan en un
  portátil de 16 hilos porque esperan un `node` falso con un plazo de 3 s. Fallan
  igual en el código original; con `RUST_TEST_THREADS=4` pasan los 199.
- **Sin verificar al redactar esto:** instalación de memoria (Hindsight), dictado
  de voz (solo CPU, previsiblemente más lento que con Metal), entrega real de
  notificaciones y atajos de teclado.
- No hay CI para Intel: los workflows del repositorio siguen siendo arm64 y Linux.
- Cada actualización de versión de un componente requiere renovar a mano su
  artefacto y hash de `macos-amd64`.

## Mantenimiento y sincronización con upstream

```sh
git remote add upstream https://github.com/LuisAlejandro/mash.git
git fetch upstream
git rebase upstream/develop      # o merge, según prefieras
```

Los conflictos más probables están en `packaging/assets.json` (versiones nuevas
de componentes), `packaging/build.sh` y `packaging/carry.sh`. Tras resolverlos:

```sh
python3 packaging/assets.py validate
python3 packaging/assets.py pins > packaging/pins.txt
python3 packaging/tests/test_assets.py
```

Si upstream sube la versión de un componente, añade su artefacto x86_64 al
bloque `macos-amd64` con el mismo `member` ajustado a la arquitectura y contrasta
el `sha256` con el publicado por el proveedor.

## Archivos modificados

| Archivo | Cambio |
| --- | --- |
| `packaging/assets.json`, `packaging/pins.txt` | Target `macos-amd64`, artefactos x86_64, componente `terminal-notifier`, exclusiones |
| `packaging/assets.py` | Comando `applies` |
| `packaging/build.sh` | Guardas de arquitectura, Herdr x86_64, Zig, marcador `updates-disabled` |
| `packaging/carry.sh` | Ghostty nativo en Intel y sustitución del binario del notifier |
| `packaging/helpers/mash-update-notice` | Respeta `updates-disabled` |
| `packaging/plugins/mash-update-notice/notice.sh` | Mensaje de actualizaciones desactivadas |
| `packaging/helpers/mash-update-processes.c` | Porción Mach-O del host en lugar de arm64 fijo |
| `packaging/helpers/mash-tool-install` | Código de salida 4 |
| `packaging/wizard/main.rs` | El código 4 no es fatal; resumen y pruebas |
| `packaging/tests/test_assets.py`, `test_mash_update_notice.sh`, `packaging/helpers/tests/test-mash-tool-install.sh` | Pruebas nuevas y ajustadas |
| `INTEL-MAC.md`, `README.md` | Documentación |
