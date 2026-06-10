# Отчет по лабораторной работе lab09

## Выполнение работы

### Инициализация проекта и настройка метрик покрытия в CMake
Был создан новый публичный репозиторий с названием lab09. В терминале виртуальной машины рабочее пространство было сформировано путем копирования структуры и файлов из lab08. После этого был полностью очищен старый кэш сборки и изменен адрес удаленного репозитория на актуальный.

Для обеспечения возможности сбора метрик покрытия кода тестами (Code Coverage) в файл CMakeLists.txt была добавлена новая управляющая опция ENABLE_COVERAGE. При её активации компилятору GCC/Clang передаются специальные флаги --coverage, а также отключается оптимизация кода (-O0) для сохранения точной структуры строк при генерации отчетов:

* Настройка локального репозитория:
cd ~/x0mwsdc/workspace/projects
cp -r lab08 lab09
cd lab09
rm -rf build
git remote remove origin
git remote add origin https://github.com/x0mwsdc/lab09.git

* Модификация CMakeLists.txt (добавление секции покрытия):
option(ENABLE_COVERAGE "Enable code coverage metrics" OFF)
if(ENABLE_COVERAGE)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} --coverage -O0 -g")
endif()

### Настройка автоматического расчета покрытия в CI (GitHub Actions)
Сценарий автоматизации конвейера непрерывной интеграции (.github/workflows/cmake.yml) был обновлен. В виртуальное окружение была добавлена установка утилиты lcov. Шаг конфигурации CMake запускается с активным флагом -DENABLE_COVERAGE=ON.

После выполнения тестов выполняется сбор данных о покрытии. Для обеспечения стабильной сборки на сервере были добавлены флаги игнорирования специфичных нестыковок строк (--ignore-errors mismatch,inconsistent), возникающих из-за развертывания макросов фреймворка GoogleTest. Также из финального отчета исключаются системные библиотеки и сторонний код, после чего утилита genhtml формирует наглядный HTML-отчет:

name: CMake Build, Test, Package, Docs, Lint and Coverage

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
    - uses: actions/checkout@v4
      with:
        submodules: true

    - name: Install Dependencies
      run: |
        sudo apt-get update
        sudo apt-get install -y doxygen clang-format cppcheck lcov

    - name: Lint Code with Clang-Format
      run: clang-format --dry-run --Werror include/*.hpp sources/*.cpp examples/*.cpp tests/*.cpp

    - name: Static Analysis with Cppcheck
      run: cppcheck --enable=all --error-exitcode=1 --suppress=missingIncludeSystem --suppress=unusedFunction include/ sources/

    - name: Configure CMake
      run: cmake -B build -S . -DBUILD_TESTS=ON -DENABLE_COVERAGE=ON

    - name: Build project
      run: cmake --build build

    - name: Run Tests
      run: cd build && ctest --output-on-failure

    - name: Generate Coverage Report
      run: |
        lcov --directory . --capture --output-file coverage.info --ignore-errors mismatch,inconsistent
        lcov --remove coverage.info '/usr/*' '*third-party*' '*tests*' --output-file filtered_coverage.info --ignore-errors unused,mismatch
        genhtml filtered_coverage.info --output-directory coverage_report

    - name: Create Packages via CPack
      run: cd build && cpack

    - name: Generate Documentation via Doxygen
      run: doxygen Doxyfile

    - name: Upload Build Artifacts
      uses: actions/upload-artifact@v4
      with:
        name: project-artifacts-and-coverage
        path: |
          build/*.tar.gz
          build/*.deb
          docs/html/
          coverage_report/

### Публикация результатов и проверка работы
Все внесенные изменения были проверены на соответствие правилам форматирования линтера, зафиксированы в системе контроля версий Git и отправлены в удаленный репозиторий lab09:

clang-format -i include/*.hpp sources/*.cpp
git add .
git commit -m"chore: integrate lcov code coverage metrics into ci"
git push -u origin main

GitHub Actions выполнил весь конвейер проверок. Шаги статического анализа, форматирования, тестирования и сборки дистрибутивов CPack прошли без ошибок. Утилита lcov успешно обработала файлы профилирования .gcda
