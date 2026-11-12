# SDF-Мир V15-C++: Техническое руководство (обновлённое)

**Версия**: V15  
**Стандарт**: C++20/23  
**Дата обновления**: 2026-07-26  
**Статус**: Этап 10+ (расширенный рендеринг + LOD, кросс-компиляторная поддержка)

---

## 1. Архитектура

### 1.1 Цели
- **SDF-специфичность**: вся геометрия через `float evaluate(float x, float y, float z)`
- **Детерминизм**: `double` для планетарных координат, `float` для локальных
- **Масштабируемость**: Sparse Virtual SDF, адаптивный LOD
- **Мультиплеер**: авторитативный сервер, QUIC + TCP fallback
- **Надёжность**: интеграционное тестирование с этапа 0, профилирование памяти с этапа 0, OTA-обновления
- **Кросс-платформенность**: поддержка MSVC, GCC и Clang с единой кодовой базой на C++20 модулях

### 1.2 Планетарная модель

| Параметр | Значение |
|----------|----------|
| Радиус планеты | 6371 км |
| Радиус ядра | 3480 км |
| Гравитация | 9.8 м/с² |

**Слои**:
- `0–3480 км` — Блокированная зона (ядро)
- `3480–6371 км` — Полный SDF с модификациями
- `6371–6383 км` — Строительство игроков
- `6383–6421 км` — Упрощённые коллизии
- `6421–6456 км` — Нет коллизий с местностью
- `6456+ км` — Кеплеровы орбиты

---

## 2. Технологический стек

### 2.1 Язык и компиляторы

| Компилятор | Минимальная версия | Платформы | Статус | Особенности |
|------------|-------------------|-----------|--------|-------------|
| **MSVC** | 19.44+ (VS 2022 17.14+) | Windows | ✅ Основной | Лучшая поддержка `.cppm`, MSBuild генератор |
| **GCC** | 13.2+ | Linux | ⬜ Целевой | Требует `-fmodules-ts`, `gcm.cache` |
| **Clang** | 18.0+ | Linux, macOS, Windows | ⬜ Целевой | libc++ modules, `clang-scan-deps` |

**Правила выбора компилятора**:
- **Разработка**: MSVC на Windows (быстрая итерация, лучшие ошибки модулей)
- **CI/CD**: GCC 13+ (Linux-сервер), Clang 18+ (macOS-клиент)
- **Релиз**: Все три компилятора проходят полный набор тестов

### 2.2 Сборка и пакеты

| Компонент | Версия/Тип | Примечание |
|-----------|------------|------------|
| Стандарт | C++20/23 | Модули (`.cppm`) вместо headers |
| Сборка | CMake 3.28+ | Политики CMP0155+ для модулей |
| Генератор | MSBuild (Win) / Ninja (Linux/macOS) | Ninja Multi-Config для кросс-платформы |
| Пакеты | vcpkg (основной) / Conan 2.0 (альтернатива) | `VCPKG_MANIFEST_MODE` |
| Стиль | Google C++ Style | `.clang-format`, `.clang-tidy` |

### 2.3 CMake Presets (кросс-компиляторные)

```json
// CMakePresets.json — фрагмент
{
  "version": 6,
  "configurePresets": [
    {
      "name": "windows-msvc-debug",
      "hidden": false,
      "generator": "Visual Studio 17 2022",
      "architecture": { "value": "x64", "strategy": "set" },
      "cacheVariables": {
        "CMAKE_CXX_STANDARD": "20",
        "CMAKE_TOOLCHAIN_FILE": "$env{VCPKG_ROOT}/scripts/buildsystems/vcpkg.cmake"
      }
    },
    {
      "name": "linux-gcc-release",
      "hidden": false,
      "generator": "Ninja Multi-Config",
      "cacheVariables": {
        "CMAKE_CXX_COMPILER": "g++-13",
        "CMAKE_CXX_STANDARD": "20",
        "CMAKE_CXX_FLAGS": "-fmodules-ts"
      }
    },
    {
      "name": "macos-clang-release",
      "hidden": false,
      "generator": "Ninja Multi-Config",
      "cacheVariables": {
        "CMAKE_CXX_COMPILER": "clang++",
        "CMAKE_CXX_STANDARD": "23",
        "CMAKE_CXX_FLAGS": "-stdlib=libc++"
      }
    }
  ]
}
```

