# Knowledge Discovery using Recommendation Systems

All this code WILL BE in a master script that can run the entire analysis.

## Install Dependencies

There are a few dependencies to install first. Run the following scripts and make sure they all succeeded.

```bash
cd dependencies
bash install.geniatagger.sh
bash install.powergraph.sh
bash install.lingpipe.sh
bash install.tclap.sh
cd ../
```

## Setup Lingpipe

## Compile ANNI vector generation code

Most of the analysis code is in Python and doesn't require compilation. Only the code to generate ANNI concept vectors requires compilation as it is written in C++.

```bash
cd anniVectors
make
cd ../
```

## Working Directory

We're going to do all analysis inside a working directory and call the various scripts from within it. So in the root of this repo we do the following.

```bash
mkdir workingDir
cd workingDir
```

## Download PubMed and PubMed Central

We need to download the abstracts from PubMed and full text articles from the PubMed Central Open Access subset. This is all managed by the prepareMedlineANDPMCData.sh script.

```bash
bash ../data/prepareMedlineAndPMCData.sh medlineAndPMC
```

## Install UMLS

This involves downloading UMLS from https://www.nlm.nih.gov/research/umls/licensedcontent/umlsknowledgesources.html and running MetamorphoSys. Install the Active dataset. Unfortunately this can't currently be done via the command line.

## Create the UMLS-based word-list

A script will pull out the necessary terms, their IDs, semantic types and synonyms from the UMLS RRF files.

```bash
bash ../data/generateUMLSWordlist.sh /projects/bioracle/ncbiData/umls/2016AB/META/ ./
```

TEMPORARY: We'll do a little simplification for the testing process. Basically we going to build a mini word-list

```bash
mv umlsWordlist.WithIDs.txt fullWordlist.WithIDs.txt
rm umlsWordlist.Final.txt

echo -e "cancer\ninib\nanib" > simpler_terms.txt
grep -f simpler_terms.txt fullWordlist.WithIDs.txt > umlsWordlist.WithIDs.txt

# Make sure Alzheimer's and Parkinson's terms are included
grep "C0002395" fullWordlist.WithIDs.txt >> umlsWordlist.WithIDs.txt
grep "C0030567" fullWordlist.WithIDs.txt >> umlsWordlist.WithIDs.txt
grep "C0030567" fullWordlist.WithIDs.txt >> umlsWordlist.WithIDs.txt

sort -u umlsWordlist.WithIDs.txt > umlsWordlist.WithIDs.txt.unique
mv umlsWordlist.WithIDs.txt.unique umlsWordlist.WithIDs.txt

cut -f 3 -d $'\t' umlsWordlist.WithIDs.txt > umlsWordlist.Final.txt
```

We then need to process this wordlist into a Python pickled file and remove stop-words and short words.

```bash
python ../text_extraction/cooccurrenceMajigger.py --termsWithSynonymsFile umlsWordlist.Final.txt --stopwordsFile ../data/selected_stopwords.txt --removeShortwords --binaryTermsFile_out umlsWordlist.Final.pickle
```

## Run text mining across all PubMed and PubMed Central

First we generate a list of commands to parse all the text files. This is for use on a cluster.

```bash
bash ../text_extraction/generateCommandLists.sh medlineAndPMC
```

Next we need to run the text mining tool on a cluster. This may need editing for your partocular environment.

```bash
bash ../text_extraction/runCommandsOnCluster.sh commands_all.txt
```

## Combine data into a dataset for analysis

