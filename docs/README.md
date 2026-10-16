# Модуль Simulation Manager и Тестовый комплекс

## 1. Назначение
Модуль является центральным ядром-оркестратором симулятора **PyScheduler**, управляющим единым циклом дискретно-событийного моделирования и связывающим воедино абонентов, физику радиоканала, генераторы сетевого трафика, очереди базовой станции и MAC-планировщик. Модуль инкапсулирует подсистему агрегации, валидации и экспорта сетевой телеметрии (`StatsManager`), а также содержит демонстрационный тестовый модуль (`TEST_MODULES.py`) и регрессионный тестовый комплекс (`tests/`) для валидации сетевых протоколов и радиоинтерфейса LTE.

## 2. Основной функционал

- Управляет миллисекундным тактом TTI (`GLOBALS.CURRENT_TIME`) и сквозным конвейером выполнения шага симуляции в методе `run_tti`.
- Координирует жизненный цикл симуляции: валидацию конфигурации, создание ресурсной сетки `RES_GRID_LTE` и планировщика `SchedulerInterface`, инициализацию очередей абонентов в `BaseStation.buffer_manager`.
- Вызывает с заданной периодичностью обновление кинематики абонентов, расчет радиоканала, сброс просроченных пакетов и генерацию нового трафика.
- Опрашивает телеметрию планировщика, AMC и PDCCH, рассчитывает агрегированные метрики (пропускная способность, загрузка PRB, индекс справедливости Джайна) и экспортирует результаты в CSV/JSON (`StatsManager`).
- Предоставляет единый стенд запуска демонстрационных сценариев (`sim_with_manager`, `sim_with_manager_qos`) и комплекс автоматизированного тестирования на базе `pytest`.

## 3. Входы, выходы и зависимости

| Вход | Откуда | Формат/смысл |
|---|---|---|
| `BaseStation` | `PyScheduler/BS_MODULE.py::BaseStation` | Экземпляр БС: полоса (МГц), тип буфера (`use_simple_buffer`), лимиты очередей |
| `UECollection` | `PyScheduler/UE_MODULE.py::UECollection` | Пул абонентов со свойствами движения, радиоканала и буферов |
| `SchedulerConfig` | Пользователь / Конфигурация | Параметры MAC-планировщика: алгоритм (`RoundRobin`, `ProportionalFair` и др.), лимиты CCE, окно |
| `SimulationConfig` | Пользователь / Конфигурация | Тайминги: `sim_duration` (мс), `update_interval`, интервалы мобильности и канала |
| `traffic_profiles` | Пользователь / `PyScheduler/TRAFFIC_MODEL.py` | Список профилей биреров с привязкой QCI и параметров распределений |
| Телеметрия такта | `SCHEDULER.py`, `AMC`, `PDCCHManager` | Словари `get_stats()` с числом активных UE, выделенными RB, TBS и CCE |

| Выход | Куда | Формат/смысл |
|---|---|---|
| `sched_result: dict` | `PyScheduler/SIMULATION_MANAGER.py::SimulationManager.run_tti` -> вызывающий код | Словарь грантов и маппинга RB на текущем TTI |
| `manager_stats.csv` | `PyScheduler/SIMULATION_MANAGER.py::StatsManager.export_csv` -> ФС / `PLOT_METRICS.py` | Временной ряд интегральных метрик симуляции (Tput, PRB, JFI, задержки) |
| `manager_stats_detailed.csv` | `PyScheduler/SIMULATION_MANAGER.py::StatsManager.export_detailed_csv` -> ФС | Детализированные per-UE метрики (через точку с запятой, локаль RU/EN) |
| `output.txt` | `PyScheduler/SIMULATION_MANAGER.py::SimulationManager.start_simulation` -> ФС | Перенаправленный консольный лог симуляции (при `_to_file=True`) |
| Отчет прогона тестов | `PyScheduler/tests` -> Консоль / CI |  |

Зависимости:
- Внутренние модули симулятора: `GLOBALS`, `BS_MODULE`, `UE_MODULE`, `SCHEDULER`, `RES_GRID`, `TRAFFIC_MODEL`, `MOBILITY_MODEL`.
- Сторонние библиотеки Python: `numpy`, `tqdm`, `matplotlib` (для графиков в `TEST_MODULES.py`), `pytest` (для запуска тестов).

## 4. Основные компоненты