### 2.4 Зависимости (vcpkg manifest)

```json
// vcpkg.json
{
  "name": "sdf-world-v15",
  "version": "15.0.0",
  "dependencies": [
    "glfw3",
    "glm",
    "imgui",
    { "name": "joltphysics", "version>=": "5.0.0" },
    { "name": "spdlog", "version>=": "1.13.0" },
    { "name": "fmt", "version>=": "10.2.0" },
    { "name": "fastnoise2", "version>=": "0.10.0" },
    { "name": "rocksdb", "version>=": "8.9.1" },
    { "name": "sqlite3", "version>=": "3.45.0" },
    { "name": "zstd", "version>=": "1.5.5" },
    { "name": "mbedtls", "version>=": "3.5.0" },
    { "name": "catch2", "version>=": "3.5.0" }
  ]
}
```

> **Примечание**: `glad` загружается вручную через `external/glad/` (генератор web). `imgui` подключается как docking-ветка.

---

## 3. Модульная структура (19 модулей)

### 3.1 Иерархия зависимостей

```
sdf-api (1)
    ↑
    ├── sdf-common (2)
    │       ↑
    │       ├── sdf-assets (3)
    │       │       ↑
    │       │       └── sdf-content (4)
    │       │
    │       ├── sdf-save (5)
    │       │       ↑
    │       └── sdf-physics (6)
    │               ↑
    │               └── sdf-engine (7)
    │                       ↑
    │                       ├── sdf-render-api (8)
    │                       │       ↑
    │                       ├── sdf-render-gl (9)
    │                       │
    │                       ├── sdf-client (10)
    │                       │       ↓
    │                       └── sdf-shader-permutations (11)
    │                               ↑
    │                               └── sdf-compute (12)
    │
    ├── sdf-server (13)
    │
    ├── sdf-launcher-core (14)
    │       ↑
    │       ├── sdf-launcher-ui (15)
    │       └── sdf-ota-updater (17)
    │
    ├── sdf-mod-manager (16)
    │
    ├── sdf-memory-profiler (18)
    │       ↑
    │       └── все модули (optional, с этапа 0)
    │
    └── sdf-integration-tests (19)
            ↑
            └── все модули (с этапа 0)
```

### 3.2 Фактическая структура папок (текущий статус 26.07.2026)

