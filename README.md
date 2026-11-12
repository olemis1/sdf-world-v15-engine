# sdf-world-v15-engine
<img width="1280" height="719" alt="photo_2026-11-10_02-10-13" src="https://github.com/user-attachments/assets/4fd1058a-0e70-44af-a48d-ef4c5c9d949c" />
<img width="1280" height="720" alt="photo_2026-11-10_02-10-07" src="https://github.com/user-attachments/assets/4d2b77e9-51aa-4ce1-b442-9084c172506d" />
<img width="1280" height="1046" alt="photo_2026-11-10_02-10-19 — копия — копия (2)" src="https://github.com/user-attachments/assets/74f85682-3f23-49d4-9072-bb542b0641f7" />


# SDF-Мир V15-C++ (Planetary Simulation & 3D Engine)

[![C++20/23](https://shields.io)](https://cppreference.com)
[![CMake](https://shields.io)](https://cmake.org)
[![OpenGL](https://shields.io)](https://opengl.org)
[![Status](https://shields.io)]()

Отечественный кроссплатформенный 3D-движок реального времени для процедурной генерации, детерминированной симуляции и интерактивной визуализации виртуальных сред планетарного масштаба на базе разреженных знаковых функций расстояния (**Sparse Virtual SDF**).

---

## 📸 Демонстрация текущего этапа (Этап 10+)

> *Примечание: сюда можно вставить прямые ссылки на ваши скриншоты с рендером планеты, сеткой Marching Cubes и интерфейсом Dear ImGui.*

* **Планетарная модель:** радиус 6371 км, ядро 3480 км, гравитация 9.8 м/с², деление на 6 высотных и орбитальных слоев.
* **Точность расчетов:** гибридные координаты (`double` для планетарного пространства, `float` для локального рендеринга и физики).
* **Генерация поверхности:** триангуляция изоповерхностей методом Marching Cubes, адаптивный LOD, 7 типов биомов и процедура распределения руд.

---

## 🛠 Системные требования и зависимости

### Аппаратные требования (минимум)
* **CPU:** 4 ядра с поддержкой AVX2
* **RAM:** 8 GB
* **GPU:** NVIDIA GTX 750 Ti / AMD RX 580 с поддержкой **OpenGL 4.3+**
* **Диск:** 20 GB свободного места

### Инструменты сборки и библиотеки
* **Компиляторы (C++20/23 Modules):**
  * **Windows:** MSVC 19.44+ (Visual Studio 2022 17.14+)
  * **Linux:** GCC 13.2+ или Clang 18.0+
  * **macOS:** Clang 18.0+ (Xcode 15+)
* **Система сборки:** CMake 3.28+ (с поддержкой `CMP0155` для `.cppm` модулей) и Ninja / MSBuild
* **Пакетный менеджер:** [vcpkg](https://github.com) (режим манифеста `vcpkg.json`)
* **Основные зависимости:** `glfw3`, `glm`, `imgui` (docking), `joltphysics`, `fastnoise2`, `spdlog`, `fmt`, `rocksdb`, `sqlite3`, `zstd`, `mbedtls`, `catch2`.

---

## 🚀 Инструкция по сборке

### 1. Предварительная подготовка
Убедитесь, что у вас установлен `vcpkg` и задана переменная окружения `VCPKG_ROOT`:

```bash
git clone https://github.com.git
./vcpkg/bootstrap-vcpkg.bat   # Для Windows
# ./vcpkg/bootstrap-vcpkg.sh  # Для Linux / macOS
```

Укажите путь к папке `vcpkg` в переменной среды `VCPKG_ROOT`.

### 2. Клонирование репозитория

```bash
git clone https://github.com
cd sdf-world-v15-engine
```

### 3. Сборка под Windows (MSVC — основной режим)

Используйте встроенные пресеты `CMakePresets.json`:

```powershell
# Конфигурация проекта (подтянет зависимости через vcpkg автоматически)
cmake --preset=windows-msvc-debug

# Сборка проекта
cmake --build build --config Debug

# Запуск модульных тестов
ctest --test-dir build -C Debug --output-on-failure
```

Или через универсальный скрипт:
```powershell
./build.ps1 -Preset windows-msvc-debug
```

### 4. Сборка под Linux (GCC 13+)

```bash
sudo apt install g++-13 ninja-build cmake
cmake --preset=linux-gcc-release
cmake --build build --config Release
ctest --test-dir build -C Release
```

### 5. Сборка под macOS (Clang 18+)

```bash
brew install llvm@18 ninja cmake
cmake --preset=macos-clang-release
cmake --build build --config Release
ctest --test-dir build -C Release
```

---

## 🏗 Архитектура проекта (19 модулей C++20)

Проект построен на изолированных модулях (`.cppm`) без использования классических заголовочных файлов в ядре:

1. `sdf-api` — базовые интерфейсы, планетарные координаты, шина событий.
2. `sdf-common` — математика SDF, система задач (`JobSystem`), менеджер памяти.
3. `sdf-assets` / `sdf-content` — асинхронная загрузка ресурсов и реестр биомов.
4. `sdf-save` — сериализация мира (SQLite / RocksDB) и миграции.
5. `sdf-physics` — детерминированная физика на базе Jolt Physics.
6. `sdf-engine` — планетарный SDF, стек модификаций рельефа, система LOD.
7. `sdf-render-api` / `sdf-render-gl` — абстракция рендера и реализация OpenGL.
8. `sdf-client` — оконный клиент (GLFW + ImGui), камера, триангуляция Marching Cubes.
9. `sdf-shader-permutations` / `sdf-compute` — управление шейдерами и GPU-вычисления.
10. `sdf-server` — авторитативный сервер (QUIC + TCP fallback).
11. `sdf-launcher-core` / `sdf-launcher-ui` / `sdf-ota-updater` — лаунчер и атомарные дельта-обновления (`bsdiff` + `zstd` + `ECDSA`).
12. `sdf-mod-manager` — менеджер модификаций и разрешение зависимостей.
13. `sdf-memory-profiler` / `sdf-integration-tests` — встроенный профайлер памяти и автотесты.

---

## 📄 Документация

* Подробное техническое руководство и дорожная карта разработки находятся в папке `docs/`.
