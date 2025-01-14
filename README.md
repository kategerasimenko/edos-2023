# LCT-1 at SemEval-2023 Task 10: Pre-training and Multi-task Learning for Sexism Detection and Classification

This is repository of the LCT-1 team submission at SemEval-2023 Task 10 (EDOS 2023). <br>
Authors: Konstantin Chernyshev, Ekaterina Garanina, Duygu Bayram, Qiankun Zheng, Lukas Edman (University of Groningen, the Netherlands).

## Task description

The task is to detect and classify sexist posts in English text from Redding and Gab platform. The task is divided into 3 hierarchical subtasks:

* TASK A - **Binary Sexism Detection**: a binary classification where systems have to predict whether a post is sexist or not sexist.
* TASK B - **Category of Sexism**: for posts which are sexist, a four-class classification where systems have to predict one of four categories: (1) threats, (2)  derogation, (3) animosity, (4) prejudiced discussions. 
* TASK C - **Fine-grained Vector of Sexism**: for posts which are sexist, an 11-class classification where systems have to predict one of 11 fine-grained vectors.

Full task description is available via [CodaLab](https://codalab.lisn.upsaclay.fr/competitions/7124).


## Requirements

Python >= 3.9 is required. During training version 3.9 was used.

We recommend to use a virtual environment:
```shell
python -m venv .venv
source .venv/bin/activate
pip install -U -r requirements.txt
```

## Data

The data is not included in the repository because of access restrictions for some datasets. 
In order to reproduce our results, you should collect the data and place it according to the following structure:

```
edos_data
└───raw
    │   edos_labelled_aggregated.csv
    │   gab_1M_unlabelled.csv
    │   reddit_1M_unlabelled.csv

multitask_data
└───raw
    │
    └───hate_speech
    │   └───HaSpeeDe_2018
    │   └───HaSpeeDe_2020
    │   └───hateval2019
    │   └───Jigsaw_unintended_bias_civil_comments
    │   └───measuring_hate_speech
    │   └───OLIDv1.0
    │
    └───misogyny
        └───AMI_EVALITA
            └───AMI2018_ELG
            └───AMI2020
        └───call_me_sexist
        └───EXIST2021
        └───Guest_et_al_online_misogyny
```

Afterwards you need to run the following notebooks and scripts in the `data_preprocessing/` folder to preprocess the data:
* `edos_dataset_split.ipynb` - to split the task data into _train_ and _eval_ sets and write the processed files to `edos_data/processed`.
* `process_datasets.ipynb` - to process all HS datasets (written to `multitask_data/formatted`) and reorganize them into tasks for multi-task learning (written to `multitask_data/processed`).
* `preprocessing.py` - to preprocess all data (masking, emoji normalisation, space normalisation). Preprocessed text is written to `text_preprocessed` column of each csv file.
* `machamp_preprocess.py` - to prepare the MTL data for MaChAmp (written to `multitask_data/machamp`).


## Experiments

We used [neptune.ai](https://neptune.ai/) for experiment tracking. In order to use it, you need to create an account 
and set up the API token and a project name, where the experiment results will be stored.


### Further pre-training

`training/dont_stop_pretraining/train.py` is the main script for further pre-training of a model using MLM task.

To run TAPT on EDOS, DAPT on 2M, and DAPT on 2M+HS accordingly:
```shell
python training/dont_stop_pretraining/train.py --base-model="roberta-large" --task-name=edos --batch-size=32 --max-epochs=10 --eval-steps=298  --preprocessing-mode=basic
python training/dont_stop_pretraining/train.py --base-model="roberta-large" --task-name=2M --batch-size=24 --max-epochs=5 --eval-steps=10000 --preprocessing-mode=basic
python training/dont_stop_pretraining/train.py --base-model="roberta-large" --task-name=2M_hate --batch-size=24 --max-epochs=5 --eval-steps=10000 --preprocessing-mode=basic
```

Main parameters:
* `--base-model` is the name of the pre-trained model (local or from HF Hub);
* `--task-name` is the dataset collection to pre-train on (`edos`, `2M`, `2M_hate` - stands for edos data only, 2M data provided by task organizers and 2M data + HS data described in the paper);
* `--preprocessing-mode` is the parameter to choose the preprocessing mode for the data:
    * `basic` - masking and space normalisation;
    * `full` - masking, emoji normalisation and space normalisation;

Use `python training/dont_stop_pretraining/train.py --help` or check the file to see other customizable parameters.


### Fine-tuning

`training/finetuning/train.py` is a universal script for fine-tuning any local or HF Hub model on EDOS data.

Main parameters:
* `--base-model` is the name of the pre-trained model (local or from HF Hub);
* `--edos-task` is the task to train on (A, B, C);
* `--config-name` is the key to the parameter configuration in `training/finetuning/edos_eval_params.json`
    * `default` was used to train initial baselines and compare between different available pre-trained models of `base`-size;
    * `updated-large` was used for fine-tuning of DAPT and TAPT `large`-sized models.

Use `python training/finetuning/train.py --help` or check the file to see other customizable parameters.

Example of fine-tuning a DAPT model:
```shell
BASE_MODEL=models/dont_stop_pretraining/dont_stop_pretraining-roberta-large-2M-basic_preproc
python training/finetuning/train.py --base-model=$BASE_MODEL --edos-task=A --config-name="updated-large"
```


### Multi-task learning

We have customized [MaChAmp](https://github.com/machamp-nlp/machamp) code (version: commit 086dcbc) to include integration with Neptune to keep track of experiments. 
The customized code is located in `machamp_repo` folder.

Example of one MTL experiment:
```shell
python training/machamp/train.py --edos-task=A --datasets=toxicity_binary,sexism_binary --languages=en
```

Example of subsequent fine-tuning after MTL training:
```shell
python training/machamp/train.py --edos-task=A --datasets=toxicity_binary,sexism_binary --languages=en --finetune-only --max-finetune-epochs=10 --finetune-learning-rate=0.000001
```

Example of running inference:
```shell
MODEL=models/machamp/multitask_A_dont_stop_pretraining-roberta-large-2M-basic_preproc_hatebinary-edosA_en_20epochs_5e-06lr_4bs
python training/machamp/predict.py --task=A --model=$MODEL
```

The list of the datasets is available in `training/machamp/datasets.json`. The dataset combinations reported in the paper are:
* Task A: `offense_binary` + fine-tuning, `hate_target,offense_binary`, `hate_binary` + fine-tuning;
* Task B: `edos_C,evalita` + fine-tuning, `exist,edos_C,evalita`;
* Task C: `edos_B,sexist` + fine-tuning, `edos_B,sexist`.

`training/machamp/train.py` also contains other customizable parameters.


### Preprocessing experiments

Example of a preprocessing experiment:
```shell
python training/preprocessing_test/train.py --base-model="roberta-base" --preprocess-masks --preprocess-hashtags --preprocess-emoji --preprocess-spaces
```

Main parameters:
* `--base-model` is the name of the pre-trained model (local or from HF Hub);
* The flags `--preprocess-masks`, `--preprocess-hashtags`, `--preprocess-emoji`, `--preprocess-spaces` control the preprocessing steps.

Use `python training/preprocessing_test/train.py --help` or check the file to see other customizable parameters.


## Results

The results and analysis will be available later.


## Acknowledgements

* We would like to thank Tommaso Caselli for the insightful supervision and multiple wonderful ideas.
* We would like to thank the Center for Information Technology of the University of Groningen for their support and for providing access to the Peregrine high-performance computing cluster.
* The first four authors are supported by the Erasmus Mundus Master's Programme ["Language and Communication Technologies"](https://lct-master.org).