```
sdf-world-v15/
├── CMakeLists.txt
├── CMakePresets.json
├── vcpkg.json
├── vcpkg-configuration.json
├── .clang-format
├── .clang-tidy
├── .gitignore
├── README.md
│
├── external/                       # Ручное подключение
│   ├── glad/                       # OpenGL 3.3 Core loader
│   └── imgui/                      # Dear ImGui 1.91 (docking)
│
├── sdf-api/                        # 1 — Интерфейсы (⚠️ структура создана, код минимален)
│   ├── CMakeLists.txt
│   └── src/
│       ├── sdf_api.cppm
│       ├── coords/
│       ├── math/
│       ├── events/
│       ├── config/
│       ├── render/
│       ├── memory/
│       ├── log/
│       ├── threading/
│       └── telemetry/
│
├── sdf-common/                     # 2 — SDF-операции, JobSystem (⚠️ структура создана)
│   ├── CMakeLists.txt
│   └── src/
│       ├── math/
│       ├── noise/
│       ├── net/
│       ├── config/
│       ├── threading/
│       ├── memory/
│       ├── log/
│       └── telemetry/
│
├── sdf-assets/                     # 3 (⬜ пустой)
├── sdf-content/                    # 4 (⬜ пустой)
├── sdf-save/                       # 5 (⬜ пустой)
├── sdf-physics/                    # 6 (⬜ пустой)
│
├── sdf-engine/                     # 7 — ✅ АКТИВНЫЙ
│   ├── CMakeLists.txt
│   └── src/
│       ├── core/
│       │   ├── planetary_sdf.cppm      # ✅ PlanetConfig, BiomeType, evaluate()
│       │   ├── sdf_modification.cppm   # ✅ Модификации SDF
│       │   └── modification_stack.cppm # ✅ Стек изменений
│       └── sdf/
│           └── lod_system.cppm         # ✅ LOD-система (создана, интеграция в процессе)
│
├── sdf-render-api/                 # 8 (⬜ пустой)
├── sdf-render-gl/                  # 9 (⬜ пустой)
│
├── sdf-client/                     # 10 — ✅ АКТИВНЫЙ
│   ├── CMakeLists.txt
│   └── src/
│       ├── core/
│       │   ├── window_manager.cpp      # ✅ GLFW + ImGui окно
│       │   ├── window_manager.h        # ✅ Заголовок
│       │   └── client_config.cppm
│       └── render/
│           ├── marching_cubes.cppm     # 🔄 В процессе исправления (убрать цвет, добавить polygoniseCell)
│           └── planet_renderer.cppm    # ⬜ Связка SDF + MC + цвета
│
├── sdf-shader-permutations/        # 11 (⬜ пустой)
├── sdf-compute/                    # 12 (⬜ пустой)
├── sdf-server/                     # 13 (⬜ пустой)
├── sdf-launcher-core/              # 14 (⬜ пустой)
├── sdf-launcher-ui/                # 15 (⬜ пустой)
├── sdf-mod-manager/                # 16 (⬜ пустой)
├── sdf-ota-updater/                # 17 (⬜ пустой)
├── sdf-memory-profiler/            # 18 (⬜ пустой)
└── sdf-integration-tests/          # 19 (⬜ пустой)
```

---

## 4. Принципы разработки

### 4.1 Единственная ответственность
Каждый файл решает одну задачу:
- Полный файл исходного кода (`.cppm` или `.cpp`)
- Полный шейдер (`.glsl`)
- Полный файл конфигурации (`.json`, `.cmake`)

**Запрещено**: «вставь это в метод», «добавь поле в класс».

### 4.2 Работоспособность кода
Каждый файл должен:
- Компилироваться: `clang++ -std=c++20 -c file.cppm` (или `g++ -std=c++20 -fmodules-ts -c`, или `cl /std:c++20 /interface`)
- Содержать все импорты
- Иметь указанные зависимости
- Быть готовым к запуску

### 4.3 Кросс-компиляторная совместимость
- **MSVC**: `/std:c++20 /W4 /permissive- /Zc:preprocessor`
- **GCC**: `-std=c++20 -fmodules-ts -Wall -Wextra -Wpedantic`
- **Clang**: `-std=c++20 -stdlib=libc++ -Wall -Wextra -Wpedantic`
- Запрещены расширения vendor-specific без `#ifdef`
- `__cplusplus` должен быть ≥ 202002L

### 4.4 Порядок разработки V15 (3 трека)