We first need to extract the cooccurrences, occurrences and sentence counts from the mined data (as they're all combined in the same output files)

```bash
bash ../combine_data/splitDataTypes.sh mined mined_and_separated
```

Next up we merged these various cooccurrence, occurrence and sentence count files down into the final data set

```bash
bash ../combine_data/produceDataset.sh mined_and_separated/cooccurrences mined_and_separated/occurrences mined_and_separated/sentencecount 2010 finalDataset
```

## Generate ANNI Vectors

ANNI requires creating concept vectors for all concepts

```bash
../anniVectors/generateAnniVectors --cooccurrenceData finalDataset/training.cooccurrences --occurrenceData finalDataset/training.occurrences --sentenceCount `cat finalDataset/training.sentenceCounts` --vectorsToCalculate finalDataset/training.ids --outIndexFile anni.training.index --outVectorFile anni.training.vectors

../anniVectors/generateAnniVectors --cooccurrenceData finalDataset/trainingAndValidation.cooccurrences --occurrenceData finalDataset/trainingAndValidation.occurrences --sentenceCount `cat finalDataset/trainingAndValidation.sentenceCounts` --vectorsToCalculate finalDataset/trainingAndValidation.ids --outIndexFile anni.trainingAndValidation.index --outVectorFile anni.trainingAndValidation.vectors
```

## Generate negative data for comparison

Next we'll create negative data to allow comparison of the different ranking methods.

```bash
negativeCount=1000000
python ../analysis/generateNegativeData.py --trueData <(cat finalDataset/training.cooccurrences finalDataset/validation.cooccurrences) --knownConceptIDs finalDataset/training.ids --num $negativeCount --outFile negativeData.validation

python ../analysis/generateNegativeData.py --trueData <(cat finalDataset/trainingAndValidation.cooccurrences finalDataset/testing.all.cooccurrences) --knownConceptIDs finalDataset/trainingAndValidation.ids --num $negativeCount --outFile negativeData.testing

```

## Run Singular Value Decomposition

We'll run a singular value decomposition on the co-occurrence data.

```bash
bash ../analysis/runSVD.sh --dimension `cat umlsWordlist.Final.txt | wc -l` --svNum 500 --matrix finalDataset/training.cooccurrences --outU svd.training.U --outV svd.training.V --outSV svd.training.SV --mirror --binarize

bash ../analysis/runSVD.sh --dimension `cat umlsWordlist.Final.txt | wc -l` --svNum 500 --matrix finalDataset/trainingAndValidation.cooccurrences --outU svd.trainingAndValidation.U --outV svd.trainingAndValidation.V --outSV svd.trainingAndValidation.SV --mirror --binarize
```

We first need to calculate the class balance for the validation set

```bash
validation_termCount=`cat finalDataset/training.ids | wc -l`
validation_knownCount=`cat finalDataset/training.cooccurrences | wc -l`
validation_testCount=`cat finalDataset/validation.cooccurrences | wc -l`
validation_classBalance=`echo "$validation_testCount / (($validation_termCount*($validation_termCount+1)/2) - $validation_knownCount)" | bc -l`
```

Now we need to test a range of singular values to find the optimal value

```bash
minSV=5
maxSV=500
numThreads=16

mkdir svd.crossvalidation
seq $minSV $maxSV | xargs -I NSV -P $numThreads python ../analysis/calcSVDScores.py --svdU svd.training.U --svdV svd.training.V --svdSV svd.training.SV --relationsToScore finalDataset/validation.subset.1000000.cooccurrences --sv NSV --outFile svd.crossvalidation/positive.NSV

seq $minSV $maxSV | xargs -I NSV -P $numThreads python ../analysis/calcSVDScores.py --svdU svd.training.U --svdV svd.training.V --svdSV svd.training.SV --relationsToScore negativeData.validation --sv NSV --outFile svd.crossvalidation/negative.NSV

seq $minSV $maxSV | xargs -I NSV -P $numThreads bash -c "python ../analysis/evaluate.py --positiveScores <(cut -f 3 svd.crossvalidation/positive.NSV) --negativeScores <(cut -f 3 svd.crossvalidation/negative.NSV) --classBalance $validation_classBalance --analysisName NSV > svd.crossvalidation/results.NSV"

cat svd.crossvalidation/results.* > svd.results
```

Then we calculate the Area under the Precision Recall curve for each # of singular values and find the optimal value

```bash
/gsc/software/linux-x86_64/python-2.7.5/bin/python ../analysis/statsCalculator.py --evaluationFile svd.results > curves.svd
sort -k3,3n curves.svd | tail -n 1 | cut -f 1 > parameters.sv
optimalSV=`cat parameters.sv`
```

We'll also calculate the optimal threshold which is useful later. We won't use a threshold to compare the different methods (as we're using the Area under the Precision Recall curve). But we will want to use a threshold to call positive/negatives later in our analysis. The threshold is calculated as the value that gives the optimal F1-score. So we sort by F1-score and pull out the associated threshold.

```bash
grep -P "^$optimalSV\t" svd.results | sort -k5,5n | cut -f 10 -d $'\t' | tail -n 1 > parameters.threshold
optimalThreshold=`cat parameters.threshold`
```

## Calculate scores for positive & negative relationships

For positive and negative

```bash
python ../analysis/ScoreImplicitRelations.py --cooccurrenceFile finalDataset/trainingAndValidation.cooccurrences --occurrenceFile finalDataset/trainingAndValidation.occurrences --sentenceCount finalDataset/trainingAndValidation.sentenceCounts --relationsToScore finalDataset/testing.all.subset.1000000.cooccurrences --anniVectors anni.trainingAndValidation.vectors --anniVectorsIndex anni.trainingAndValidation.index --outFile scores.testing.positive

python ../analysis/ScoreImplicitRelations.py --cooccurrenceFile finalDataset/trainingAndValidation.cooccurrences --occurrenceFile finalDataset/trainingAndValidation.occurrences --sentenceCount finalDataset/trainingAndValidation.sentenceCounts --relationsToScore negativeData.testing --anniVectors anni.trainingAndValidation.vectors --anniVectorsIndex anni.trainingAndValidation.index --outFile scores.testing.negative
```

For SVD

```bash
python ../analysis/calcSVDScores.py --svdU svd.trainingAndValidation.U --svdV svd.trainingAndValidation.V --svdSV svd.trainingAndValidation.SV --relationsToScore finalDataset/testing.all.subset.1000000.cooccurrences --outFile scores.testing.positive.svd --sv $optimalSV
  
python ../analysis/calcSVDScores.py --svdU svd.trainingAndValidation.U --svdV svd.trainingAndValidation.V --svdSV svd.trainingAndValidation.SV --relationsToScore negativeData.testing --outFile scores.testing.negative.svd --sv $optimalSV
```

## Generate precision/recall curves for each method with associated statistics

First we need to calculate the class balance.

```bash
testing_termCount=`cat finalDataset/trainingAndValidation.ids | wc -l`
testing_knownCount=`cat finalDataset/trainingAndValidation.cooccurrences | wc -l`
testing_testCount=`cat finalDataset/testing.all.cooccurrences | wc -l`
testing_classBalance=`echo "$testing_testCount / (($testing_termCount*($testing_termCount+1)/2) - $testing_knownCount)" | bc -l`
```
Then we run evaluate on the separate columns of the score file

```bash
python ../analysis/evaluate.py --positiveScores <(cut -f 3 scores.testing.positive.svd) --negativeScores <(cut -f 3 scores.testing.negative.svd) --classBalance $testing_classBalance --analysisName "SVD_$optimalSV" >> curves.all

python ../analysis/evaluate.py --positiveScores <(cut -f 3 scores.testing.positive) --negativeScores <(cut -f 3 scores.testing.negative) --classBalance $testing_classBalance --analysisName factaPlus >> curves.all

python ../analysis/evaluate.py --positiveScores <(cut -f 4 scores.testing.positive) --negativeScores <(cut -f 4 scores.testing.negative) --classBalance $testing_classBalance --analysisName bitola >> curves.all

python ../analysis/evaluate.py --positiveScores <(cut -f 5 scores.testing.positive) --negativeScores <(cut -f 5 scores.testing.negative) --classBalance $testing_classBalance --analysisName anni >> curves.all

python ../analysis/evaluate.py --positiveScores <(cut -f 6 scores.testing.positive) --negativeScores <(cut -f 6 scores.testing.negative) --classBalance $testing_classBalance --analysisName arrowsmith >> curves.all

python ../analysis/evaluate.py --positiveScores <(cut -f 7 scores.testing.positive) --negativeScores <(cut -f 7 scores.testing.negative) --classBalance $testing_classBalance --analysisName jaccard >> curves.all

python ../analysis/evaluate.py --positiveScores <(cut -f 8 scores.testing.positive) --negativeScores <(cut -f 8 scores.testing.negative) --classBalance $testing_classBalance --analysisName preferentialAttachment  >> curves.all
```

Then we finally calculate the area under the precision recall curve for each method.

```bash
/gsc/software/linux-x86_64/python-2.7.5/bin/python ../analysis/statsCalculator.py --evaluationFile curves.all > curves.stats
```

## Calculate Predictions for Following Years

Insert explanation here

```bash
rm -f yearByYear.results
for testFile in `find finalDataset/ -name 'testing*subset*' | grep -v all | sort`
do
  testYear=`basename $testFile | cut -f 2 -d '.'`

  cooccurrenceCount=`cat $testFile | wc -l`

  python ../analysis/calcSVDScores.py --svdU svd.trainingAndValidation.U --svdV svd.trainingAndValidation.V --svdSV svd.trainingAndValidation.SV --relationsToScore $testFile --sv $optimalSV --threshold $optimalThreshold --outFile tmpPredictions

  predictionCount=`cat tmpPredictions | wc -l`

  echo -e "$testYear\t$cooccurrenceCount\t$predictionCount" >> yearByYear.results
done
```

## Plot Figures

Lastly we plot the figures used in the paper. The first is a comparison of the different methods.

```bash
/gsc/software/linux-x86_64-centos5/R-3.0.2/bin/Rscript ../plots/comparison.R curves.stats comparison.tiff
```

The next is a comparison shown as Precision-Recall curves

```bash
/gsc/software/linux-x86_64-centos5/R-3.0.2/bin/Rscript ../plots/PRcurves.R curves.all PRcurve.tiff
```

The next is violin plots comparing the different methods

The next is an analysis of the years after the split year

```bash
/gsc/software/linux-x86_64-centos5/R-3.0.2/bin/Rscript ../plots/yearByYear.R yearByYear.results yearByYear.tiff
```

## Make Predictions for Alzheimer's and Parkinson's Disease

Lastly we'll output predictions for Alzheimer's and Parkinson's.

```bash
# Store the CUIDs for Alzheimer's and Parkinson's
echo "C0002395" >> cuids.alzheimers.txt
echo "C0030567" >> cuids.parkinsons.txt
echo "C0242422" >> cuids.parkinsons.txt

# Use the CUIDs to find the row numbers for each term
grep -nFf cuids.alzheimers.txt umlsWordlist.WithIDs.txt | cut -f 1 -d ':' | awk ' { print $0-1; } ' > ids.alzheimers.txt
grep -nFf cuids.parkinsons.txt umlsWordlist.WithIDs.txt | cut -f 1 -d ':' | awk ' { print $0-1; } ' > ids.parkinsons.txt

# Get all terms that are of type Pharmacologic Substance (T121) or Clinical Drug (T200)
grep -n -P "(T121)|(T200)" umlsWordlist.WithIDs.txt | cut -f 1 -d ':' | awk ' { print $0-1; } ' > ids.drugs.txt

# Filter for those terms that actually appear in the matrix (using the trainingAndValidation.ids file)
grep -xFf finalDataset/trainingAndValidation.ids ids.drugs.txt > ids.drugs.txt.filtered
mv ids.drugs.txt.filtered ids.drugs.txt

# Start calculating scores
python ../analysis/calcSVDScores.py --svdU svd.trainingAndValidation.U --svdV svd.trainingAndValidation.V --svdSV svd.trainingAndValidation.SV --idsFileA ids.alzheimers.txt --idsFileB ids.drugs.txt --sv $optimalSV --threshold $optimalThreshold --outFile predictions.alzheimers.txt
sort -k3,3gr predictions.alzheimers.txt > predictions.alzheimers.txt.sorted
mv predictions.alzheimers.txt.sorted predictions.alzheimers.txt

python ../analysis/calcSVDScores.py --svdU svd.trainingAndValidation.U --svdV svd.trainingAndValidation.V --svdSV svd.trainingAndValidation.SV --idsFileA ids.parkinsons.txt --idsFileB ids.drugs.txt --sv $optimalSV --threshold $optimalThreshold --outFile predictions.parkinsons.txt
sort -k3,3gr predictions.parkinsons.txt > predictions.parkinsons.txt.sorted
mv predictions.parkinsons.txt.sorted predictions.parkinsons.txt

# Lastly get the actual terms out of the file for the predictions
cat predictions.alzheimers.txt | awk -v f=umlsWordlist.WithIDs.txt ' BEGIN { x=0; while (getline < f) dict[x++] = $0; } { print $0"\t"dict[$2]; } ' > predictions.alzheimers.withterms.txt
cat predictions.parkinsons.txt | awk -v f=umlsWordlist.WithIDs.txt ' BEGIN { x=0; while (getline < f) dict[x++] = $0; } { print $0"\t"dict[$2]; } ' > predictions.parkinsons.withterms.txt

```

