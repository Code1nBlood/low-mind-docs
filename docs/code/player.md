# ProtoController v2.1

| Раздел | Описание |
| --- | --- |
| **Назначение** | Контроллер от первого лица с инерцией и тряской головы для хорроров |
| **Версия** | 2.1 |
| **База** | Brackeys ProtoController v1.0 |
| **Нода** | `CharacterBody3D` |

## Структура сцены
| Нода | Тип | Путь |
| --- | --- | --- |
| CharacterBody3D | Root | (скрипт здесь) |
| Head | Node3D | `$Head` |
| Camera3D | Camera3D | `$Head/Camera3D` |
| InteractRay | RayCast3D | `$Head/Camera3D/InteractRay` |
| Collider | CollisionShape3D | `$Collider` |

## Основные настройки
| Параметр | Тип | Дефолт | Описание |
| --- | --- | --- | --- |
| `can_move` | bool | `true` | Можно ли двигаться |
| `has_gravity` | bool | `true` | Включена ли гравитация |
| `can_jump` | bool | `true` | Можно ли прыгать |
| `can_sprint` | bool | `false` | Можно ли бегать |
| `can_freefly` | bool | `false` | Режим noclip |

> 💡 **Логика флагов**
> - Все параметры `can_*` работают как переключатели. При `false` соответствующий блок в `_physics_process` или `_unhandled_input` полностью пропускается, что экономит процессорное время.
> - `check_input_mappings()` при старте автоматически отключает функции, если в `Project Settings > Input Map` не созданы нужные действия, предотвращая краши.

## Speeds
| Параметр | Тип | Дефолт | Описание |
| --- | --- | --- | --- |
| `look_speed` | float | `0.002` | Чувствительность мыши |
| `base_speed` | float | `7.0` | Скорость ходьбы |
| `sprint_speed` | float | `10.0` | Скорость бега |
| `jump_velocity` | float | `6.0` | Сила прыжка |
| `freefly_speed` | float | `25.0` | Скорость в noclip |

## Movement Physics
| Параметр | Тип | Дефолт | Описание |
| --- | --- | --- | --- |
| `acceleration` | float | `12.0` | Скорость разгона |
| `deceleration` | float | `14.0` | Скорость торможения |
| `air_control` | float | `0.3` | Множитель ускорения в воздухе (0–1) |

> 💡 **Логика физики и инерции**
> - **`move_toward(current, target, step)`**: Плавно меняет `velocity.x/z` на `target_velocity` с шагом `accel/decel * delta`. Заменяет резкое присваивание `=`, создавая ощущение массы.
> - **`air_control`**: При прыжке `current_accel *= air_control`. Игрок почти не меняет горизонтальную траекторию в воздухе, что повышает реализм.
> - **Ортогональная обработка**: Горизонталь (`x/z`) и вертикаль (`y`) считаются отдельно. Гравитация не сбивает инерцию разгона, а прыжок не ломает горизонтальный вектор.

## Head Bob
| Параметр | Тип | Дефолт | Описание |
| --- | --- | --- | --- |
| `head_bob_enabled` | bool | `true` | Включить тряску |
| `bob_frequency` | float | `2.0` | Базовая частота шагов |
| `bob_amplitude` | float | `0.06` | Максимальная амплитуда |
| `bob_sprint_multiplier` | float | `1.6` | Усиление частоты при беге |
| `bob_tilt_amount` | float | `0.03` | Наклон камеры по оси Z |

> 💡 **Логика тряски камеры (Head Bob)**
> - **`PI * 2.0` vs `TAU`**: `sin(bob_time * PI * 2.0)` задает полный цикл шага. Замена на `PI*2.0` устраняет зависимость от глобальной константы `TAU`.
> - **`speed_factor`**: `clamp(скорость / base_speed, 0.0, 2.0)`. Масштабирует тряску пропорционально движению. В покое → `0`, на пике → удвоенная амплитуда.
> - **Динамический крен**: `lerp(camera.rotation.z, target_tilt, delta * 8.0)`. Плавный наклон камеры при стрейфе (`-input_dir.x`). `lerp` предотвращает резкие скачки.
> - **Сброс позиции**: При остановке `head.position.lerp(head_default_pos, delta * 6.0)` мягко возвращает камеру в исходную точку.

