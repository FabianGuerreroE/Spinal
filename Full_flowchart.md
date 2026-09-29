# Diagramme de blocs de Spinal

Spinal traite deux sources indépendantes jusqu'à l'étape d'analyse finale :

- signaux PPG multicanaux et multi-longueurs d'onde ;

- vidéos de moniteurs médicaux, numérisées par segmentation et OCR.

## Flux principal

```mermaid
flowchart TD
    subgraph ENTRADAS[Entrées et ressources]
        PPG0["CSV PPG brut"]
        VID0["Vidéos de moniteurs"]
        EVENTS["Événements CSV"]
        HEADER["header.csv"]
        MAP["resources/time_mapping.json"]
        BOXES["resources/reference_boxes.json"]
    end

    subgraph PPG["Branche PPG et qualité du signal"]
        CONVERT["convert_2022.py\n(optionnel)"]
        CLEAN["preproc/clean.py\n+ io_utils.py\n+ signal_utils.py"]
        CLEANED["output/cleaned/YYYY-MM-DD.npz\nPPG propre + respiration"]
        INFER["sqa/infer_sqi.py\nConvNeXt1D + régression Beta"]
        RAW["SQI par fenêtres\noutput/ppg_sqi/YYYY-MM-DD_sqi_windows.npz"]
        POST["sqa/postprocess_sqi.py\nfenêtres de 8 s -> blocs de 2 s"]
        SQI["SQI post-traité\noutput/SpinalMot/YYYY-MM-DD_sqi_windows.npz"]

        PPG0 --> CONVERT
        PPG0 --> CLEAN
        CONVERT --> CLEAN
        CLEAN --> CLEANED
        CLEANED --> INFER
        linkStyle 0,1,2,3,4 stroke:#FF0000,stroke-width:2px
        INFER --> RAW
        RAW --> POST
        POST --> SQI
        linkStyle 5,6,7 stroke:#FF6060,stroke-width:2px
    end

    subgraph VIDEO["Branche de numérisation vidéo"]
        SPLIT["digitalisation/split_video.py\n(optionnel)"]
        ANNOTATE["digitalisation/annotate.py\nsélection des régions d'intérêt"]
        SEGMENT["digitalisation/segment.py\nSAM 3 : suivi des boîtes"]
        TRACKED["tracked_boxes_per_frame_*.json"]
        OCR["digitalisation/ocr.py\nPARSeq + validation physiologique"]
        DIGITS["*_digits.csv\nvaleurs par frame"]
        CLEAN_DIGITS["digitalisation/clean_metrcis.py\nalignement + interpolation courte + baseline"]
        REFERENCE["data/digit/numbers/<DATE>/\nréférence numérisée propre"]

        VID0 --> SPLIT
        VID0 --> ANNOTATE
        VID0 --> SEGMENT
        SPLIT --> MAP
        ANNOTATE --> BOXES
        BOXES --> SEGMENT
        SEGMENT --> TRACKED
        TRACKED --> OCR
        VID0 --> OCR
        OCR --> DIGITS
        DIGITS --> CLEAN_DIGITS
        MAP --> CLEAN_DIGITS
        CLEAN_DIGITS --> REFERENCE
        linkStyle 8,9,10,11,12,13,14,15,16,17,18,19,20 stroke:#00FF00,stroke-width:2px
    end

    subgraph TRAIN["Entraînement et évaluation optionnels"]
        DEEPBEAT["data/deepbeat"]
        PREP_DB["preproc/prepare_deepbeat.py"]
        DB_DATA["processed_deepbeat/*.npz"]
        PREP_FT["preproc/prep_finetuning.py\nfenêtres + labels faibles"]
        FT_DATA["processed_finetune/*.npz"]
        TRAIN_MODEL["sqa/train.py\nclassification ou régression Beta"]
        CHECKPOINT["checkpoints/*.ckpt"]
        EVAL["sqa/eval.py\nmétriques + latents"]
        EXPLORE["sqa/explore_sqi.py\nUMAP des latents"]

        DEEPBEAT --> PREP_DB --> DB_DATA --> TRAIN_MODEL
        linkStyle 21,22,23 stroke:#FFFF00,stroke-width:2px
        CLEANED --> PREP_FT
        linkStyle 24 stroke:#FF0000,stroke-width:2px
        EVENTS --> PREP_FT
        linkStyle 25 stroke:#0000FF,stroke-width:2px
        PREP_FT --> FT_DATA --> TRAIN_MODEL
        linkStyle 26,27 stroke:#FF00FF,stroke-width:2px
        TRAIN_MODEL --> CHECKPOINT
        linkStyle 28 stroke:#FF8080,stroke-width:2px
        CHECKPOINT --> INFER
        CHECKPOINT --> EVAL
        EVAL --> EXPLORE
        linkStyle 29,30,31 stroke:#FF8080,stroke-width:2px
    end

    subgraph ANALYSIS["Analyse finale"]
        ANALYZE["data_analysis/run_analysis.py"]
        METRICS["metrics.py\nmetrics_analysis.py\nppg_analysis.py\nsqi_analysis.py"]
        EVENT_UTILS["utils/event_utils.py"]
        RESULTS["output/analyse/\nCSV + graphiques"]

        SQI --> ANALYZE
        linkStyle 32 stroke:#FF6060,stroke-width:2px
        CLEANED --> ANALYZE
        linkStyle 33 stroke:#FF0000,stroke-width:2px
        REFERENCE --> ANALYZE
        linkStyle 34 stroke:#00FF00,stroke-width:2px
        EVENTS --> ANALYZE
        linkStyle 35 stroke:#0000FF,stroke-width:2px
        HEADER --> ANALYZE
        linkStyle 36 stroke:#00FFFF,stroke-width:2px
        MAP --> ANALYZE
        linkStyle 37 stroke:#00FF00,stroke-width:2px
        METRICS --> ANALYZE
        EVENT_UTILS --> ANALYZE
        ANALYZE --> RESULTS
        linkStyle 40 stroke:#F8F880,stroke-width:3px
    end
```