| Этап | Клиент-трек | Сервер-трек | Инфра-трек | Результат |
|------|-------------|-------------|------------|-----------|
| 0 | — | — | CMake, vcpkg, TestFramework, Profiler stub | Сборка + тесты пустые проходят |
| 1 | sdf-api | sdf-api (shared) | — | Базовые интерфейсы |
| 2 | sdf-common | sdf-common (shared) | Profiler интеграция | SDF + JobSystem + профайлинг |
| 3 | sdf-assets | — | — | Ассеты |
| 4 | sdf-save | sdf-save (shared) | — | Сохранения |
| 5 | sdf-physics | sdf-physics (server-side) | — | Физика |
| 6 | sdf-engine | sdf-engine (minimal) | — | Планетарный SDF |
| 7 | sdf-content | — | — | Контент |
| 8 | sdf-render-api | — | — | Рендер абстракция |
| 9 | sdf-render-gl | — | — | OpenGL |
| 10 | sdf-client | sdf-server (basic) | — | Первый запуск + сервер одновременно ⭐ |
| 11 | sdf-shader-permutations | — | — | Шейдеры |
| 12 | sdf-launcher-core | — | — | Лаунчер |
| 13 | sdf-launcher-ui | — | — | UI лаунчера |
| 14 | sdf-mod-manager | — | — | Моды |
| 15 | sdf-ota-updater | sdf-ota-updater (server) | — | OTA клиент+сервер |
| 16 | sdf-compute | sdf-compute (server physics) | — | Compute |
| 17 | — | sdf-server (full) | — | Полный сервер |
| 18 | — | — | sdf-integration-tests | Интеграция |
| 19–25 | Механики, AI, орбита | Механики, AI, валидация | — | Фичи |
| 26 | Релиз | Релиз | OTA, rollback | Готово |

### 4.5 Тестирование перед продолжением
1. Скопировать файл по пути
2. Собрать: `cmake --preset=windows-msvc-debug && cmake --build build`
3. Тесты: `ctest -R module_name` (или `./build/test_module` для GCC/Clang)
4. Сообщить результат
5. Продолжить

---

## 5. Механики V15

| Механика | Модуль | Интерфейс |
|----------|--------|-----------|
| Отменяемые задачи | sdf-common/threading | `ICancelableJob` |
| Иерархический бюджет | sdf-common/memory | `IBudgetPartition` |
| Приоритетные события | sdf-api/events | `PriorityEventBus` |
| Миграции сохранений | sdf-save/migrations | `IMigration` |
| Fallback транспорт | sdf-common/net | `ITransport` |
| Atmosphere гистерезис | sdf-engine/view | `IAtmosphereProfile` |
| Lifetime management | sdf-launcher-core | `ISystemLifetime` |
| Dependency resolution | sdf-mod-manager | `IDependencyResolver` |
| Телеметрия | sdf-api/telemetry | `IPerformanceTelemetry` |
| OTA Delta Updates | sdf-ota-updater | `IOTAUpdater` |
| Memory Profiling | sdf-memory-profiler | `IMemoryProfiler` |
| Integration Testing | sdf-integration-tests | `IScenario` |

---

## 6. Детализация этапов V15 (актуальный статус на 26.07.2026)

### Этап 0 (Инфраструктура с тестами и профайлером)
- ✅ Структура 19 модулей (CMakeLists.txt для каждого)
- ✅ CMake 3.28+, vcpkg manifest
- ✅ `.clang-format`, `.clang-tidy`
- ✅ MSVC preset рабочий
- ⬜ GCC preset (Linux)
- ⬜ Clang preset (macOS)
- ⬜ TestFramework: Catch2 подключён, но тесты пустые
- ⬜ Profiler stub: интерфейс есть, реализация no-op

### Этап 1 (sdf-api — shared)
- ✅ `PlanetCoordinates` с `is_valid()`
- ✅ `EventBus` с `HandlerPriority`
- ✅ `ICancelableJob`, `BudgetPartition`
- ⬜ Полная документация интерфейсов

### Этап 2 (sdf-common — shared + profiler)
- ✅ SDF операции (union, subtract, intersect)
- ✅ SDF примитивы (сфера, куб, тор)
- ✅ Perlin шум (`glm::simplex` — временно, до FastNoise2)
- ✅ `JobSystemImpl` с cancellation
- ✅ `BudgetAllocatorImpl` с иерархией
- ✅ `ITransport`
- ⬜ Profiler интеграция: `AllocationTracker` подключён к аллокатору

### Этап 3 (sdf-assets)
- ⬜ `AssetManager`, `AsyncLoader` (заглушки)

