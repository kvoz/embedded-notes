# GCC-ARM в контейнере

Конфигурация образа выполняется через `Dockerfile`. Описание приводится для среды `CubeIDE-1.12.1` и тулчейна `gcc-arm-none-eabi-10.3-2021.10-x86_64-linux`.

## Вариант 1

В данном варианте сборка будет осуществляться через `headless`-режим работы среды `CubeIDE`. Нативный скрипт для такой сборки лежит внутри директории среды разработки.

Для данного варианта требуется генерация в `CubeMX` проекта `STM32CubeIDE`.

### Подготовка

Требуется скачать архив `STM32CubeIDE-Lnx` с официального [сайта](https://www.st.com/en/development-tools/stm32cubeide.html).

Чтобы не увеличивать количество слоёв в образе `Docker` скачанный архив предварительно надо обработать в локальной директории:

```bash
# Распакова из zip архива
$ unzip en.st-stm32cubeide_1.12.1_16088_20230420_1057_amd64.sh

# Запуск скрипта: распакова среды без установки в текущую директорию
$ bash ./st-stm32cubeide_1.12.1_16088_20230420_1057_amd64.sh --noexec --target .
```

Для дальнейшего создания образа `Docker` потребуется `st-stm32cubeide_1.12.1_16088_20230420_1057_amd64.tar.gz`, его необходимо поместить рядом с `Dockerfile` создаваемого образа. Остальные файлы архива можно удалить.

### Создание образа Docker

Содержание `Dockerfile`:

```dockerfile
# Базовый образ
FROM ubuntu:22.04

# Создание и переход в директорию, где будет установлена среда разработки
WORKDIR /opt/st/stm32cubeide_1.12.1

# Копирование из текущей локальной папки (где лежит Dockerfile) архива среды разработки
COPY st-stm32cubeide_1.12.1_16088_20230420_1057_amd64.tar.gz .

# Распаковка среды и удаление архива
RUN tar xfz st-stm32cubeide_1.12.1_16088_20230420_1057_amd64.tar.gz && \
    rm st-stm32cubeide_1.12.1_16088_20230420_1057_amd64.tar.gz
```

Запуск чистой сборки через `CubeIDE` в headless-режиме (выполняется из командной строки запущенного контейнера, см. запуск [ниже](#сборка-образов-и-запуск-контейнеров)):

```bash
$ /opt/st/stm32cubeide_1.12.1/headless-build.sh -data .. -import "." -cleanBuild proj_name
```

Параметры:

* `data` - указывает директорию для сохранения конфигурации workspace;
* `import` - указывает директорию, где хранятся файлы проекта (текущая).


Так как сборку по сути выполняет `Eclipse CDT`, то конфигурация сборки (настройка компилятора, линковщика и т.п.) сохраняются в `.cproject`. При использовании систем контроля версий имеет смысл его отслеживать.

По умолчанию, после генерации проекта в `CubeMX` для `CubeIDE` не создаются артефакты типа `.hex` и `.bin`, необходимо включить эту настройку в свойствах проекта через `CubeIDE` (Properties - C/C++ Build - Settings - Post build outputs).

## Вариант 2

В данном варианте сборка будет осуществляться через систему `make` и `ARM`-тулчейн.

Для данного варианта требуется генерация в `CubeMX` проекта `Makefile`.

### Подготовка

Не требуется, т.к. файлы тулчейна будут установлены отдельным слоем в `Docker` образе.

### Создание образа Docker

Содержание `Dockerfile`:

```dockerfile
# Базовый образ
FROM ubuntu:22.04

# Установка дополнительных инструментов для выполнения сборок
RUN apt-get update && \
    apt-get clean && \ 
    apt-get install -y \
    build-essential \
    wget \
    curl

# Переменная, определение версии тулчейна
ARG ARM_TOOLCHAIN_VERSION="10.3-2021.10"

# Переход в рабочий каталог /opt
WORKDIR /opt

# Скачивание и распаковка тулчейна с сайта arm.com
RUN wget -qO- https://developer.arm.com/-/media/Files/downloads/gnu-rm/${ARM_TOOLCHAIN_VERSION}/gcc-arm-none-eabi-${ARM_TOOLCHAIN_VERSION}-x86_64-linux.tar.bz2 | tar -xj

# Добавление в переменную среды PATH директории с бинарными файлами тулчейна
ENV PATH $PATH:/opt/gcc-arm-none-eabi-${ARM_TOOLCHAIN_VERSION}/bin
```

Запуск сборки выполняется из командной строки запущенного контейнера (см. запуск [ниже](#сборка-образов-и-запуск-контейнеров)) через систему `make`:

```bash
$ make clean
$ make all
```

## Сборка образов и запуск контейнеров

Сборка `Docker` образа:

```bash
# Сборка из текущей директории с тегом (на выбор пользователя)
$ docker build . -t my_repo/my_tag
# Проверка созданного образа
$ docker images
```

Для выполнения сборки в контейнере через коммандную строку без запуска сред разработки необходимо из директории, где находится целевой проект создать контейнер с монтированием директории проекта:

```bash
# Контейнер создаётся, в него монтируется директория proj_name из текущего каталога.
# Точка монтирования /home/dev/proj_name, она будет установлена в качестве рабочего каталога.
# Контейнер будет работать в интерактивном режиме, с поддержкой псевдо-TTY с заданным именем (на выбор пользователя).
$ docker container create -v ./proj_name:/home/dev/proj_name -w /home/dev/proj_name -i -t --name my_container my_repo/my_tag
# Проверка созданного контейнера
$ docker ps --all
# Запуск контейнера
$ docker container start --attach -i my_container
```

Для запуска сборки проекта через `VS Code` можно использовать плагин `Dev containers` со следующей конфигурацией в `.devcontainer/devcontainer.json`:

```json
// See https://aka.ms/vscode-remote/devcontainer.json for format details.
{
    "name": "Stm32CubeIDE SDK",
    "image": "my_repo/my_tag",
    // Add the IDs of any extensions you want installed in the array below.
    "customizations": {
        "vscode": {
            "extensions": [
                "spmeesseman.vscode-taskexplorer",
                "ms-vscode.makefile-tools"
            ],
            "settings": {
                "terminal.integrated.defaultProfile.linux": "bash",
                "terminal.integrated.profiles.linux": {
                    "bash": {
                        "path": "bash",
                        "icon": "terminal-bash"
                    }
                },
                "taskExplorer.pathToPrograms": {
                    "make": "make"
                }
            }
        }
    }
}
```

Имя образа контейнера должно совпадать с ранее созданным (поле `image`).

В конфигурации добавлены два плагина:

* `Task Explorer` - для отображения задач сборки,
* `Makefile Tools` - для поддержки редактирования `Makefile`.

Также выполняется корректировка настройки команды `make` (вместо `nmake`) и открытие терминала внутри контейнера.
