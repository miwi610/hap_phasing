# Haplotye phasing workflow v1.0
Requirement : megalodon v2.5.0
Requirement : Clair3 v0.1.12
Requirement : whatshap v1.4
Requirement : bgzip, tabix
Requirement : bcftools v1.15.1
Requirement : samtools 1.9
Requirement : modbam2bed v0.5.3
Requirement : SNPsplit v0.3.2

Steps 1-4 are to identify SNPs that are not in reference genome (GT:1/1 or 1/2) and replace those positions of reference genome with ALT(1) nucleotides.
These steps are to identify SNPs that always shows GT:0/1.
# Step 1. Basecalling and mapping ONT data with megalodon
Requirement : megalodon v2.5.0
myref=genome.mmi
fast5_pass=fast5
guppy_config=dna_r9.4.1_450bps_sup_prom.cfg
megalodon ${fast5_pass} \
  --guppy-${guppy_config} \
  --remora-modified-bases dna_r9.4.1_e8 sup 0.0.0 5mc CG 0 \
  --outputs basecalls mappings mod_mappings mods \
  --reference ${myref}

# Step 2. Clair3 SNP identification.
Requirement : Clair3 v0.1.12
Requirement : Whatshap v1.4
bam=mapping.bam
myref=genome.fa
MODEL_NAME="r941_prom_sup_g5014"
THREADS=24
run_clair3.sh \
  --bam_fn=${bam} \
  --ref_fn=${myref} \
  --threads=${THREADS} \
  --platform="ont" \
  --model_path="path_to_model_dir/${MODEL_NAME}"

# Step 3. Identifying SNPs with 1/1 or 1/2 mark.
Requirement : bgzip, tabix
gzip -dc ./merge_output.vcf.gz | perl -F'\t' -anle 'if ($=~/^#/) {print $;} elsif ($F[6] eq "PASS" and length($F[3])==1) {if ($F[9] =~/1/1/ and length($F[4])==1) {print $_;} elsif ($F[9]=~/1/2/) {@a=split/,/,$F[4];if (length($a[0])==1 and length($a[1])==1) {$F[9]=~s@1/2@0/1@;print join("\t",@F[0..3],$a[0],@F[5..9]);}}}' > ref_SNP.vcf
bgzip ref_SNP.vcf
tabix -p vcf ref_SNP.vcf.gz

# Step 4. Reconstrucion of genome fasta with SNPs in above.
Requirement : bcftools v1.15.1
myref=genome.fa
out=genome_ref.fa
vcf=ref_SNP.vcf.gz
bcftools consensus -f ${myref} ${vcf}  > ${out}

# Step 5. Re-mapping ONT data with new fasta with megalodon
Requirement : megalodon v2.5.0
myref=genome_ref.mmi
fast5_pass=fast5
guppy_config=dna_r9.4.1_450bps_sup_prom.cfg
megalodon ${fast5_pass} \
  --guppy-${guppy_config} \
  --remora-modified-bases dna_r9.4.1_e8 sup 0.0.0 5mc CG 0 \
  --outputs basecalls mappings mod_mappings mods \
  --reference ${myref}

# Step 6. Clair3 phasing, 2nd time.
Requirement : Clair3 v0.1.12
Requirement : whatshap v1.4
bam=mapping.bam
ref=genome_ref.fa
MODEL_NAME="r941_prom_sup_g5014"
THREADS=24
run_clair3.sh \
  --bam_fn=${bam} \
  --ref_fn=${myref} \
  --threads=${THREADS} \
  --platform="ont" \
  --model_path="path_to_model_dir/${MODEL_NAME}"

# Step 7. Collect phased SNP data. Chromosome number needs to be adjusted.
Requirement : bgzip, tabix
cd tmp/phase_output/phase_vcf
gzip -dc phased_chr{{1..22},X}.vcf.gz  > ref_phased.vcf
bgzip ref_phased.vcf
tabix -p vcf ref_phased.vcf.gz

# Step 8. Sort BAM file
Requirement : samtools 1.9
samtools sort -o mappings_sort.bam mappings.bam
samtools index mappings_sort.bam

# Step 9. Haplotype phasing ONT reads
Requirement : whstshap v1.14
in=mappings_sort.bam
out=mappings_haplotag.bam
vcf=ref_phased.vcf.gz
myref=genome_ref.fa
haplist=ref_phased_haplotypes.tsv
cpu=24
whatshap haplotag \
  -o ${out} \
  --reference ${myref} \
  --output-threads=${cpu} \
  --output-haplotag-list=${haplist} \
  --ignore-read-groups \
  ${vcf} ${in}

