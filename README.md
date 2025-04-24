# Introduction
Our project is based on the original paper that we not only replicated but also expanded upon. [Learning to execute instructions in a Minecraft dialogue](https://www.aclweb.org/anthology/2020.acl-main.232/).
This project was developed as an extension of the original ACL 2020 paper, and includes additional work and improvements. It was collaboratively developed by Stanley Joel Gona (gona@uni-potsdam.de) and Mackwyn Quadras (quadras@uni-potsdam.de).

More details on this work can be found at [this landing page](http://juliahmr.cs.illinois.edu/Minecraft/).

# Setup
## Dependencies
Install requirements into a virtual environment using Anaconda and the [environment.yaml](environment.yaml) file we provide:
```
conda env create -f environment.yaml
```
This uses Python 3.6.10, PyTorch 1.2.0 and nltk 3.4.5 and assumes installing PyTorch on a GPU machine with CUDA 10. If you want to instead install on a CPU machine, use a lower CUDA or if you want more control on installation, read on. Otherwise, you can move on to [Data](#data).

For an alternate setup, we provide manual installation commands below which achieve the same result as the YAML file:

- To create the conda env:
```
conda create --name <your env name> python=3.6.10
```
- To install packages:
```
conda install pytorch==1.2.0 torchvision==0.4.0 cudatoolkit=10.0 -c pytorch (edit this as per your system specs)
conda install scikit-learn=0.23.1
conda install -c anaconda nltk=3.4.5
conda install -c conda-forge gitpython=3.1.3
conda install -c conda-forge matplotlib=3.2.2
```
Refer [PyTorch](https://pytorch.org/get-started/previous-versions/#conda) for alternate ways of installing PyTorch based on different systems specs.

## Data
- Download the tokenizer. Run the Python interpreter and type the commands:
```
>>> import nltk
>>> nltk.download('punkt')
```
This will create a directory named `nltk_data` within your home directory containing a bunch of tokenizers.


We generated the vocabulary file using the 'GoogleNews-vectors-negative300.bin.gz' dataset and stored it in vocabulary/glove.42B.300d-lower-1r-speaker-oov_as_unk-all_splits. If you prefer to create a custom vocabulary file, you can obtain the embedding files from https://code.google.com/archive/p/word2vec/ and generate your vocabulary file using the following command:
```
!python3 /content/BAP-unipotsdam_lmgs/python/vocab.py --embeddings_file /content/minecraft-bap-models/vocabulary/glove.42B.300d-lower-1r-speaker-oov_as_unk-all_splits/glove.6B.50d.txt --embed_size 300 --oov_as_unk --lower --use_speaker_tokens --use_builder_action_tokens --threshold 0 --verbose
```

# Raw data
In `data`, the [Minecraft Dialogue Corpus](https://drive.google.com/drive/folders/16lDzswcQh8DR2jkQJdoVTK-RyVDFPHKa?usp=sharing) (without screenshots) is in `logs`. The target structures are in `gold-configurations`. `logs/splits.json` defines the train-test-val split by defining subsets of target structures. `lexicons` contains `synonym_substitutions.json`, a file containing word synonyms we use for data augmentation.

All of this raw data is sourced from our previous work [Collaborative Dialogue in Minecraft](https://www.aclweb.org/anthology/P19-1537/).

# Preprocessed data
The original authors preprocess the raw data to obtain train/val/test samples for the Builder Action Prediction (BAP) task. These may be downloaded from https://uofi.box.com/s/pwm7gr71hncqbtyscy9quyd446j19ydx.

There are two files per data split:
- `<split name>-samples.pkl`: A list of samples. The data format for a sample can be found [here](docs/data_format.md).
- `<split name>-jsons.pkl`: A list of game logs. A game log is a json/dict corresponding to a dialog. These are from the original Minecraft Dialogue Corpus. Samples are extracted from every such log.

## Data preprocessing
All of our code resides in [python](python). If you want to re-generate the samples yourself, run data_loader.py as follows:
```
python data_loader.py --lower --dump_dataset --save_dest_dir=<destination directory> --split=<train/val/test>
```

Run `python data_loader.py -h` for more information on the above CLAs amongst others. This reads in the Minecraft Dialogue Corpus and writes samples to `<destination directory>`.

To generate a smaller dataset (possibly for development purposes), simply interrupt the script while running with a `KeyboardInterrupt` (`Ctrl + c`).

# Path Customization
We executed the code in Google Colab by first cloning the GitHub repository using the `git clone` command from this directory: https://github.com/GrimGamer1999/BAP-unipotsdam_lmgs. During execution, we used absolute paths that corresponded to the Colab directory structure.

Before running the code, please customize the paths in the code files to suit your preferences, as this step is essential to prevent potential errors. It's worth noting that if you choose to run the code in Google Colab following the mentioned `git clone` process, it should run without issues.

# Training
Code to train, generate from and evaluate models resides in [python/training_infra](python/training_infra). `trainer.py` is a helper script that can initialize and train a model with different combinations of hyperparameters in succession. It is intended to be used to train multiple models in sequence on a single GPU without need for intervention. It first reads in a specified configuration file that defines the different hyperparameter options to be used, then trains a model for each combination of those hyperparameters. Post training a model, it also runs the model on the val set to produce generated output. Following that, evaluation is also performed to compute the net action F1 metric amongst other stats.

The hyperparameter configuration file lists a hyperparameter, by name, per line, with the values you would like to use as whitespace-separated options following the names. An example hyperparameter configuration file can be found [here](config/hyperparameters-3_0.config). All hyperparameter configuration files should ideally be stored in the `config` directory.

To run the trainer, call it as follows:
```
python trainer.py <path to hyperparameter config file> <other non hyperparameter args ...>
```
We personally attempted to execute the code using the following commands, along with making some adjustments to the trainer file itself:
```
!python3 trainer.py /content/BAP-unipotsdam_lmgs/config/hyperparameters-3.config --num_epochs=40 --rnn="gru" --saved_dataset_dir=/content/BAP-unipotsdam_lmgs/data/lower
```

Note that this command is compatible with Google collab paths and must be changed accordingly when running in the local system

Other non hyperparameter args that can be specified on the command line include args for specifying where the preprocessed data  samples reside, etc. (However, if you choose, you can also specify those in the config file as well.) For a complete list of allowed hyperparameters and other non hyperparameter args, run `python trainer_action_pred_seq.py -h`.

To interrupt training after a few epochs (possibly for development purposes), simply interrupt the script while running with a `KeyboardInterrupt` (`Ctrl + c`). You can interrupt generation similarly to avoid generating on the entire dataset. However, if you want to do this in a more principled manner, we recommend using [development mode](#development-mode) which we describe below.

The script creates and writes to `models/ActionsDecoder/<yyyymmdd>/ActionsDecoder_trainer-<timestamp-ms>` where the timestamps are determined by the start time of the script. Inside it are:
- A log of the trainer script's console messages
- A timestamped sub-directory for each model trained. A sub-directory will contain:
    -  The model is stored as as an encoder module and a decoder module. The files are named `ActionsDecoder-encoder-best_by_loss.pkl` and `ActionsDecoder-decoder-best_by_loss.pkl` respectively. This is the model obtained from early stopping (the model achieving smallest loss on the validation set) and the one we report.
    - Model files are also written after every epoch with the name `ActionsDecoder-<encoder/decoder>-epoch-<epoch number>.pkl`. (By default, a model is saved every epoch. This can be toggled in the script through a CLA.)
    - The model after the very last epoch named `ActionsDecoder-<encoder/decoder>-final.pkl` 
    - `ActionsDecoder_train.log`: A log of console messages during the model's training, etc.
    - `config.txt`: A list of all hyperparameters as well as trainer.py arguments used to train the model.
    - `eval-best.txt`: A small snippet containing details of when the best model (by early stopping) was found, as well as its loss on the validation set.
    - `eval-final.txt`: A small snippet containing details of when the very last model was trained, as well as its loss on the validation set.
    - `generated_sentences*.txt`: Contains metrics like net action F1, generated action sequences for each example accompanied by some of the model input (the previous utterances) and the ground truth sequence. To compile evaluation results in a more structured format, we recommend using the `compile_results.py` script as described [below](#compiling-results).
    - `generated_seqs*.p`: A machine-readable pickle file containing the model inputs, output and ground truth for each val example. This is the output of the generation module.
    - `raw_sentences*.txt` -- Contains generated sequences for each example
    - `ActionsDecoder_train.train_val_losses.png`: A plot of the learning curves on train and val

## Development mode
To run the script on a small amount of data for development purposes, set `--development_mode` as a CLA.

# Generation and evaluation only
If you only want to run generation and evaluation using a trained model, you will need `generate.py`. Run it as follows:
```
python generate.py <path to model directory> --saved_dataset_dir=<path to data> --split=<train/val/test> --init_rngs
```
Run `python generate.py -h` for more information on the above CLAs amongst others. For example, you can try setting the beam size, maximum number of decoding time steps, whether to use constrained decoding or not, etc. It will generate as output all the aformentioned generation related files.

To interrupt generation to avoid generating on the entire dataset (possibly for development purposes), simply interrupt the script while running with a `KeyboardInterrupt` (`Ctrl + c`). However, if you want to do this in a more principled manner, we recommend using [development mode](#development-mode) again.

*NOTE: Generation should ideally be done on a CPU (instead of a GPU) for efficiency reasons.*

# Evaluation only
If you only want to run evaluation only using a trained model and its generated output (the aforementioned pickled `generated_seqs*.p` file), you will again need `generate.py`, but this time with an extra flag to set. Run it as follows:
```
python generate.py <path to model directory> --eval_only --regenerate_sentences
```
It will generate as output the aforementioned `generated_sentences*.txt` file.

# Compiling results
`compile_results.py` is a meta script that compiles results for all models into a single csv file. Run it as follows:
```
python compile_results.py <path to dir containing models> <args ...>
```

Run `python compile_results.py -h` for more information on the CLAs. Use `<args ...>` to set those args that you set during training/generation. For example, if you set `beam_size` to 5 during training/generation, use the same here.

It produces as output `cumulative_evals*.csv`, a CSV file containing one row per model. Columns include hyperparameter values, metrics, etc.

# Vocabularies
Vocabulary files used in the utterances encoder are stored in the `vocabulary` directory. It contains two vocabs: `glove.42B.300d-lower-1r-speaker-builder_actions-oov_as_unk-all_splits/vocab.pkl` and `glove.42B.300d-lower-1r-speaker-oov_as_unk-all_splits/vocab.pkl`. The former is the one to be used when using builder action tokens and the latter when not. Our code handles switching between these two vocabs seamlessly -- so you will not need to.

To learn more on how to generate vocab files, see [this](docs/vocab.md).
