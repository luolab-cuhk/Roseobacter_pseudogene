# Roseobacter_pseudogene

# Preparation
It's strongly recommended to install ruby https://www.ruby-lang.org/en/documentation/installation/ before using the scripts. The reason is because I have prepared the scripts written in Ruby for some simple data processing and related stuff. Of cours, it is possible to write your own scripts instead of using my Ruby scripts.

After installation of Ruby, do the following in bash.
```
export RUBYLIB=$RUBYLIB:$PWD/scripts/prepare_scripts/lib
gem install bio parallel
```

Make sure that the installation of these packages is successfull.
```
ruby scripts/checkRubyPackages.rb
```


# Step 0: files
1. genomic annotation; Have your genbank files named as "taxon.gbk"
2. genome sequence: Have your genome files named as "taxon.fas". In case there are more than one contig, remember to merge all contigs into a single (pseudo)chromosome (the script Psi-Phi modified by Siyao will remove the pseudogenes identifed by the program whose locations happen to span two contigs in later steps). Also, name the "pseudochromosome" generated by the above step the same as the file name. For example, if the name of the file is "roseobacter123.fas", then the name of the "pseudochromosome" should be "roseobacter123" and in the format of a fasta file it should be like ">roseobacter123".
3. protein sequences: Have the protein files named as "taxon.faa". Also, the name of each protein sequence should be named as "taxon|protein_id" where the taxon is the name of the file. For example, if the original name of the protein sequence is "protein123" and it is from the species "Bradyrhizobium_sp_123", the name of this sequence should be named as "Bradyrhizobium_sp_123|protein123" (don't forget the ">" in front of the sequence name in the fasta-formatted file).


# Step 1: all-against-all tblastn
Let's first prepare the database for tblastn:
```
for i in data/*fas; do b=`basename $i`; c=${b%.fas}; makeblastdb -in $i -dbtype nucl -parse_seqids -out ./data/$c; done
```

Enter the folder **data** and run tblastn in batch.
```
cd data #
ruby scripts/do_blast_in_batch.rb --seq_indir . --db_indir . --cpu 10 --nt 4
```
The name of the BLAST output should look like "roseobacter.vs.bradyrhizobium.blast", if the query is roseobacteria.faa and the subject is bradyrhizobium.fas.


# Step 2: run Psi-Phi
Make sure that you are in the folder "data". If not, type ```cd data```.

Run Psi_Phi module1
```perl ../scripts/scripts_for_Psi_Phi/batch.module1.pl```
Note that there might be some warning messages but it's fine to ignore them based on my experience. I'm unsure what it means as the researcher who modifies the corresponding script has graduated (Siyao Li).

Run Psi_Phi module2
```perl ../scripts/scripts_for_Psi_Phi/batch.module2.pl```
Be sure to make all input files in the same folder (or you could edit the original scripts)
The output should be like "roseobacter_use_bradyrhizobium" where you can find the info for each pseudogene identified by PsiPhi
