# Roseobacter_pseudogene

# Preparation
1. It's strongly recommended to install ruby https://www.ruby-lang.org/en/documentation/installation/ before using the scripts. The reason is because I have prepared the scripts written in Ruby for some simple data processing and related stuff. Of cours, it is possible to write your own scripts instead of using my Ruby scripts.

2. After installation of Ruby, do the following in bash (**you need to go into this folder first**).
```
export RUBYLIB=$RUBYLIB:$PWD/scripts/prepare_scripts/lib
gem install bio parallel
```

Make sure that the installation of these packages is successfull.
```
ruby scripts/checkRubyPackages.rb
```

3. **[Optional]** If you want to do filtering (i.e., steps 3-6), make sure that bedtools (https://bedtools.readthedocs.io/en/latest/) is successfully installed and its path is added to the environment variable *PATH*. Otherwise, if you only want to perform the original Psi-Phi, this is not necessary.


# Step 0: files
1. genomic annotation; Have your genbank files named as "taxon.gbk"
2. genome sequence: Have your genome files named as "taxon.fas". In case there are more than one contig, remember to merge all contigs into a single (pseudo)chromosome (the script Psi-Phi modified by Siyao will remove the pseudogenes identifed by the program whose locations happen to span two contigs in later steps). Also, name the "pseudochromosome" generated by the above step the same as the file name. For example, if the name of the file is "roseobacter123.fas", then the name of the "pseudochromosome" should be "roseobacter123" and in the format of a fasta file it should be like ">roseobacter123".
3. protein sequences: Have the protein files named as "taxon.faa". Also, the name of each protein sequence should be named as "taxon|protein_id" where the taxon is the name of the file. For example, if the original name of the protein sequence is "protein123" and it is from the species "Bradyrhizobium_sp_123", the name of this sequence should be named as "Bradyrhizobium_sp_123|protein123" (don't forget the ">" in front of the sequence name in the fasta-formatted file).


# Step 1: all-against-all tblastn
Let's first prepare the database for tblastn:
```
for i in data/*fas; do b=`basename $i`; c=${b%.fas}; makeblastdb -in $i -dbtype nucl -parse_seqids -out ./data/$c; done
```

Enter the folder *data* and run tblastn in batch.
```
cd data
ruby scripts/do_blast_in_batch.rb --seq_indir . --db_indir . --cpu 10 --nt 4
```
The name of the BLAST output should look like "roseobacter.vs.bradyrhizobium.blast", if the query is roseobacteria.faa and the subject is bradyrhizobium.fas.


# Step 2: Run Psi-Phi
Make sure that you are in the folder "data". If not, type ```cd data```.

Run Psi_Phi module1
```perl ../scripts/scripts_for_Psi_Phi/batch.module1.pl```
Note that there might be some warning messages but it's fine to ignore them based on my experience. I'm unsure what it means as the researcher who modifies the corresponding script has graduated (Siyao Li).

Run Psi_Phi module2
```perl ../scripts/scripts_for_Psi_Phi/batch.module2.pl```
Be sure to make all input files in the same folder (or you could edit the original scripts)
The output should be like "roseobacter_use_bradyrhizobium" where you can find the info for each pseudogene identified by PsiPhi


# Step 3: Summarize the result of Psi-Phi
Go back to the parent folder of data
```cd ..```

Summarize the result of Psi-Phi
```
ruby scripts/prepare_scripts/from_pseudo_to_bed.rb --indir data/ --outdir filtering/prepare --force
```
Note that pseudogenes that overlap with at least 1 bp will be merged into a "larger" pseudogene in the file *filtering/prepare/all_pseudo.txt* which we will be using in subsequent filtering steps. You can change the distance cutoff of merging the pseudogenes identified by Psi-Phi by using *-d*. This is done by *bedtools count* (see https://bedtools.readthedocs.io/en/latest/content/tools/cluster.html for more details). In case you would like to merge only those with overlaps of >= 10 bp, you can set this parameter as *-10*.


# Step 4: Generate tblastn results
ruby scripts/process_scripts/check_pseudogene_aln.rb -i filtering/prepare/all_pseudo.txt -g data -p data --cpu 12 --outdir filtering/tblastn_result --force


# Step 5: Further filtering
```
mkdir single_query

ruby -F"\t" -alne 'next if $F[0]!~/\w/; a=$F[4].split(", "); n=a.size; n ==1 and puts a[0]' filtering/prepare/all_pseudo.txt | sort|uniq -c | awk '{print $2"\t"$1}'|sort -nk 2 > filtering/single_query/ori.list

for i in 5 10 20 30; do awk -v i=$i '{if($2>=i){print $1}}' single_query/ori.list > single_query/$i.list; done
```


# Step 6: Final step
```
ruby scripts/process_scripts/find_orf.rb --check_indir filtering/tblastn_result/ -g data/ -p data/ --cpu 16 -i fildering/prepare/all_pseudo.txt --problematic_query_list filtering/single_query/5.list --outdir filtering/filter_5 --force
```


# Step 7: analyze results
The final result will be output in the file "filter_XX/all_pseudo.filtered.txt".
Important: for filter_XX, XX denotes a way of filtering: if a single orf has identified two many "pseudogenes", then we think it is more likely that this orf is wrongly annotated to be too long, much longer than its actual length. So we decide to use an arbitrary cutoff: if a orf identifies say >= 10 "pseudogenes", then we don't consider any of them as pseudogenes. Because the cutoff is set arbitrary, it is suggested to use different cutoffs (5, 10, 20, etc.) and to compare the results.
