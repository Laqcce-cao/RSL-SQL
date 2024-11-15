# RSL-SQL: Robust Schema Linking in Text-to-SQL Generation



## Overview

![](figs/framework.png)


## Project directory structure

- Download `pytorch_model.bin` and place it in the `few_shot/sentence_transformers/` folder. Download address: https://huggingface.co/sentence-transformers/all-mpnet-base-v2/tree/main

- Download the `column_meaning.json` file and place it in the `data/` folder. Download address: https://github.com/quge2023/TA-SQL/blob/master/outputs/column_meaning.json

- Download the `test.json` file of the development set in the `data/` folder

- Download the `test_tables.json` file and place it in the `data/` folde

- Download the `train-00000-of-00001-fe8894d41b7815be.parquet` file and place it in the `few_shot/` folder. Download address: https://huggingface.co/datasets/xu3kev/BIRD-SQL-data-train/tree/main/data

- Download the `dev.json` file of the development set in the `few_shot/` folder




```plaintext
laqcce/
├── README.md
├── requirements.txt
│
├── data/
│   ├── column_meaning.json
│   ├── test.json
│   └── test_tables.json
│
├── database/
│   └── test_databases/
│
├── few_shot/
│   ├── sentence_transformers/
│   ├── dev.json
│   └── train-00000-of-00001-fe8894d41b7815be.parquet
│
└── src/
    └── configs/
        └── config.py
```

## environment



```bash
conda create -n rsl_sql python=3.8
conda activate rsl_sql
pip install -r requirements.txt
```
Modify parameter configuration in `src/configs/config.py`

```python
dev_databases_path = 'database/test_databases'
dev_json_path = 'data/dev.json'
api = '..'
base_url = 'http://'
```







## RUN

### 1. Data Preprocessing
```bash
# Construct `ppl_dev.json`. 
python src/data_construct.py 

#Construct few-shot examples pairs,If the examples in the development set can be used as context, then cu is set to 1, otherwise it is 0
python few_shot/construct_QA.py --cu 0/1

# Generate few-shot examples
python few_shot/slg_main.py --dataset src/information/ppl_dev.json --out_file src/information/example.json --kshot 3

# add few-shot examples to ppl_dev.json
python src/information/add_example.py
```



### 2. preliminary sql generation and bidirectional schema linking
```bash
# step 1: preliminary sql
# There are two output files in this step, one is `src/sql_log/preliminary_sql.txt` and the other is `src/schema_linking/LLM.json`
# If an error occurs, you need to save these two files in time, then continue running and save the subsequent results.
python src/step_1_preliminary_sql.py --ppl_file src/information/ppl_dev.json --sql_out_file src/sql_log/preliminary_sql.txt --Schema_linking_LLM src/schema_linking/LLM.json --start_index 0

# schema linking
python src/bid_schema_linking.py --pre_sql_file src/sql_log/preliminary_sql.txt --sql_sl_output src/schema_linking/sql.json --hint_sl_output src/schema_linking/hint.json --LLM_sl_output src/schema_linking/LLM.json --Schema_linking_output src/schema_linking/schema.json
cp src/schema_linking/schema.json src/information

# add schema linking to ppl_dev.json
python src/information/add_sl.py
```

### 3. SQL Generation based simplified schema and Information augmentation
```bash
# step 2: sql generation
# There are two output files in this step, one is `src/sql_log/step_2_information_augmentation.txt` and the other is `src/information/augmentation.json`
# If an error occurs, you need to save these two files in time, then continue running and save the subsequent results.
python src/step_2_information_augmentation.py --ppl_file src/information/ppl_dev.json --sql_2_output src/sql_log/step_2_information_augmentation.txt --information_output src/information/augmentation.json --start_index 0

# add augmentation to ppl_dev.json
python src/information/add_augmentation.py
```

### 4. SQL selection
```bash
# step 3: sql selection
# There is one output files in this step, one is `src/sql_log/step_3_binary.txt`.
# If an error occurs, you need to save these two files in time, then continue running and save the subsequent results.
python src/step_3_binary_selection.py --ppl_file src/information/ppl_dev.json --sql_3_output src/sql_log/step_3_binary.txt --sql_1 src/sql_log/preliminary_sql.txt --sql_2 src/sql_log/step_2_information_augmentation.txt --start_index 0
```

### 5. SQL refinement
```bash
# step 4: sql refinement
# There is one output files in this step, one is `src/sql_log/final_sql.txt`.
python src/step_4_self_correction.py --ppl_file src/information/ppl_dev.json --sql_4_output src/sql_log/final_sql.txt --sql_refinement src/sql_log/step_3_binary.txt --start_index 0
```
