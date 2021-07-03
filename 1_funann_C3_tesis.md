# General *Laccaria* and Psathyrellaceae genome annotation

Rodolfo Ángeles, 2020-2021

General annotation of the four *L. trichodermophora* genome assemblies plus 3 public available *Laccaria* genome and other seven Psathyrellaceae genomes.

All scripts were run from the `~/[...]/C3/bin/` directory.

**Work flow:**

0. Data and tools

1. Clean
2. Predict
3. Annotate
4. Compare



**Run:**

Some configurations and activate the conda funannotate environment

```shell
export LC_ALL=en_US.UTF-8
source activate /home/rangeles/.conda/envs/funannotate
```

```shell
cd C3/bin
```

## 1. Cleaning

```shell
nohup sh 1.1_funann.clean.sh > 1.1_nohup.clean.Psathy.out &
```

script `1.1_funann.clean.sh`:

```sh
#!bin/sh
#cleaning and formating Laccaria genomes to funannotate
#Rodolfo Angeles, nov 2020

#make `list2funannotate.txt` with prefix

## 1 Cleaning assemblies
for smpl in $(cat list2funannotate.txt); do
    # 1.1 clear repeated contigs
    funannotate clean -i ../data/$smpl.fn -o ../out/$smpl.clean.fn
    # 1.2 sort and relabel headers
    funannotate sort -i ../out/$smpl.clean.fn -o ../out/$smpl.clean.sort.fn
    # 1.3 softmasking with tantan
    funannotate mask --cpus 20 -i ../out/$smpl.clean.sort.fn -o ../out/$smpl.clean.sort.mask.fn
    
    #delete tmp files
    rm $smpl.clean.fn $smpl.clean.sort.fn
done

#END
```

poner todos los outs del nohup

## 2. Predictions

```shell
nohup sh 1.2_funann.pred.sh > 1.2_nohup.pred.Psathy.out &
```

Script `1.2_funann.pred.sh`:

```sh
#!bin/sh
#Predicting Fusarium proteomes with funannotate
#by train/run Augustus, snap, GlimmerHMM
#Training steps use the L. bicolor S238N-H82 proteome
#Rodolfo Angeles, nov 2020
# 2.1 prediction step
for smpl in $(cat list2funannotate.txt); do
    funannotate predict \
    -i ../out/$smpl.clean.sort.mask.fn \
    -o ../out/$smpl \
    -s $smpl \
    --isolate $smpl \
    --name $smpl \
    --ploidy 1 \
    --protein_evidence ../data/Proteomes/S238N-H82.Lb.protein.faa \
    --cpus 16
done
#End
```



## 3. Annotation

Run phobius and antismash in remote web servers

```shell
nohup sh 1.3_funann_remote.sh > 1.3_nohup_remote.out &
```

Script `1.3_funann_remote.sh`:

```shell
#!bin/sh
#run antysmash and phobius in remote servers
#Rodolfo Angeles, january 2021

#run
for smpl in $(cat list2funannotate.txt); do
	funannotate remote \
	-m all \
	-e rodolfo.angeles.argaiz@gmail.com \
	-i ../out/$smpl \
	--force
done	
#End
```

Download/update databases

```sh
funannotate setup -b all
```

Run InterProScan and funannotate

```shell
nohup sh 1.3_funann.annot.sh > 3_nohup.annot.Psathyrellaceae.out &
```

script `1.3_funann.annot.sh`:

```sh
#!bin/sh
#annotate Laccaria genomes with funannotate
#Rodolfo Angeles, nov 2020

#Steps
# 1 interpro
# 2 funannot

# 1 interpro
mkdir ../out/interproscan
for smpl in $(cat list2funannotate.txt); do
    interproscan.sh -i ../out/$smpl/predict_results/*.proteins.fa \
    -f xml -iprlookup -dp \
    -d ../out/interproscan
done

# 2 annotate with funannotate prediction and interproscan xml
for smpl in $(cat list2funannotate.txt); do
    funannotate annotate -i ../out/$smpl \
    --iprscan ../out/interproscan/$smpl*.xml \
    --busco_db basidiomycota \
    --cpus 20
done
#End
```



## 4. Comparison

