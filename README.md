# PARADE

[![DOI](https://zenodo.org/badge/DOI/10.5281/zenodo.3974431.svg)](https://doi.org/10.5281/zenodo.3974431)

This repository contains the code for our paper
- PARADE: Passage Representation Aggregation for Document Reranking

If you're interested in how to run PARADE for the TREC-COVID challenge, 
please check out the **covid** branch.

## Introduction
PARADE (PAssage Representation Aggregation for Document rE-ranking) is a document reranking model based on the pre-trained language models.

We support the following PARADE variants:
- PARADE-Avg (named `cls_avg` in the code)
- PARADE-Attn (named `cls_attn` in the code)
- PARADE-Max (named `cls_max` in the code)
- PARADE (named `cls_transformer` in the code)

We support two instantiations of pre-trained models:
- BERT
- ELECTRA

## Getting Started
To run PARADE, there're three steps ahead.
We give a detailed example on how to run the code the Robust04 dataset using the title query.

### 1. Data Preparation
To run a 5-fold cross-validation, data for 5 folds are required.
The standard `qrels`, `query`, `trec_run` files can be accomplished by [Anserini](https://github.com/castorini/anserini),
please check out their notebook for further details.
Then you need to split the documets into passages, write them into TFrecord files.
The `corpus` file can also be extracted by Anserini to form a `docno \t content` paired text.

```bash
export max_num_train_instance_perquery=1000
export rerank_threshold=100
export BERT_PRETRAINED_DIR="gs://cloud-tpu-checkpoints/bert"
export num_segment=16

for fold in {1..5}
do
  export output_dir="/data2/robust/runs/title/training.data/num-segment-${num_segment}/fold-${fold}-train-${max_num_train_instance_perquery}-test-${rerank_threshold}
  mkdir -p $output_dir 

  python3 generate_data.py \
    --trec_run_filename=/data2/robust04/runs/title/run.robust04.title.bm25.txt \
    --qrels_filename=/data/anserini/src/main/resources/topics-and-qrels/qrels.robust04.txt \
    --query_filename=/data/anserini/src/main/resources/topics-and-qrels/topics.robust04.txt \
    --query_field=title \
    --corpus_filename=/data2/robust04/runs/title/run.robust04.title.bm25.txt.docno.uniq_rawdocs.txt \
    --output_dir=$output_dir \
    --dataset=robust04 \
    --fold=$fold \
    --vocab_filename=${BERT_PRETRAINED_DIR}/uncased_L-24_H-1024_A-16/vocab.txt \
    --plen=150 \
    --overlap=50 \
    --max_seq_length=256 \
    --max_num_segments_perdoc=${num_segment} \
    --max_num_train_instance_perquery=$max_num_train_instance_perquery \
    --rerank_threshold=$rerank_threshold 
done
```
You should be able to see 5 sub-folders generated in the `output_dir` folder,
with each contains a train file and a test file.
Note that if you're going to run the code on TPU, you need to upload the training/testing data to Google Cloud Storage (GCS).

The above snippet is also available in `scripts/run.convert.data.sh`.

If you bother getting the raw text from Anserini, 
you can also replace the `anserini/src/main/java/io/anserini/index/IndexUtils.java` file by the `extra/IndexUtils.java` file in this repo,
then re-build Anserini (version 0.7.0).
Below is how we fetch the raw text
```bash
anserini_path="path_to_anserini"
index_path="path_to_index"
# say you're given a BM25 run file run.BM25.txt
cut -d ' ' -f3 run.BM25.txt | sort | uniq > docnolist
${anserini_path}/target/appassembler/bin/IndexUtils -dumpTransformedDocBatch docnolist -index ${index_path}
```
then you get the required raw text in the directory that contains docnolist. 
Alternatively, you can refer to the `search_pyserini.py` file in the **covid** branch and fetch the docs using pyserini.
Everything is prepared now!

### 2. Model Traning and Evaluation


For all the pre-trained models, we first fine-tune them on the MSMARCO passage collection.
This is IMPORTANT, as it can improve the nDCG@20 by 2 points generally.
To figure out the way of doing that, please check out [dl4marco-bert](https://github.com/nyu-dl/dl4marco-bert).
The fine-tuned model will be the initialized model in PARADE.
Just pass it to the `BERT_ckpt` argument in the following snippet. 
If you want to escape this fine-tuning step,
check out these fine-tuned [models](#resource) on the MSMARCO passage ranking dataset.
Now train the model:

```bash
export method=cls_transformer
export tpu_name=node-5
export field=title
export dataset=robust04
export num_segment=16
export max_num_train_instance_perquery=1000
export rerank_threshold=100
export learning_rate=3e-6
export epoch=3

export BERT_config=gs://cloud-tpu-checkpoints/bert/uncased_L-12_H-768_A-12/bert_config.json 
export BERT_ckpt="Your_path_to_the_pretrain_ckpt"
export runid=electra_base_onMSMARCO_${method}

for fold in {1..5}
do
  export data_dir="gs://Your_gs_bucket/adhoc/training.data/$dataset/$field/num-segment-${num_segment}/fold-$fold-train-$max_num_train_instance_perquery-test-$rerank_threshold"
  export output_dir="gs://Your_gs_bucket/adhoc/experiment/$dataset/$field/num-segment-${num_segment}/$runid/fold-$fold"

  python3 -u run_reranking.py \
    --pretrained_model=electra \
    --do_train=True \
    --do_eval=True \
    --train_batch_size=32 \
    --eval_batch_size=32 \
    --learning_rate=$learning_rate \
    --num_train_epochs=$epoch \
    --warmup_proportion=0.1 \
    --aggregation_method=$method \
    --dataset=$dataset \
    --fold=$fold \
    --trec_run_filename=/data2/robust04/runs/title/run.robust04.title.bm25.txt \
    --qrels_filename=/data/anserini/src/main/resources/topics-and-qrels/qrels.robust04.txt \
    --bert_config_filename=${BERT_config} \
    --init_checkpoint=${BERT_ckpt} \
    --data_dir=$data_dir \
    --output_dir=$output_dir \
    --max_seq_length=256 \
    --max_num_segments_perdoc=${num_segment} \
    --max_num_train_instance_perquery=$max_num_train_instance_perquery \
    --rerank_threshold=$rerank_threshold \
    --use_tpu=True \
    --tpu_name=$tpu_name 
done
```
After the 5-fold training/testing,
you need to download the results from GCS, merge them, and evaluate them!
```bash
epoch=3
gs_dir=gs://Your_gs_bucket/adhoc/experiment/$dataset/$field/num-segment-${num_segment}/$runid
local_dir=/data2/$dataset/reruns/$field/num-segment-${num_segment}/$runid
qrels_path=/data/anserini/src/main/resources/topics-and-qrels/qrels.${dataset}.txt
mkdir -p $local_dir

# download 5-fold resutls from GCS
# if you're in a local machine, you don't need to download
for fold in {1..5}
do
  gsutil cp $gs_dir/fold-${fold}/fold_*epoch_${epoch}_bert_predictions_test.txt $local_dir
done

# make sure that we find out all the 5 result files
num_result=$(ls $local_dir |wc -l)
if [ "$num_result" != "5" ]; then
  echo Exit. Wrong number of results. Expect 5 but  $num_result result files found!
  exit
fi

# evaluate the result using trec_eval
cat ${local_dir}/fold_*epoch_${epoch}_bert_predictions_test.txt >> ${local_dir}/merge_epoch${epoch}
/data/tool/trec_eval-9.0.7/trec_eval ${qrels_path} ${local_dir}/merge_epoch${epoch} -m ndcg_cut.20 -m P.20  >> ${local_dir}/result_epoch${epoch}
cat ${local_dir}/result_epoch${epoch}
```
The model performance will automatically output on your screen. 
When searching the title queries on the Robust04 collecting, it outputs
```
P_20                    all     0.4604
ndcg_cut_20             all     0.5399
```
Looks good!
The above steps can also be done all at once by running `scripts/run.reranking.sh`.

### 3. Significance Test
To do a significance test, just configurate the `trec_eval` path in the `evaluation.py` file. 
Then simply run the following command, here we compare PARADE with BERT-MaxP:
```
python evaluation.py \
  --qrels /data/anserini/src/main/resources/topics-and-qrels/qrels.robust04.txt \
  --baselines /data2/robust04/reruns/title/bertmaxp.dai/bertbase_onMSMARCO/merge \
  --runs /data2/robust04/reruns/title/num-segment-16/electra_base_onMSMARCO_cls_transformer/merge_epoch3
```
then it outputs
```
OrderedDict([('P_20', '0.4277'), ('ndcg_cut_20', '0.4931')])
OrderedDict([('P_20', '0.4604'), ('ndcg_cut_20', '0.5399')])
OrderedDict([('P_20', 1.2993259211924425e-11), ('ndcg_cut_20', 8.306604295574242e-09)])
```
The upper two lines are the sanity checks of your run performance values.
The last line shows the p-values.
PARADE achieves significant improvement over BERT-MaxP (p < 0.01) !

### Knowledge Distillation (Optional)
You can also perform knowledge distillation for the smaller PARADE models.
Please follow the above steps to fine-tune the smaller models first.
Then run the following command:
```bash
export method=cls_transformer
export field=title
export dataset=robust04
export num_segment=16
export max_num_train_instance_perquery=1000
export rerank_threshold=100
export learning_rate=3e-6
export epoch=3

student_BERT_config=gs://Your_gs_bucket/checkpoint/uncased_L-4_H-512_A-8_small/bert_config.json
teacher_BERT_config=gs://cloud-tpu-checkpoints/bert/uncased_L-12_H-768_A-12/bert_config.json
student_BERT_ckpt_parent=gs://Your_gs_bucket/checkpoint/bert_small_onRobust04
teacher_BERT_ckpt_parent=gs://Your_gs_bucket/adhoc/experiment/robust04/title/num-segment-16/bertbase_onMSMARCO_cls_transformer

#MSE CE 
kd_method=MSE
kd_lambda=0.75
runid=base_to_small_kd-${kd_method}_lambda-${kd_lambda}_lr-${learning_rate}

for fold in {1..5}
do
  export data_dir="gs://Your_gs_bucket/adhoc/training.data/$dataset/$field/num-segment-${num_segment}/fold-$fold-train-$max_num_train_instance_perquery-test-$rerank_threshold"
  export output_dir="gs://Your_gs_bucket/adhoc/experiment/$dataset/$field/num-segment-${num_segment}/KD/$runid/fold-$fold"
  export teacher_BERT_ckpt=${student_BERT_parent}/fold-${fold}/model.ckpt-18000
  export student_BERT_ckpt=${student_BERT_parent}/fold-${fold}/model.ckpt-18000

  python3 -u run_knowledge_distill.py \
    --pretrained_model=bert \
    --kd_method=${kd_method} \
    --kd_lambda=${kd_lambda} \
    --teacher_bert_config_file=${teacher_BERT_config} \
    --student_bert_config_file=${student_BERT_config} \
    --teacher_init_checkpoint=${teacher_BERT_ckpt}\
    --student_init_checkpoint=${student_BERT_ckpt}\
    --do_train=True \
    --do_eval=True \
    --train_batch_size=32 \
    --eval_batch_size=32 \
    --learning_rate=$learning_rate \
    --num_train_epochs=$epoch \
    --warmup_proportion=0.1 \
    --aggregation_method=$method \
    --dataset=$dataset \
    --fold=$fold \
    --trec_run_filename=/data2/robust04/runs/title/run.robust04.title.bm25.txt \
    --qrels_filename=/data/anserini/src/main/resources/topics-and-qrels/qrels.robust04.txt \
    --data_dir=$data_dir \
    --output_dir=$output_dir \
    --max_seq_length=256 \
    --max_num_segments_perdoc=${num_segment} \
    --max_num_train_instance_perquery=$max_num_train_instance_perquery \
    --rerank_threshold=$rerank_threshold \
    --use_tpu=True \
    --tpu_name=$tpu_name 
done

gs_dir=gs://Your_gs_bucket/adhoc/experiment/$dataset/$field/num-segment-${num_segment}/KD/$runid
local_dir=/data2/$dataset/reruns/$field/num-segment-${num_segment}/KD/$runid
qrels_path=/data/anserini/src/main/resources/topics-and-qrels/qrels.${dataset}.txt

./scripts/download_and_evaluate.sh  ${gs_dir} ${local_dir} ${qrels_path} ${epoch} 
```
The script also lies in `scripts/run.kd.sh`.
It outputs the following results with regard to PARADE using the BERT-small model (4 layers)!
```bash
P_20                    all     0.4365
ndcg_cut_20             all     0.5098
```

### <a name="resource"></a> Useful Resources

- Fine-tuned models on the MSMARCO passage ranking dataset:

| Model        | L / H    | MRR on MSMARCO DEV | Path |
|--------------|----------|--------------------|------|
| ELECTRA-Base | 12 / 768 | 0.3698     | [Download](https://zenodo.org/record/3974431/files/vanilla_electra_base_on_MSMARCO.tar.gz)    |
| BERT-Base    | 12 / 768 | 0.3637     | [Download](https://zenodo.org/record/3974431/files/vanilla_bert_base_on_MSMARCO.tar.gz)    |
| \            | 10 / 768 | 0.3622     | [Download](https://zenodo.org/record/3974431/files/vanilla_bert_medium_10_base_on_MSMARCO.tar.gz)    |
| \            | 8 / 768  | 0.3560     | [Download](https://zenodo.org/record/3974431/files/vanilla_bert_medium_8_base_on_MSMARCO.tar.gz)   |
| BERT-Medium  | 8 / 512  | 0.3520     | [Download](https://zenodo.org/record/3974431/files/vanilla_bert_medium_on_MSMARCO.tar.gz)    |
| BERT-Small   | 4 / 512  | 0.3427     | [Download](https://zenodo.org/record/3974431/files/vanilla_bert_mini_on_MSMARCO.tar.gz)   |
| BERT-Mini    | 4 / 256  | 0.3247     | [Download](https://zenodo.org/record/3974431/files/vanilla_bert_mini_on_MSMARCO.tar.gz)  |
| \            | 2 / 512  | 0.3160     | [Download](https://zenodo.org/record/3974431/files/vanilla_bert_tiny_small_on_MSMARCO.tar.gz)    |
| BERT-Tiny    | 2 /128   | 0.2600     | [Download](https://zenodo.org/record/3974431/files/vanilla_bert_tiny_on_MSMARCO.tar.gz)   |

- Our run files on the Robust04 and GOV2 collections: 
[Robust04](https://zenodo.org/record/3974431/files/robust04.PARADE.runs.tar.gz), 
[GOV2](https://zenodo.org/record/3974431/files/gov2.PARADE.runs.tar.gz).

### Acknowledgement
Some snippets of the codes are borrowed from 
[NPRF](https://github.com/ucasir/NPRF),
[Capreolus](https://github.com/capreolus-ir/capreolus),
[dl4marco-bert](https://github.com/nyu-dl/dl4marco-bert),
[SIGIR19-BERT-IR](https://github.com/AdeDZY/SIGIR19-BERT-IR).