### Этап 4 (sdf-save — shared)
- ⬜ SQLite, RocksDB (заглушки)
- ⬜ `IMigration`, `WorldMetadata`

### Этап 5 (sdf-physics — client + server)
- ⬜ Jolt, `SDFCollisionShape` (заглушки)
- ⬜ `FixedPointMath`
- ⬜ Серверная валидация физики

### Этап 6 (sdf-engine — client + minimal server) — ✅ АКТИВЕН
- ✅ `PlanetarySDF` с `evaluate()`, `getNormal()`, `getBiome()`, `getColor()`
- ✅ `ViewModeManager` с `AtmosphereProfile`
- ✅ Модификации SDF (копание)
- ✅ Биомы: Ocean, Beach, Plains, Forest, Mountains, Snow, Desert
- ✅ Руды: уголь, железо, золото
- ✅ `LODSystem` (модуль создан, интеграция в процессе)
- ⬜ Сервер: базовая генерация чанков

### Этап 7 (sdf-content)
- ✅ `BiomeRegistry` (встроен в sdf-engine)
- ⬜ JSON-загрузка биомов
- ⬜ Рецепты крафта (частично в sdf-engine)

### Этап 8 (sdf-render-api)
- ⬜ `IRenderDevice`, `IShaderCompiler` (заглушки)

### Этап 9 (sdf-render-gl)
- ⬜ `OpenGLRenderDevice` (заглушка)
- ✅ GLAD загружен в `external/`
- ✅ GLFW 3.4 работает через vcpkg

### Этап 10 (sdf-client + sdf-server basic) — 🔄 В ПРОЦЕССЕ
- ✅ `GLFWWindow`, камера (WASD + мышь)
- ✅ Dear ImGui интегрирован
- ✅ Шейдеры (Phong lighting)
- 🔄 `SurfaceRenderer` — marching cubes в процессе исправления
- ⬜ `TCPFallback`
- ⬜ Сервер: `QUICServer`, `TCPFallbackServer` базовые
- ⬜ Интеграция: клиент коннектится к серверу (ping/pong)

**Текущая задача этапа 10**:
1. Исправить `marching_cubes.cppm` (убрать цвет, добавить `polygoniseCell` по Paul Bourke)
2. Создать `planet_renderer.cppm` (цвета + геометрия)
3. Обновить `window_manager.cpp` для модульной структуры
4. Интегрировать LOD-систему

### Этап 11 (sdf-shader-permutations)
- ⬜ Препроцессор, компилятор (заглушки)

### Этап 12 (sdf-launcher-core)
- ⬜ `SystemInitializer` (заглушка)

### Этап 13 (sdf-launcher-ui)
- ⬜ `ServerBrowser` (заглушка)

### Этап 14 (sdf-mod-manager)
- ⬜ `ModInstaller`, `DependencyResolver` (заглушки)

### Этап 15 (sdf-ota-updater — client + server)
- ⬜ `OTAUpdater` (заглушка)
- ⬜ `DeltaPatcher` (bsdiff/zstd)
- ⬜ `SignatureVerifier` (ECDSA)
- ⬜ `ResumeManager`
- ⬜ `AtomicSwap`

### Этап 16 (sdf-compute + memory-profiler full)
- ⬜ Compute шейдеры
- ⬜ `IMemoryProfiler` полная реализация
- ⬜ `LeakDetector`, `OOMPredictor`, `FlameGraphBuilder`

### Этап 17 (sdf-server full)
- ⬜ Полный сервер: `BehaviorTree`, `NavigationSDF`, `AnticheatEngine`

### Этап 18 (sdf-integration-tests)
- ⬜ `HeadlessClient` (null rendering)
- ⬜ `HeadlessServer`
- ⬜ `ScenarioRunner`
- ⬜ `LoginScenario`, `DeterminismAssertions`

