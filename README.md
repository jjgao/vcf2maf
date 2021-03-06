vcf2maf
=======

[![DOI](https://zenodo.org/badge/doi/10.5281/zenodo.14107.svg)](http://dx.doi.org/10.5281/zenodo.14107)

To convert a [VCF](http://samtools.github.io/hts-specs/) into a [MAF](https://wiki.nci.nih.gov/x/eJaPAQ), each variant must be mapped to only one of all possible gene transcripts/isoforms that it might affect. This selection of a single effect per variant, is often subjective. So this project is an attempt to make the selection criteria smarter, reproducible, and more configurable. And the default criteria must lean towards best practices.

Quick start
-----------

Download the latest stable branch of vcf2maf, and view the detailed usage manual:

    curl -LO https://github.com/mskcc/vcf2maf/archive/master.zip; unzip master.zip; cd vcf2maf-master
    perl vcf2maf.pl --man

To download properly versioned releases, [click here](https://github.com/mskcc/vcf2maf/releases) for a list.

If you don't have [VEP](http://useast.ensembl.org/info/docs/tools/vep/index.html) installed, see the sections below. VEP is preferred for it's CLIA-compliant [HGVS formats](http://www.hgvs.org/mutnomen/recs.html), and is used by default. So after installing VEP, you can test the script like so:

    perl vcf2maf.pl --input-vcf data/test.vcf --output-maf data/test.maf

To fill columns 16 and 17 of the output MAF with tumor/normal sample IDs, and to parse out genotypes and allele counts from matched genotype columns in the VCF, use options `--tumor-id` and `--normal-id`. Skip option `--normal-id` if you didn't have a matched normal:

    perl vcf2maf.pl --input-vcf data/test.vcf --output-maf data/test.maf --tumor-id WD1309 --normal-id NB1308

VCFs from variant callers like [VarScan](http://varscan.sourceforge.net/somatic-calling.html#somatic-output) use hardcoded sample IDs TUMOR/NORMAL in the genotype columns of the VCF. To have this script correctly parse the correct genotype columns, while still printing the proper IDs in the output MAF:

    perl vcf2maf.pl --input-vcf data/test_varscan.vcf --output-maf data/test_varscan.maf --tumor-id WD1309 --normal-id NB1308 --vcf-tumor-id TUMOR --vcf-normal-id NORMAL

If you have VEP in a different folder like `/opt/vep`, and cached in `/srv/vep`, there are options available to point the script there:

    perl vcf2maf.pl --input-vcf data/test.vcf --output-maf data/test.maf --vep-path /opt/vep --vep-data /srv/vep

If you have a MAF-like file that you want to reannotate, then use `maf2maf`, which simply runs `maf2vcf` followed by `vcf2maf`:

    perl maf2maf.pl --input-maf data/test.maf --output-maf data/test.vep.maf

Install VEP
-----------

Ensembl's VEP ([Variant Effect Predictor](http://useast.ensembl.org/info/docs/tools/vep/index.html)) is popular for how it picks a single effect per gene as [detailed here](http://www.ensembl.org/info/docs/tools/vep/script/vep_other.html#pick), its CLIA-compliant [HGVS variant format](http://www.hgvs.org/mutnomen/recs.html), and [Sequence Ontology nomenclature](http://useast.ensembl.org/info/genome/variation/predicted_data.html#consequences) for variant effects.

To follow these instructions, we'll assume you have these packaged essentials installed:

    sudo yum install -y curl rsync tar make perl perl-core
    ## OR ##
    sudo apt-get install -y curl rsync tar make perl perl-base

You'll also need `samtools` and `tabix` in your `$PATH`, which can be found at [htslib.org](http://www.htslib.org/download/)

Handle VEP's Perl dependencies using cpanminus to install them under `~/perl5`:

    curl -L http://cpanmin.us | perl - --notest -l ~/perl5 LWP::Simple LWP::Protocol::https Archive::Extract Archive::Tar Archive::Zip CGI DBI Time::HiRes

Set PERL5LIB to find those libraries. Add this to the end of your `~/.bashrc` to make it persistent:

    export PERL5LIB=~/perl5/lib/perl5:~/perl5/lib/perl5/x86_64-linux

Create temporary shell variables pointing to where we'll store VEP and its cache data (non default paths can be used, but specify `--vep-path` and `--vep-data` when running vcf2maf):

    export VEP_PATH=~/vep
    export VEP_DATA=~/.vep

Download the v81 release of VEP:

    mkdir $VEP_PATH $VEP_DATA; cd $VEP_PATH
    curl -LO https://github.com/Ensembl/ensembl-tools/archive/release/81.tar.gz
    tar -zxf 81.tar.gz --starting-file variant_effect_predictor --transform='s|.*/|./|g'

Download and unpack VEP's offline cache for GRCh37, GRCh38, and GRCm38:

    rsync -zvh rsync://ftp.ensembl.org/ensembl/pub/release-81/variation/VEP/homo_sapiens_vep_81_GRCh{37,38}.tar.gz $VEP_DATA
    rsync -zvh rsync://ftp.ensembl.org/ensembl/pub/release-81/variation/VEP/mus_musculus_vep_81_GRCm38.tar.gz $VEP_DATA
    cat $VEP_DATA/*_vep_81_GRC{h37,h38,m38}.tar.gz | tar -izxf - -C $VEP_DATA

Install the Ensembl API, the reference FASTAs for GRCh37/GRCh38/GRCm38, and some neat VEP plugins:

    perl INSTALL.pl --AUTO afp --SPECIES homo_sapiens,mus_musculus --ASSEMBLY GRCh38,GRCm38 --PLUGINS CADD,ExAC,dbNSFP,UpDownDistance --DESTDIR $VEP_PATH --CACHEDIR $VEP_DATA
    perl INSTALL.pl --AUTO afp --SPECIES homo_sapiens --ASSEMBLY GRCh37 --PLUGINS CADD,ExAC,dbNSFP,UpDownDistance --DESTDIR $VEP_PATH --CACHEDIR $VEP_DATA

Convert the offline cache for use with tabix, that significantly speeds up the lookup of known variants:

    perl convert_cache.pl --species homo_sapiens,mus_musculus --version 81_GRCh37,81_GRCh38,81_GRCm38 --dir $VEP_DATA

Download and tabix-index the VCF needed by VEP's ExAC plugin:

    cd $VEP_DATA
    curl -LO ftp://ftp.broadinstitute.org/pub/ExAC_release/release0.3/ExAC.r0.3.sites.vep.vcf.gz
    tabix -p vcf ExAC.r0.3.sites.vep.vcf.gz

Download and tabix-index the datafile needed by VEP's dbNSFP plugin:

    cd $VEP_DATA
    curl -LO ftp://dbnsfp:dbnsfp@dbnsfp.softgenetics.com/dbNSFPv3.0c.zip
    unzip dbNSFPv3.0c.zip
    cat dbNSFP*chr* | bgzip -c > dbNSFP.gz
    tabix -s 1 -b 2 -e 2 dbNSFP.gz

Test running VEP in offline mode with the ExAC plugin, on the provided sample GRCh37 VCF:

    cd $VEP_PATH
    perl variant_effect_predictor.pl --species homo_sapiens --assembly GRCh37 --offline --no_progress --everything --shift_hgvs 1 --check_existing --check_alleles --total_length --allele_number --no_escape --xref_refseq --dir $VEP_DATA --fasta $VEP_DATA/homo_sapiens/81_GRCh37/Homo_sapiens.GRCh37.75.dna.primary_assembly.fa --plugin ExAC,$VEP_DATA/ExAC.r0.3.sites.vep.vcf.gz --input_file example_GRCh37.vcf --output_file example_GRCh37.vep.txt

Authors
-------

    Cyriac Kandoth (ckandoth@gmail.com)

License
-------

    Apache-2.0 | Apache License, Version 2.0 | https://www.apache.org/licenses/LICENSE-2.0