## Lecture du diagramme
1. `clean.py` transforme les CSV PPG en fichiers `.npz` avec un signal propre et la respiration.
2. `infer_sqi.py` applique le modèle entraîné sur des fenêtres de 8 secondes ; `postprocess_sqi.py` les réorganise en blocs temporels de 2 secondes.
3. La branche vidéo génère une référence indépendante : `annotate.py` définit les ROI, `segment.py` les suit avec SAM 3, `ocr.py` extrait les nombres et `clean_metrcis.py` les aligne temporellement.
4. `run_analysis.py` est le point de convergence. Il calcule les métriques PPG, compare la qualité du signal et met en relation les résultats avec la référence numérisée et les événements.
5. `train.py`, `eval.py` et `explore_sqi.py` sont des outils optionnels pour entraîner, évaluer et explorer le modèle SQI ; ils ne font pas obligatoirement partie de l'analyse d'une session.

## Módulos transversales

- `src/dataset/utils/io_utils.py`: lecture de PPG, métadonnées et conversions temporelles.
- `src/dataset/utils/signal_utils.py`: filtres, rééchantillonnage, normalisation et opérations sur les blocs.
- `src/dataset/utils/event_utils.py`: événements et zones `PRE`, `DURING` et `POST`.
- `src/dataset/sqa/core_utils.py`: chargement et normalisation des checkpoints.
- `src/digitalisation/utils.py`: ID d'écran, plages physiologiques et opérations sur les ROI.
- `src/dataset/utils/visualization/`: graphiques de diagnostic de l'ETL et du SQI.

La branche PPG et la branche vidéo restent indépendantes jusqu'à `run_analysis.py`.