### Этап 19–25 (Механики, AI, Орбита)
- ⬜ Сеть полная
- ⬜ `DiggingController`, `BuildingController`
- ⬜ Impostors, LOD
- ⬜ `BehaviorTree`, `NavigationSDF`
- ⬜ `KeplerElements`, `OrbitCalculator`

### Этап 26 (Релиз)
- ⬜ Все модули
- ⬜ Integration tests проходят на MSVC + GCC + Clang
- ⬜ Нет утечек в 1-часовом тесте
- ⬜ OTA staging, rollback

---

## 7. Ключевые системы

### 7.1 OTA Updater

| Компонент | Технология |
|-----------|------------|
| Delta | bsdiff + zstd |
| Подпись | ECDSA P-256 |
| Докачка | HTTP Range |
| Атомарность | Rename + symlink (POSIX) / MoveFileEx (Windows) |
| Rollback protection | Version whitelist |

### 7.2 Memory Profiler

| Функция | Накладные расходы |
|---------|-------------------|
| Allocation tracking | +8 bytes, +5% CPU |
| Leak detection | 10ms pause |
| OOM prediction | 0 |
| Flame graph | +2% CPU |

### 7.3 Integration Tests

| Сценарий | Таймаут |
|----------|---------|
| login | 30s |
| digging | 60s |
| flight_to_orbit | 120s |
| multiplayer_sync | 60s |
| physics_stress | 30s |
| memory_pressure | 60s |
| network_partition | 60s |
| mod_hot_reload | 30s |

---

## 8. Требования к коду

### 8.1 C++20/23

| Возможность | Применение |
|-------------|------------|
| Modules | Интерфейсы (`.cppm`) |
| Concepts | `SDFEvaluator`, `OTAUpdater`, `MemoryProfiler`, `Scenario` |
| Coroutines | Async I/O, OTA download |
| `std::source_location` | Allocation tracking |
| `std::stacktrace` | Profiler, crash reporter |
| `std::expected` | Ошибки |

### 8.2 Запреты

| Запрещено | Причина | Альтернатива |
|-----------|---------|--------------|
| `std::function` hot path | Type erasure | Шаблоны |
| `virtual` hot path | Vtable | CRTP |
| `new/delete` raw | Утечки | `std::unique_ptr` |
| exceptions control flow | Стоимость | `std::expected` |
| `std::map` частый доступ | Дерево | `std::unordered_map` |
| `iostream` | Медленно | `fmt::format` |
| RTTI | Overhead | `static_assert` |
| Compiler-specific extensions | Непереносимость | Стандарт C++20 |

### 8.3 Компилятор-специфичные замечания

**MSVC**:
- Использовать `/permissive-` для строгого соответствия стандарту
- `/Zc:preprocessor` для conforming preprocessor (важно для модулей)
- `/W4 /WX` — warnings as errors

**GCC**:
- `-fmodules-ts` обязателен для модулей
- `gcm.cache` должен быть в `.gitignore`
- `-fmodule-mapper` для сложных проектов

**Clang**:
- `-stdlib=libc++` на macOS и при желании на Linux
- `clang-scan-deps` для сканирования зависимостей модулей
- `-fmodules` и `-fbuiltin-module-map` для стандартных модулей (C++23)

---

## 9. Тестирование

| Тип | Инструмент |
|-----|------------|
| GPU | RenderDoc |
| CPU | Tracy |
| Память | sdf-memory-profiler |
| Статический анализ | clang-tidy (все три компилятора) |
| Модульные | Catch2 |
| Интеграционные | sdf-integration-tests |
| CI | GitHub Actions (GCC, Clang), Azure Pipelines (MSVC) |

---

## 10. Платформы

### 10.1 Минимальные требования

| Компонент | Спецификация |
|-----------|--------------|
| CPU | 4 ядра, AVX2 |
| RAM | 8 GB |
| GPU | GTX 750 Ti / RX 580 |
| API | OpenGL 4.3 |
| Диск | 20 GB + 5 GB OTA |

