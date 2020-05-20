# OpenNIR (experimaestro version)

OpenNIR-xpm is an end-to-end neural ad-hoc ranking pipeline.

This is an adaptation of [OpenNIR](https://github.com/Georgetown-IR-Lab/OpenNIR) using experiment manager tools (experimaestro and datamaestro).


## Quick start

This is an example for training 

```python
import click
from pathlib import Path
import os
from datamaestro import prepare_dataset
import logging
import multiprocessing


logging.basicConfig(level=logging.INFO)
CPU_COUNT = multiprocessing.cpu_count()


from experimaestro import experiment
from experimaestro_ir.evaluation import TrecEval
from experimaestro_ir.models import BM25
from experimaestro_ir.anserini import IndexCollection, SearchCollection

from onir.rankers.drmm import Drmm
from onir.trainers import PointwiseTrainer
from onir.datasets.robust import RobustDataset
from onir.pipelines import Learner
from onir.random import Random
from onir.vocab import WordvecUnkVocab
from onir.predictors import Reranker

# --- Defines the experiment


@click.option("--small", is_flag=True, help="Reduce the number of iterations (testing)")
@click.option("--debug", is_flag=True, help="Print debug information")
@click.option("--port", type=int, default=12345, help="Port for monitoring")
@click.argument("workdir", type=Path)
@click.command()
def cli(port, workdir, debug, small):
    """Runs an experiment"""
    logging.getLogger().setLevel(logging.DEBUG if debug else logging.INFO)

    # Sets the working directory and the name of the xp
    with experiment(workdir, "drmm", port=port) as xp:
        xp.setenv("JAVA_HOME", os.environ["JAVA_HOME"])
        random = Random()
        
        # Prepare the collection
        wordembs = prepare_dataset("edu.stanford.glove.6b.50")        
        vocab = WordvecUnkVocab(data=wordembs, random=random)
        robust = RobustDataset.prepare().submit()

        # Train with OpenNIR DRMM model
        ranker = Drmm(vocab=vocab).tag("ranker", "drmm")
        predictor = Reranker()
        trainer = PointwiseTrainer()
        learner = Learner(trainer=trainer, random=random, ranker=ranker, valid_pred=predictor, dataset=robust)
        learner.submit()


if __name__ == "__main__":
    cli()
```

## Features

The features below are from [OpenNIR](http://github.com/)
### Rankers

Available in the `onir.rankers` module

 - DRMM `onir.ranker.drmm.Drmm` [paper](https://arxiv.org/abs/1711.08611)
 - (*planned*) Duet (local model) [paper](https://arxiv.org/abs/1610.08136)
 - (*planned*) MatchPyramid  [paper](https://arxiv.org/abs/1606.04648)
 - (*planned*) KNRM [paper](https://arxiv.org/abs/1706.06613)
 - (*planned*) PACRR  [paper](https://arxiv.org/abs/1704.03940)
 - (*planned*) ConvKNRM  [paper](https://www.semanticscholar.org/paper/432b36c1bec275c2778c66f9897f9e02f7d8b579)
 - (*planned*) Vanilla BERT `config/vanilla_bert` [paper](https://arxiv.org/abs/1810.04805)
 - (*planned*) CEDR models `config/cedr/[model]` [paper](https://arxiv.org/abs/1810.04805)
 - (*planned*) MatchZoo models [source](https://github.com/NTMC-Community/MatchZoo)
   - (*planned*) MatchZoo's KNRM 
   - (*planned*) MatchZoo's ConvKNRM 

### Datasets

 - TREC Robust 2004
 - MS-MARCO `config/msmarco`
 - ANTIQUE `config/antique`
 - TREC CAR `config/car`
 - New York Times `config/nyt` -- for [content-based weak supervision](https://arxiv.org/abs/1707.00189)
 - TREC Arabic, Mandarin, and Spanish `config/multiling/*` -- for [zero-shot multilingual transfer learning](https://arxiv.org/pdf/1912.13080.pdf) ([instructions](https://opennir.net/multilingual.html))

### Evaluation Metrics

 - `map` (from trec_eval)
 - `ndcg` (from trec_eval)
 - `ndcg@X` (from trec_eval, gdeval)
 - `p@X` (from trec_eval)
 - `err@X` (from gdeval)
 - `mrr` (from trec_eval)
 - `rprec` (from trec_eval)
 - `judged@X` (implemented in python)

### Vocabularies

 - (**planned**) Binary term matching `vocab=binary` (i.e., changes interaction matrix from cosine similarity to to binary indicators)
 - Pretrained word vectors. Find the list with `datamaestro search tag:"word embeddings"`

## Citing OpenNIR

If you use OpenNIR, please cite the real OpenNIR WSDM demonstration paper and
look at acknowledgements of the original [OpenNIR](https://github.com/Georgetown-IR-Lab/OpenNIR).

```
@InProceedings{macavaney:wsdm2020-onir,
  author = {MacAvaney, Sean},
  title = {{OpenNIR}: A Complete Neural Ad-Hoc Ranking Pipeline},
  booktitle = {{WSDM} 2020},
  year = {2020}
}
```