## Input Actions
| Параметр | Тип | Дефолт | Действие |
| --- | --- | --- | --- |
| `input_left` | String | `ui_left` | Движение влево |
| `input_right` | String | `ui_right` | Движение вправо |
| `input_forward` | String | `ui_up` | Движение вперед |
| `input_back` | String | `ui_down` | Движение назад |
| `input_jump` | String | `ui_accept` | Прыжок |
| `input_sprint` | String | `sprint` | Бег |
| `input_freefly` | String | `freefly` | Переключить noclip |

## 💡 Логика взаимодействия и Noclip
> - **`_update_target()`**: Оптимизированная проверка. Выходит сразу, если `new_target == current_target`. Вызывает `show_prompt()`/`hide_prompt()` только при смене объекта.
> - **`has_method("interact")`**: Безопасный вызов через рефлексию. Если у объекта нет скрипта с `interact()`, действие игнорируется без ошибки.
> - **Freefly**: `collider.disabled = true` отключает физику. `velocity = Vector3.ZERO` обнуляет инерцию, предотвращая неконтролируемый "улет" при переключении.
> - **Разделение потоков**: `_unhandled_input` обрабатывает события мыши/клавиш, `_physics_process` — расчёт кадров. Гарантирует отсутствие потери ввода при просадках FPS.

## Пресеты для хоррора
| Пресет | acceleration | deceleration | base_speed | sprint_speed | bob_amplitude | bob_frequency |
| --- | --- | --- | --- | --- | --- | --- |
| Реализм | 8.0 | 10.0 | 5.0 | 7.5 | 0.04 | 1.8 |
| Тяжелый | 6.0 | 8.0 | 4.0 | 6.0 | 0.05 | 1.6 |
| Паника | 15.0 | 12.0 | 7.0 | 11.0 | 0.09 | 2.5 |
| Отключено | 20.0 | 20.0 | 7.0 | 10.0 | 0.0 | 2.0 |

## Внутренние переменные
| Переменная | Тип | Назначение |
| --- | --- | --- |
| `mouse_captured` | bool | Захвачена ли мышь |
| `look_rotation` | Vector2 | Накопленный поворот камеры |
| `move_speed` | float | Текущая целевая скорость |
| `freeflying` | bool | Активен ли noclip |
| `bob_time` | float | Накопитель фазы синусоиды |
| `head_default_pos` | Vector3 | Точка сброса позиции головы |
| `current_target` | Node | Активный объект взаимодействия |

## Методы
| Метод | Аргументы | Описание |
| --- | --- | --- |
| `_ready()` | - | Инициализация, проверка маппингов, сохранение `head_default_pos` |
| `_unhandled_input(event)` | InputEvent | Захват мыши, поворот, взаимодействие |
| `_physics_process(delta)` | float | Гравитация, инерция, head bob, `move_and_slide()` |
| `_handle_head_bob(...)` | float, Vector2, bool | Расчет фазы, амплитуды и наклона камеры |
| `rotate_look(rot_input)` | Vector2 | Поворот базы и головы, сброс `Basis()` для защиты от дрейфа |
| `enable/disable_freefly()` | - | Переключение `collider` и обнуление скорости |
| `capture/release_mouse()` | - | Управление `Input.MOUSE_MODE_*` |
| `_update_target()` | - | Оптимизированное обновление цели RayCast |
| `check_input_mappings()` | - | Валидация Input Map с авто-отключением функций |

## Константы
| Константа | Тип | Значение | Описание |
| --- | --- | --- | --- |
| `INTERACT_DISTANCE` | float | `3.0` | Дальность `RayCast3D` |

## Зависимости

| Зависимость | Тип | Обязательно |
| :--- | :--- | :--- |
| Head | Node3D | Да |
| Camera3D | Camera3D | Да |
| InteractRay | RayCast3D | Да |
| Collider | CollisionShape3D | Да |
| Input Map: ui_left | Action | Да |
| Input Map: ui_right | Action | Да |
| Input Map: ui_up | Action | Да |
| Input Map: ui_down | Action | Да |
| Input Map: ui_accept | Action | Опционально |
| Input Map: sprint | Action | Опционально |
| Input Map: freefly | Action | Опционально |
| Input Map: interact | Action | Опционально |

# Дата изменения:
31.05.2026
