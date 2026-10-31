# Plan de acción: renderer v2 / v3 en NVIDIA

Fork: `RomanFama592/iss_pc` · base: `developer` @ `37c05ba` (27/10/2026, igual que `upstream/developer`)
Máquina de prueba: RTX 4060 Ti 8 GB (driver 591.59, subgroup de 32 hilos), Ryzen 9 5900X, 64 GB, Windows.
Objetivo, en orden: **sin crasheos → imagen correcta → FPS**.

---

## 1. Diagnóstico (con evidencia)

### 1.1 El crash con native passes NO es de NVIDIA ni de native passes: ya está corregido
- Las 2 partidas que se cortaron (29/10) fallan en `tlg-probe-v2.exe+0x177b4e` = `ImGui::RenderTextClipped`, leyendo `0x101`.
- El fallo se atribuye a `ImGui::BeginCombo`: con `rsp` en el crash, los argumentos `pos_min`/`pos_max` estaban en `+0x50`/`+0x68` de la pila del llamador. De los 10 llamadores de `RenderTextClipped` en el binario v1.0, solo `BeginCombo` usa esos offsets.
- Causa en v1.0 (`cde38af`, `port/gpu/shim/tlg_overlay.cpp`): `fps_labels` tenía **4** textos y `TlgGraphics::FpsLimits` **5** valores (`{30, 60, 120, 240, 0}`). Con el límite "sin límite" (índice 4), `Combo` lee `fps_labels[4]` (basura) y crashea al dibujar el menú Insert. Además, los textos estaban corridos (60 se veía como "120").
- **Ya corregido** en `50607e5` (22/10, "Fix Renderer v2 overlay cursor and FPS selector"), con `static_assert`. Lo incluyen `beta-2026.10.23-*`, `beta-2026.10.24-*` y `beta-rc1`.
- El crash viejo de NVIDIA con native passes (`nvoglv64.dll+0xede28f`) se corrigió en v1.0 (PR #490). En los logs del usuario no aparece ningún otro crash de native passes.

### 1.2 Piso blanco / vegetación negra en v2: causa probable, `S_WQM_B64` sin implementar
- A/B en la misma partida y el mismo lugar: renderer clásico = correcto; v2 (con o sin native passes) = asfalto mojado blanco quemado y hojas y pasto negros.
- **MSAA descartado:** los dos renderers mantienen el MSAA del juego en NVIDIA (`tlg_msaa_default.h` lo apaga solo en AMD/Intel Windows).
- **Diferencia clave:** en el clásico, `S_WQM_B64` está implementado (`port/gpu/shadps4/shader_recompiler/frontend/translate/scalar_alu.cpp`, portado de SotC `debb0c49`/`e3227814`). Su comentario describe exactamente estos síntomas:
  - *"SotC's lake/river water does s_wqm_b64 vcc, exec … leaving VCC unset made that branch take garbage and skip the water normal map"*, que coincide con la superficie mojada mal sombreada.
  - *"alpha-tested foliage grows its test mask to quads so the neighbours used for the texture LOD stay alive"*, que coincide con las hojas negras.
- En **v2** (`port/gpu_v2/shadps4/.../scalar_alu.cpp:103`) y en **v3** (`port/gpu_v3/...`) `S_WQM_B64` es un `break` (no-op).
- Diferencias secundarias del clásico que faltan en v2: `Shader::Toggles::LaneDivergence` y `IsLaneVaryingSource` en `lower_wave64_pass.cpp` (492 líneas en el clásico, 199 en v2), barreras LDS para grupos de menos de 64 hilos y de varios waves, y `ReadLane` uniforme (`recompiler_toggles.h`).
- **v2 ya tiene dos correcciones para hardware de 32 hilos, pero no hay forma de encenderlas.** `Profile::lds_barriers_multi_wave` y `Profile::wave_memory_barriers` (`shared_memory_barrier_pass.cpp`) salen de `Common::Perf::GetTuning()`, cuyos campos valen `false` y **nada los cambia** en v2:
  - `perf_sampler.cpp`, que leería `perf_tuning.ini`, no se compila en v2.
  - `ISS_TUNING` (`gpu/bridge.cpp:486`) no reconoce esas claves.
  - El log del usuario lo confirma: `Compute shader LDS barriers for multi wave workgroups: false, buffer memory barriers within split GCN waves: false`.
  - Según su propio comentario, `wave_memory_barriers` cubre los *"GPU particle sorting passes within a wave"* (humo y partículas).

### 1.3 v3 (axylpup) en NVIDIA: riesgo alto
- v3 pide `requiredSubgroupSize = 64` solo si `IsSubgroupSize64Supported()` (AMD). En NVIDIA (máximo 32) los shaders de compute corren con 32 hilos.
- v3 **no tiene** `lower_wave64_pass` ni `S_WQM_B64`. Los FPS publicados (76–105) son de una RX 9060 XT.
- Esperable en NVIDIA: los mismos errores que en v2 o peores (humo, partículas e iluminación por tiles en compute).
- El autor mide en una RTX 2060 y nunca ejecutó v3 (etapas 3b y 4 pendientes, `docs/RENDERER_V3_AXYLPUP.md` en la rama `feat/renderer-v3-axylpup`). **Datos en NVIDIA y el lowering de wave64 son el aporte más valioso.**

---

## 2. Entorno de trabajo

### 2.1 Git en tu PC
- El repo está en `D:\Juegos\Others\PortsNativosPS4\Infamous Second Son\iss_pc` (rama `developer`, remotes `origin` = tu fork, `upstream` = HayatoG).
- Usá **Git for Windows** directamente: `git fetch upstream`, ramas, commits y push desde tu sesión. La terminal de Claude en tu PC no puede borrar archivos ni salir a GitHub, así que yo preparo los cambios y vos hacés commit y push, o me vinculás GitHub para que empuje desde la nube.
- Antes de compilar: `git submodule update --init --recursive`.

### 2.2 Compilar (dos opciones)
**A. GitHub Actions en tu fork (sin instalar nada):**
1. En tu fork: Settings → Actions → habilitar workflows.
2. Actions → **Beta** → *Run workflow* → `branch` = tu rama (ej. `fix/v2-wqm`).
3. Descargar el artifact `infamous-windows-x86_64` (zip, ~120 MB). Tarda ~30–60 min.

**B. Local con MSYS2 CLANG64 (iteración rápida):**
1. Instalar MSYS2, abrir la terminal **CLANG64** y ejecutar:
   ```
   pacman -S --needed git patch mingw-w64-clang-x86_64-{toolchain,cmake,ninja,pkgconf,python,vulkan-headers,vulkan-loader,vulkan-memory-allocator,glslang,sdl3,fmt,boost,robin-map,xxhash,ffmpeg,zydis,spirv-tools,spirv-headers,spirv-cross,freetype}
   ```
   Es la lista de `.github/workflows/build.yml`, job `windows`.
2. Desde `port/`: `TLG_RENDERER_V2=1 bash build.sh`. Genera `out/tlg-probe-v2.exe`.
3. Para probar solo el renderer, reemplazar `out\tlg-probe-v2.exe` en una **copia** de la instalación del juego. Nunca en la instalación con la que jugás.

### 2.3 Instalaciones separadas
- `Infamous Second Son\` = la que usás para jugar (renderer clásico, configuración estable).
- `ISS-dev\` = la beta o tu build de prueba. Copiá `user\savedata` para tener la misma partida, con el mismo `game_dir`.
- Punto de prueba fijo: **el estacionamiento de las capturas** (día y noche), con overlay F10 y la misma cámara.

---

## 3. Fases

Cada fase cierra con medición y capturas. La siguiente empieza solo si la anterior lo justifica.

### Fase 0: línea base con beta-rc1 (sin código, 1 sesión)
- Bajar `beta-rc1` (pre-release 28/10) a `ISS-dev\`. Trae el fix del crash, ultrawide nativo y DLSS como opción.
- v2 + native passes, overlay visible, menú Insert abierto y cerrado varias veces, ~30 min de juego.
- Registrar FPS en el estacionamiento y en la ciudad, VRAM, si crashea, y capturas del piso y la vegetación.
- **Criterio:** 0 crasheos. Si hay alguno, ir a la Fase 3 con el log.

### Fase 1: corregir la imagen en v2 (ramas `fix/v2-tuning-wave-barriers` y `fix/v2-wqm`)
0. **Primer PR (2 líneas):** en `port/gpu_v2/gpu/bridge.cpp`, dentro del parser de `ISS_TUNING`, agregar
   `else if(name=="lds_barriers_multi_wave")set(tuning.lds_barriers_multi_wave,value);` y
   `else if(name=="wave_memory_barriers")set(tuning.wave_memory_barriers,value);`.
   Ya participan de la clave del caché (`SetComputeShaderVariant`), así que se recompilan solas.
   Probar agregando `,lds_barriers_multi_wave=1,wave_memory_barriers=1` a `ISS_TUNING` en `gpu_v2/defaults.env` de la instalación de prueba.
   El log debe decir `... multi wave workgroups: true ... split GCN waves: true`. Comparar piso, follaje, humo y partículas, además de los FPS.
1. Portar `S_WQM_B64` del clásico a `port/gpu_v2/shadps4/shader_recompiler/frontend/translate/scalar_alu.cpp`. Las operaciones de IR (`InverseBallot`, `QuadBroadcast`, `Ballot`) ya existen en v2.
2. Agregar el interruptor `ISS_WQM` (por defecto 1, igual que el clásico) e **incluirlo en la clave del caché de shaders** de v2. Si no se incluye, se reusan los SPIR-V viejos. `SetComputeShaderVariant` en `vk_pipeline_cache.cpp` solo cubre compute; WQM afecta a los **pixel shaders**, así que hay que buscar la clave de los gráficos o borrar el caché en cada A/B.
3. Build (opción A o B), borrar el caché de shaders de v2 y dos sesiones en el punto de prueba con `ISS_WQM=1` y `ISS_WQM=0`.
4. **Criterio:** piso y follaje iguales al clásico con `ISS_WQM=1`, sin pérdida de FPS medible.
5. Si no alcanza: portar `LaneDivergence`/`IsLaneVaryingSource` a `lower_wave64_pass.cpp` de v2 y después las barreras LDS (`TLG_LDS_SMALL`/`LDS_MULTI`/`LDS_READLANE`). Una por commit, cada una con su interruptor A/B.
6. PR a `upstream/developer` con capturas antes y después, logs y el hash de origen (SotC `debb0c49`, `e3227814`) en el commit.

### Fase 2: robustez del overlay y diagnóstico de crashes (rama `fix/fault-unwind`)
1. `port/src/probe.c` (`fatal_exception`): la pila se recorre por la cadena de `rbp`, y el código con `-fomit-frame-pointer` (ImGui, el driver) da 0 frames. Agregar un recorrido con `RtlLookupFunctionEntry` + `RtlVirtualUnwind` (el binario MinGW/clang tiene `.pdata`) y además imprimir el símbolo más cercano si hay. Así el próximo crash dice quién llamó, sin análisis a mano.
2. `static_assert` de tamaño en todos los `ChoiceRow`/`Combo` con arreglos estáticos (v2 y `port/gpu_v3/shim/tlg_overlay.cpp:446`, que hoy está bien pero sin el assert).
3. Revisar los otros desplegables con arreglos estáticos (por ejemplo `extras`, `corners`, `layouts`, `DisplayModeNames`) para que la cantidad de textos coincida con `count`.

### Fase 3: estabilidad de native passes en NVIDIA (solo si la Fase 0 muestra crasheos)
- Con el unwind de la Fase 2, clasificar cada crash: driver (`nvoglv64.dll`), recorder de native passes o shim.
- Interruptores para aislar, ya existentes en `gpu_v2/defaults.env`: `ISS_NATIVE_PASSES_ASYNC_SCAN`, `ISS_NATIVE_PASSES_SUBMIT`, `ISS_NATIVE_PASSES_DYNAMIC`, `ISS_PARALLEL_RENDER`, `ISS_NATIVE_GBUFFER_SETS`, `ISS_NATIVE_MATERIALS`. Uno por sesión.
- Opcional: `VK_NV_device_diagnostic_checkpoints` (el driver lo expone) para device lost.

### Fase 4: v3 en NVIDIA (en coordinación con el autor)
- La rama es experimental: *"nada va a developer sin PR y sin autorización"*. **Primero abrir una issue o discusión** ofreciendo pruebas en NVIDIA y el port de wave64, y acordar alcance.
- 4a. Ayudar en la etapa 3b del plan del autor (stubs → shim real, base de memoria del guest `0x200000000`).
- 4b. Portar `lower_wave64_pass` + `S_WQM_B64` (+ barreras LDS para varios waves y `wave_memory_barriers`) a `port/gpu_v3`, activos cuando `!IsSubgroupSize64Supported()`. Lo aprendido en la Fase 1 dice cuáles importan.
- 4c. Fase 0 del autor en la 4060 Ti: v2 frente a v3, misma ruta, VSync off, FPS medio, 1% low, CPU/GPU por cuadro.

---

## 4. Reglas del proyecto upstream (para que los PR entren)
- GPL-2.0-or-later; mantener SPDX y créditos de origen (SotC, Castlesea, axylpup) y el hash de origen en el commit.
- Nada de archivos del juego ni de NVIDIA en el repo.
- Un cambio por commit, con interruptor A/B cuando altera shaders. Usar la plantilla de PR (`.github/pull_request_template.md`).
- PR contra `developer` (no `main`).

## 5. Riesgos
- `ISS_WQM` puede afectar otros efectos que hoy "andan por casualidad": comparar varias zonas, no solo el estacionamiento.
- Un build local puede no ser idéntico al de CI (LTO): para medir FPS, usar el artifact de Actions.
- v3 puede cambiar mucho en pocos días: rebasar seguido sobre `upstream/feat/renderer-v3-axylpup`.