| Файл / класс / функция | Назначение | Где используется |
|---|---|---|
| `PyScheduler/SIMULATION_MANAGER.py::SimulationManager` | Центральный класс оркестрации симуляции, фасад настройки и запуска | `TEST_MODULES.py`, сценарии моделирования, будущий CLI |
| `PyScheduler/SIMULATION_MANAGER.py::SimulationManager.start_simulation` | Инициализация компонентов и выполнение главного цикла TTI с прогресс-баром | Основная точка входа для старта моделирования |
| `PyScheduler/SIMULATION_MANAGER.py::SimulationManager.run_tti` | Исполнение единичного шага дискретного времени (физика -> буферы -> трафик -> планирование) | Внутренний цикл `start_simulation`, пошаговые интеграционные тесты |
| `PyScheduler/SIMULATION_MANAGER.py::SimulationConfig` | Датакласс настроек таймингов симуляции и интервалов обновления физики | `SimulationManager.__init__` |
| `PyScheduler/SIMULATION_MANAGER.py::SchedulerConfig` | Датакласс конфигурации MAC-планировщика (алгоритм, лимиты UE/CCE, окно) | `SimulationManager.set_scheduler` |
| `PyScheduler/SIMULATION_MANAGER.py::StatsManagerConfig` | Датакласс параметров телеметрии (интервал сбора, уровни детализации, префикс) | `SimulationManager.set_stats_manager` | 
| `PyScheduler/SIMULATION_MANAGER.py::StatsManager` | Сборщик телеметрии с планировщика/AMC/PDCCH, расчет Jain's Fairness Index и экспорт | `SimulationManager.start_simulation` |
| `PyScheduler/SIMULATION_MANAGER.py::MetricLevel` | Перечисление уровней сбора метрик: `NONE`, `BASIC`, `ADVANCED`, `FULL` | `LevelsConfig`, `StatsManager.collect` |
| `PyScheduler/TEST_MODULES.py::sim_with_manager_qos` | Эталонный скрипт многопользовательской симуляции с QoS и очередями RLC UM | Основная точка входа в `TEST_MODULES.py::__main__` |
| `PyScheduler/TEST_MODULES.py::sim_with_manager` | Демонстрационный скрипт симуляции с `SimpleBuffer` и отрисовкой графиков | Точка ручной проверки радиомоделей |
| `PyScheduler/TEST_MODULES.py::visualize_users_mobility` | Отрисовка 2D-траекторий перемещения абонентов на карте соты через matplotlib | `TEST_MODULES.py::sim_with_manager` |
| `PyScheduler/TEST_MODULES.py::visualize_users_sinr` | Построение временного графика SINR абонентов во времени | `TEST_MODULES.py::sim_with_manager` |
| `PyScheduler/tests/layered_buffer_tests` | Пакет из 7 тестовых модулей очередей RLC UM, мультиплексирования и PDCP Discard | Регрессионный запуск `pytest` |
| `PyScheduler/tests/mobility_module_tests` | Пакет из 5 тестовых модулей кинематики (RandomWalk, Waypoint, Gauss-Markov, стресс) | Регрессионный запуск `pytest` |
| `PyScheduler/tests/scheduler_tests` | Пакет тестов инвариантов распределения RBG и бенчмаркинга FD-алгоритмов | Регрессионный запуск `pytest` |

## 5. Один типичный сценарий
Сценарий запуска симуляции `sim_with_manager_qos()`:
1. **Инициализация топологии сети:** Скрипт задает границы полигона `MapBorders(-1000, 1000, -1000, 1000)`, создает базовую станцию `BaseStation(bandwidth=10, ch_model_type='UMa', use_simple_buffer=False)` и генерирует 5 абонентов в `UECollection` с привязкой к БС.
2. **Параметризация симулятора:** Создается экземпляр `SimulationManager`. Через сеттеры задаются БС, коллекция UE, алгоритм `ProportionalFair`, длительность 5000 мс, профили трафика (потоки QCI 1, 4, 5, 6, 7, 8 через `setup_traffic_profiles`) и конфигурация сбора метрик.
3. **Запуск и выполнение TTI:** Вызывается `sim.start_simulation()`. Метод валидирует параметры (`_check_required_parameters`), выделяет ресурсную сетку `RES_GRID_LTE` и планировщик `SchedulerInterface`, инициализирует буферы RLC и запускает цикл `for tti in range(5000)`. На каждом TTI метод `run_tti` обновляет координаты и канал пользователей, генерирует пакеты, добавляет их в буфер БС и вызывает диспетчеризацию `scheduler.schedule()`.
4. **Фиксация телеметрии и финализация:** Каждые 10 TTI `StatsManager.collect(tti)` фиксирует состояние сети, периодически обновляется счетчик Jain's Fairness Index в прогресс-баре. По завершении 5000 TTI данные выгружаются в `emp_stats.csv` и `emp_stats_detailed.csv`, закрываются прогресс-бары и восстанавливаются системные потоки.


## 6. Связанные материалы
- [Диаграммы архитектуры](docs/diagrams.md)
- [Ревизия кода и реестр наблюдений](docs/audit.md)
- [План развития модуля ](docs/features.md)