### 10.2 Рекомендуемые требования

| Компонент | Спецификация |
|-----------|--------------|
| CPU | 6 ядер, AVX-512 |
| RAM | 16 GB |
| GPU | RTX 3060 / RX 6700 XT |
| API | OpenGL 4.6 |

### 10.3 Целевые платформы и компиляторы

| Платформа | Компилятор | Статус | CI |
|-----------|------------|--------|-----|
| Windows 10/11 | MSVC 19.44+ | ✅ Активная разработка | Azure Pipelines |
| Windows 10/11 | Clang-CL 18+ | ⬜ Целевой | GitHub Actions |
| Ubuntu 24.04 | GCC 13.2+ | ⬜ Целевой | GitHub Actions |
| Ubuntu 24.04 | Clang 18+ | ⬜ Целевой | GitHub Actions |
| macOS 14+ | Clang 18+ (Xcode 15+) | ⬜ Целевой | GitHub Actions |

---

## 11. CI/CD и кросс-компиляция

### 11.1 GitHub Actions (пример workflow)

```yaml
# .github/workflows/ci.yml
name: Cross-Compiler CI
on: [push, pull_request]

jobs:
  build-msvc:
    runs-on: windows-latest
    steps:
      - uses: actions/checkout@v4
      - uses: lukka/run-vcpkg@v11
      - run: cmake --preset=windows-msvc-release
      - run: cmake --build build --config Release
      - run: ctest --test-dir build -C Release

  build-gcc:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: sudo apt install g++-13 ninja-build
      - uses: lukka/run-vcpkg@v11
      - run: cmake --preset=linux-gcc-release
      - run: cmake --build build --config Release
      - run: ctest --test-dir build -C Release

  build-clang:
    runs-on: macos-latest
    steps:
      - uses: actions/checkout@v4
      - run: brew install llvm@18 ninja
      - uses: lukka/run-vcpkg@v11
      - run: cmake --preset=macos-clang-release
      - run: cmake --build build --config Release
      - run: ctest --test-dir build -C Release
```

### 11.2 Скрипты сборки

```powershell
# build.ps1 (Windows — универсальный)
param([string]$Preset = "windows-msvc-debug")
cmake --preset=$Preset
cmake --build build --config Debug
```

```bash
#!/bin/bash
# build.sh (Linux/macOS — универсальный)
PRESET=${1:-linux-gcc-release}
cmake --preset=$PRESET
cmake --build build --config Release
```

---

## 12. Сроки V15 (обновлённые)

| Команда | Срок | Комментарий |
|---------|------|-------------|
| 1 опытный (MSVC) | 16–22 месяцев | Текущий режим |
| 3–4 человека (разные компиляторы) | 14–18 месяцев | С учётом CI и кросс-платформы |
| MVP (только MSVC) | 12–14 месяцев | Без GCC/Clang в релизе |
| Полный релиз (3 компилятора) | 18–24 месяца | Все CI зелёные |

---

## 13. Заключение

Данное руководство является **единственным актуальным документом** для SDF-Мир V15. Оно сохраняет полную структуру 19 модулей и 27 этапов разработки, но обновлено под текущие реалии:

- **Технологический стек**: vcpkg вместо Conan, MSVC как основной компилятор, GCC и Clang как целевые
- **Текущий статус**: Этап 10+ с работающим GLFW-окном, ImGui, камерой, Phong-шейдерами, планетарным SDF, биомами и LOD-системой
- **Ближайшие задачи**: исправить `marching_cubes.cppm`, создать `planet_renderer.cppm`, интегрировать LOD

**Следующий шаг**: согласовать приоритеты и приступить к завершению **Этапа 10** (полноценный рендер поверхности планеты) с последующим расширением на GCC/Clang CI.

Готов к генерации конкретных файлов по любому из этапов. Укажите: *"Этап 10: файлы marching_cubes.cppm, planet_renderer.cppm"* и т.д.