```shell
nohup sh 1.4_funann_compa.sh > 1.4_nohup.compa.out &
```

script `1.4_funann_compa.sh`:

```sh
#!bin/sh
#compare Laccaria (7)(+ 7 Psathyrelaceae) genomes with funannotate
#Rodolfo Angeles, jan 2021

#add salple prefix in the .gbk
for smpl in $(cat list2funannotate.txt); do
        cd ../out/$smpl/annotate_results
        sed -i 's/FUN_/$smpl/g' *.gbk
        cd ../..
done

#funannotate compare
#caution!
#this script will compare all annotate_results
#placed in the ../out/ dir
funannotate compare \
-i ../out/*/annotate_results/*.gbk \
-o Laccarias_fun.compare.v4 --run_dnds estimate \
--cpus 8

mv Laccarias_fun.compare.v4 ../res/
mv Laccarias_fun.compare.v4.tar.gz ../res/
#END
```



## Additional steps

#### get BUSCO results

```shell
grep "C:" */annotate_misc/run_busco/short_summary_busco.txt
```

Get the % of the annotated proteome

Put all  `*.txt` together 

```shell
cat C._angulatus_CBS_144469_C._angulatus_CBS_144469.annotations.txt | awk 'BEGIN {FS="\t"} $2=="CDS" {print}' | awk 'BEGIN {FS="\t"} $7=="" {print}' | awk 'BEGIN {FS="\t"} $9=="" {print}' | awk 'BEGIN {FS="\t"} $10=="" {print}' | awk 'BEGIN {FS="\t"} $11=="" {print}' | awk 'BEGIN {FS="\t"} $14=="" {print}' | awk 'BEGIN {FS="\t"} $15=="" {print}' | awk 'BEGIN {FS="\t"} $16=="" {print}' | awk 'BEGIN {FS="\t"} $16=="" {print}' | awk 'BEGIN {FS="\t"} $17=="" {print}' | awk 'BEGIN {FS="\t"} $18=="" {print}' | awk 'BEGIN {FS="\t"} $19=="" {print}' | wc -l
```

```shell
#!bin/bash
#sacar el numero de proteinas sin ninguna anotacion
#rodolfo Angeles, enero 2020

for smpl in $(cat list.txt); do
	cat $smpl | \
	awk 'BEGIN {FS="\t"} $2=="CDS" {print}' | \
	awk 'BEGIN {FS="\t"} $7=="" {print}' | \
	awk 'BEGIN {FS="\t"} $9=="" {print}' | \
	awk 'BEGIN {FS="\t"} $10=="" {print}' | \
	awk 'BEGIN {FS="\t"} $11=="" {print}' | \
	awk 'BEGIN {FS="\t"} $14=="" {print}' | \
	awk 'BEGIN {FS="\t"} $15=="" {print}' | \
	awk 'BEGIN {FS="\t"} $16=="" {print}' | \
	awk 'BEGIN {FS="\t"} $16=="" {print}' | \
	awk 'BEGIN {FS="\t"} $17=="" {print}' | \
	awk 'BEGIN {FS="\t"} $18=="" {print}' | \
	awk 'BEGIN {FS="\t"} $19=="" {print}' | \
	wc -l >> counts.txt
	done
paste list.txt counts.txt > no.anotado.txt
cat no.anotado.txt
#End
```



#### Create annotation tar ball to share

```sh
#Create directory
cd /space21/PGE/rangeles/C3/res/Laccarias_fun.compare.v4/annotations/
mkdir more_annotation_files/
cd more_annotation_files/

#copy all principal annotation resulting files
for i in C._angulatus C._cinerea C._marcescibilis C._micaceus C._strossmayeri L._amethystina L._bicolor L._trichodermophora P._aberdarensis; do
	cp -r ../../../../out/$i*/annotate_results/ .
done

# move and delete non-necesary dir
mv annotate_results/* .
rm -r annotate_results/

#tar and compress
tar -czvf Laccarias_fun.compare.v4.tar.gz Laccarias_fun.compare.v4/

#generate md5sum
md5sum Laccarias_fun.compare.v4.tar.gz > Laccarias_fun.compare.v4.tar.gz.md5sum
```

