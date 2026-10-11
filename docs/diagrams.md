# Диаграммы модуля Simulation Manager и Тестовый комплекс

## 1. Текущая структура 
Показана структура взаимодействия модулей работы в `Simulation_manager`

> Актуальный файл с редактора Drawio для удоства изучения [docs/Simulator_manager.drawio](docs/Simulator_manager.drawio).


![alt text](image.png)


## 2. Типовой сценарий (AS-IS)
Сценарий: Исполнение единичного миллисекундного такта TTI (1 мс) внутри главного цикла `start_simulation()` при вызове `run_tti(current_time)`.

```mermaid
sequenceDiagram
    autonumber
    participant SM as SimulationManager
    participant UE as UECollection
    participant BS as BaseStation (BufferManager)
    participant TR as PacketManager
    participant SCH as SchedulerInterface
    participant ST as StatsManager

    Note over SM: Начало миллисекундного такта (TTI)
    SM->>SM: Установка GLOBALS.CURRENT_TIME = tti
    
    rect rgb(240, 245, 255)
        Note over SM,UE: 1. Кинематика и физика радиоканала
        SM->>UE: UPDATE_ALL_USERS(current_time, intervals...)
        UE->>UE: Перемещение абонентов (MobilityModel)
        UE->>UE: Расчет затухания, SINR и CQI (ChannelModel)
    end

    rect rgb(245, 255, 240)
        Note over SM,BS: 2. Протокольные очереди и генерация нагрузки
        SM->>BS: upd_buffers_all() (Проверка PDCP Discard таймеров)
        loop Для каждого активного UE
            SM->>TR: generate_packets(ue_id, current_time)
            TR-->>SM: Список сгенерированных Packet
            SM->>BS: add_packet(ue_id, pkt)
        end
    end

    rect rgb(255, 248, 240)
        Note over SM,SCH: 3. Распределение радиоресурсов (L2/MAC)
        SM->>UE: GET_USERS_FOR_SCHEDULER()
        UE-->>SM: Список абонентов с отчетами CQI и буферами
        SM->>SCH: schedule(current_time, users)
        SCH->>SCH: Фильтрация, ранжирование, CCE, аллокация RBG
        SCH->>BS: extract_data(tbs_bytes) (Списание переданных байт)
        SCH-->>SM: sched_result (гранты и статистика такта)
    end

    rect rgb(255, 240, 245)
        Note over SM,ST: 4. Сбор телеметрии (каждые collect_interval TTI)
        opt tti % collect_interval == 0
            SM->>ST: collect(tti)
            ST->>SCH: get_stats()
            ST->>SCH: amc.get_stats()
            ST->>SCH: pdcch_manager.get_stats()
            ST->>ST: Запись snapshot в history
        end
    end
```
