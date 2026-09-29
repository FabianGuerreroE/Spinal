```mermaid
flowchart LR
    A["2026-09-28.npz<br/>(clean.py: ppg, respi, time)"] --> B["infer_sqi.py"]
    CKPT["sqi_beta_final.ckpt<br/>(train.py)"] --> B
    
    B --> C["2026-09-28_sqi_windows.npz<br/>(SQI por ventanas de 8 s)"]
    
    C --> D["postprocess_sqi.py"]
    D --> E["2026-09-28_postprocessed.npz<br/>(block_sqi_postprocessed: bloques de 2 s<br/>+ estados stable / spike / transient / transition)"]
    D --> F["Gráficos de Diagnóstico 2 s<br/>(*_sqi_2s.png)"]
end
```