# Step 10. Split ONT-BAM into two alleles.
Requirement : whstshap v1.14
haplist=ref_phased_haplotypes.tsv
whatshap split  \
  --output-h1 mappings_H1.bam  \
  --output-h2 mappings_H2.bam  \
  mappings.bam ${haplist}
whatshap split  \
  --output-h1 mod_mappings_H1.bam  \
  --output-h2 mod_mappings_H2.bam  \
  mod_mappings.bam ${haplist}

# Step 11. Sort BAM files.
Requirement : samtools 1.9
samtools sort   mod_mappings_H1.bam -o mod_mappings_H1_sort.bam ; samtools index  mod_mappings_H1_sort.bam
samtools sort   mod_mappings_H2.bam -o mod_mappings_H2_sort.bam ; samtools index  mod_mappings_H2_sort.bam

# Step 12. ONT bam to 5mC level.
Requirement : modbam2bed v0.5.3
myref=genome_ref.fa
bam1=mod_mappings_H1_sort.bam
bam2=mod_mappings_H2_sort.bam
cpu=24
modbam2bed -e -m 5mC --cpg -t ${cpu} ${myref} ${bam1} > ${bam1%.bam}.bed
modbam2bed -e -m 5mC --cpg -t ${cpu} ${myref} ${bam2} > ${bam2%.bam}.bed

# Step 13. Create 10kb-bin bedgraph file
chrom_size=genome.chrom.size   # chrom.size file, list of chr and N bases, delimitted by TAB.
cat ${chrom_size} ${bam1%.bam}.bed | perl -F'\t' -anle 'BEGIN {$bin=10000;$mod="m";print "track type=bedGraph";@clist=(1..22,"X");$colm=12;$colu=11;$_="chr$_" for @clist;} if (@F < 3) {$size{$F[0]}=$F[1];} else {if ($F[3] eq $mod) {$l=int(($F[1]-1)/$bin);$mC{"$F[0]:$l"}+=$F[$colm];$uC{"$F[0]:$l"}+=$F[$colu]          ;}} END {for $c (@clist) {for $i (0..int($size{$c}/$bin)) {$l2=($i + 1)*$bin; $l2=$size{$c} if $l2 > $size{$c};print join("\t",$c,$i*$bin,$l2,                             ($mC{"$c:$i"}+$uC{"$c:$i"}) > 0 ? $mC{"$c:$i"} / ($mC{"$c:$i"}+$uC{"$c:$i"}) * 100 : 0) if ($mC{"$c:$i"}+$uC{"$c:$i"})>20; }}}' > ${f%.bam}_10kb_5mC.bedgraph
cat ${chrom_size} ${bam2%.bam}.bed | perl -F'\t' -anle 'BEGIN {$bin=10000;$mod="m";print "track type=bedGraph";@clist=(1..22,"X");$colm=12;$colu=11;$_="chr$_" for @clist;} if (@F < 3) {$size{$F[0]}=$F[1];} else {if ($F[3] eq $mod) {$l=int(($F[1]-1)/$bin);$mC{"$F[0]:$l"}+=$F[$colm];$uC{"$F[0]:$l"}+=$F[$colu]          ;}} END {for $c (@clist) {for $i (0..int($size{$c}/$bin)) {$l2=($i + 1)*$bin; $l2=$size{$c} if $l2 > $size{$c};print join("\t",$c,$i*$bin,$l2,                             ($mC{"$c:$i"}+$uC{"$c:$i"}) > 0 ? $mC{"$c:$i"} / ($mC{"$c:$i"}+$uC{"$c:$i"}) * 100 : 0) if ($mC{"$c:$i"}+$uC{"$c:$i"})>20; }}}' > ${f%.bam}_10kb_5mC.bedgraph

# Step 14. Swap REF and ALT bases based at GT tag labelled with "1|0", output SNP list for SNPsplit tool.
gzip -dc ref_phased.vcf.gz | perl -F'\t' -anle '@t=split(/:/,$F[9]);@F[3,4] = @F[4,3] if $t[0] eq "1|0";print join("\t","$F[0]_" . $i++,@F[0,1],1,"$F[3]/$F[4]","$F[0]:$t[4]") if $t[4] =~/^\d+$/;' > ref_phased_snpsplit.txt

# Step 15. Split BAM for Methylome and RNAseq into two alleles.
Requirement : SNPsplit v0.3.2
SNPsplit --snp_file ref_phased_snpsplit.txt          --bisulfite methylome.bam
SNPsplit --snp_file ref_phased_snpsplit.txt                      rnaseq.bam

