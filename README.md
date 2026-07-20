# .github

# Immunology-DataNova-2026

three projects testing whether ml can detect a health condition from something cheap to collect: a voice recording, a blood gene expression profile, an antibody sequence.

i wasn't trying to get the highest number. i was trying to find out which numbers actually hold up once you stop cheating yourself with bad evaluation. so everything here is split by subject, tested against a null or an external group where i had one, and the failures are still in the repos.

## the three projects

| repo | question | best honest result |
|---|---|---|
| [voice](https://github.com/Immunology-DataNova-2026/Can-AI-detect-health-conditions-from-someones-voice) | can your voice reveal disease? | **auc 0.866** (voice disorders) |
| [gene expression](https://github.com/Immunology-DataNova-2026/Can-AI-classify-immune-aging-immunosenescence--from-gene-expression) | can blood gene expression tell immune age? | **auc 0.932** internal, **0.811** external |
| [antibody](https://github.com/Immunology-DataNova-2026/Can-a-model-predict-which-antibody-sequences-are-likely-to-bind-SARS-CoV-2-variants) | which antibodies bind which sars-cov-2 variants? | **auroc 0.907** on unseen antigens |

## voice

| task | accuracy | auc | level |
|---|---|---|---|
| voice disorders (saarbrücken, 1,119 speakers) | 0.804 | **0.866** | per-speaker |
| parkinson's (sakar, 252 speakers) | 0.821 | **0.850** | per-speaker |
| parkinson's (oxford/uci, 31 speakers) | 0.72 | 0.71 | per-record |
| voice disorders (voiced, 208 speakers) | 0.63 | 0.65 | per-record |
| acute respiratory illness (3 corpora) | 0.53 | **0.54** | cross-dataset |

the split is permanent vs temporary. stuff that physically changes your vocal cords or motor control (parkinson's, laryngeal disorders) shows up fine. a cold doesn't. that one looked like 0.84 in-distribution and dropped to 0.54 on other datasets, and i proved why: a model trained on zero audio, just age/sex/recording date, matched it. it had learned which batch a clip came from, not whether anyone was sick.

also worth knowing, the two working models fail in opposite directions. voice disorders barely flags healthy people but misses a lot of sick ones (sens 0.627, spec 0.924). parkinson's is the reverse (sens 0.915, spec 0.547).

## gene expression

internal 5-fold cv auc **0.932** (95% ci [0.856, 0.986]), accuracy 0.882 vs a 0.618 baseline. on **23 people from separate geo series it never saw**, auc **0.811** (ci [0.592, 0.962]).

three checks that it's real:
- 100 random 25-gene panels average 0.587 auc. mine beats 97 of 100 (p = 0.030), so the gene selection is doing actual work
- refitting the feature filter inside each fold instead of once on everything costs 0.048 auc, meaning i used the correct version
- the panel independently picks up **klrb1 (cd161)**, a known immunosenescence marker, declining with age. **lrrn3** is the single strongest gene

honest caveat: that external group is 23 people, so the ci runs 0.59 to 0.96. direction is convincing, precision isn't.

## antibody

trained on avida-sars-cov-2 (49,685 antibody-antigen pairs, 17 antigens), dual esm-2 encoders with cross-attention.

| evaluation | auroc | auprc |
|---|---|---|
| held-out antigens (3 never seen) | **0.9065** | 0.8813 |
| random split of the same data | 0.9419 | 0.9282 |
| full test set (pooled) | 0.7739 | 0.7051 |

first two rows are the point. a random split makes it look **+0.0353 better** because the model gets to see other pairs from the same antigen. holding whole antigens out is the real test, and 0.907 on antigens it's never encountered is legit.

## the thing tying all three together

i measured how much a common evaluation shortcut inflates each project's own result. same data, same model, only the protocol changed:

| project | shortcut | honest | shortcut | inflation |
|---|---|---|---|---|
| voice (parkinson's, uci) | split by recording not patient | 0.71 | 0.95 | **+0.24 auc** |
| gene expression | fit filter once not per fold | 0.932 | 0.980 | **+0.048 auc** |
| antibody | random split not antigen holdout | 0.907 | 0.942 | **+0.035 auroc** |

leakage isn't equally bad everywhere. it's brutal when the thing leaking is the identity you're trying to generalize across (a patient with 6 recordings), and pretty mild when the model actually learned something transferable. reporting all three including the small ones is more honest than just showing off the 0.24.

the short version of what i'd tell anyone starting one of these:
1. split by person, never by sample
2. test on data someone else collected
3. don't use features that secretly contain the answer (clinical questionnaire scores will happily inflate your result and teach you nothing)

## running it

each repo has a pipeline script, a predict script, the trained model, and a model card with full-precision metrics, confusion matrices, and bootstrap cis.

```bash
pip install -r requirements.txt
python pipeline.py            # gene expression
python <task>/pipeline.py     # voice, one experiment
python train.py               # antibody
python predict.py <input>     # score something new
```

controls are in `controls_voice.py`, `controls_immuno.py`, `controls_antibody.py`. seeds are fixed (42 for voice and gene, 0 for antibody) so re-runs reproduce the committed numbers.

all datasets are public (saarbrücken, sakar + oxford parkinson's, voiced, coswara/coughvid/sound-dr, geo gse123696-98, avida-sars-cov-2). raw data isn't vendored, cc by attributions are in `THIRD_PARTY_LICENSES/`.

## limitations

- voice and gene results are within-corpus. subject-independent, but one dataset each. only the respiratory one got a real cross-dataset test, and it failed
- the gene external group is 23 people
- the antibody ablation is one seed with three specific held-out antigens, so a different draw would move the number
- none of this is clinically validated. public benchmarks, not diagnostic tools
