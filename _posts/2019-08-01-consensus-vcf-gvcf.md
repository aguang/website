---
title: Building a consensus sequence with vcf files
draft: false
categories: ["posts"]
tags: ["vcfs","gatk","bcftools","phylogenies"]
---

Note: this post has also been [cross-posted](https://cbc.brown.edu/blog/consensus-vcf/) to the Center for Computational Biology of Human Disease at Brown, where I currently work. References to Oscar and CBC have to do with our computing cluster (named Oscar) and the shorthand for our center (CBC).

Building a consensus sequence from a VCF file is [apparently](https://www.biostars.org/p/65885/) [asked](https://www.biostars.org/p/65885/) [a lot](https://www.biostars.org/p/134834/). Different use cases for it exist, one of which is to build phylogenies.

In theory, this should be easy: go along the reference and replace the reference base call with the SNP call instead. Indeed, `bcftools consensus` from [bcftools](https://samtools.github.io/bcftools/bcftools.html) should do the trick perfectly well.

Alas, it is not so. Some of the difficulty appears to be inconsistencies in the VCF files produced by different tools. The [VCF format specification](http://samtools.github.io/hts-specs/) by hts-specs is not necessarily followed, in particular by GATK which uses `<NON_REF>` to indicate symbolic alleles in genomic VCFs. [According to them](https://www.broadinstitute.org/gatk/guide/article?id=4017):

> "The first thing you'll notice, hopefully, is the `<NON_REF>` symbolic allele listed in every record's `ALT` field. This provides us with a way to represent the possibility of having a non-reference allele, and to indicate our confidence either way."

Meanwhile, VCF4.3 explicitly uses `<*>` to indicate the unspecified alternate allele. Previous versions of samtools used `X` and `<X>` as well although this was not so well documented. `bcftools` however explicitly does not handle any symbolic alleles except `<DEL>`, including `<*>`. A detailed history explaining the adoption of different symbols as unspecified alternate alleles is documented in [an issue on samtools/hts-specs](https://github.com/samtools/hts-specs/issues/352). Thus, depending on how your VCF files were produced you may or may not be able to use a consensus building method.

So, what should you do?

If your VCF files are from GATK, then recent versions of GATK4 now have [`FastaAlternateReferenceMaker`](https://software.broadinstitute.org/gatk/documentation/tooldocs/3.8-0/org_broadinstitute_gatk_tools_walkers_fasta_FastaAlternateReferenceMaker.php), which is simple to run on gVCF/VCF files from GATK4. This is the easiest solution. On Oscar using CBC's conda environments, this is simply
```
gatk4 FastaAlternateReferenceMaker -R $REFERENCE -O $CONSENSUS_FASTA -V $VCF
```

Another possibility is to run `sed s/<NON_REF>// $INPUT_VCF` then run bcftools. However, you must be careful with this, as if you run it on genomic VCFs then converters like `bcftools convert` will not work, but neither will they give an error. Instead you will just get a file with only the positions present in the genomic VCF. If you do this, the correct order to run commands instead should be:

```
bcftools convert --gvcf2vcf --fasta-ref $REFERENCE -o ${VCF} ${GVCF}
sed 's/<NON_REF>//' ${VCF} > ${SED.VCF}
bcftools view -Oz -o ${SED.VCF.GZ} ${SED.VCF}
bcftools index ${SED.VCF.GZ}
bcftools consensus -f $REFERENCE -o $CONSENSUS_FASTA ${SED.VCF.GZ}
```

So a bit more involved. From running both methods, I found that they produce the same consensus sequences, but base calls made as `N` by `FastaAlternateReferenceMaker` get called as one of the other ambiguous [IUPAC](https://www.bioinformatics.org/sms/iupac.html) codes by `bcftools`. I'm not sure of why that is, but my guess would be it has to do with removing `<NON_REF>` from the VCFs. Ultimately I don't think it affects multiple sequence alignment much.

If your VCF files are from bcftools/samtools mpileup, then their consensus method should work just fine, and they detail how to get a consensus [in their manual](https://samtools.github.io/bcftools/howtos/consensus-sequence.html).