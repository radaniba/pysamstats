pysamstats
==========

A Python utility for calculating statistics against genome positions
based on sequence alignments from a SAM or BAM file.

* Source: https://gihub.com/alimanfoo/pysamstats
* Download: http://pypi.python.org/pypi/pysamstats

Installation
------------

Building pysamstats depends on [numpy](http://www.numpy.org/), please install
that first. Then try:

```
$ pip install pysam=0.7.8
$ pip install --upgrade pysamstats
```

N.B., pysamstats also depends on [pysam](http://code.google.com/p/pysam/)
which needs to be installed **before** attempting to install pysamstats. If you have problems installing pysam,
email the [pysam user group](https://groups.google.com/forum/#!forum/pysam-user-group).

**Please note that pysamstats currently only works with the 0.7.x branch of pysam. If you also need to use the 0.8.x branch of pysam for some other purpose, it is recommended that you install pysam 0.7.8 and pysamstats into a virtual environment.**

Alternatively, clone the git repo and build in-place:

```
$ git clone git://github.com/alimanfoo/pysamstats.git
$ cd pysamstats
$ python setup.py build_ext --inplace 
```

Usage
-----

From the command line:

```
$ pysamstats --help
Usage: pysamstats [options] FILE

Calculate statistics against genome positions based on sequence alignments
from a SAM or BAM file and print them to stdout.

Options:
  -h, --help            show this help message and exit
  -t TYPE, --type=TYPE  Type of statistics to print, one of: coverage,
                        coverage_strand, coverage_ext, coverage_ext_strand,
                        coverage_gc, variation, variation_strand, tlen,
                        tlen_strand, mapq, mapq_strand, baseq, baseq_strand,
                        baseq_ext, baseq_ext_strand, coverage_binned,
                        coverage_ext_binned, mapq_binned, alignment_binned,
                        tlen_binned.
  -c CHROMOSOME, --chromosome=CHROMOSOME
                        Chromosome name.
  -s START, --start=START
                        Start position (1-based).
  -e END, --end=END     End position (1-based).
  -z, --zero-based      Use zero-based coordinates (default is false, i.e.,
                        use one-based coords).
  -u, --truncate        Truncate pileup-based stats so no records are emitted
                        outside the specified position range.
  -d, --pad             Pad pileup-based stats so a record is emitted for
                        every position (default is only covered positions).
  -D MAX_DEPTH, --max-depth=MAX_DEPTH
                        Maximum read depth permitted in pileup-based
                        statistics. The default limit is 8000.
  -f FASTA, --fasta=FASTA
                        Reference sequence file, only required for some
                        statistics.
  -o, --omit-header     Omit header row from output.
  -p N, --progress=N    Report progress every N rows.
  --window-size=N       Size of window for binned statistics (default is 300).
  --window-offset=N     Window offset to use for deciding which genome
                        position to report binned statistics against. The
                        default is 150, i.e., the middle of 300bp window.

Pileup-based statistics types (each row has statistics over reads in a pileup column):

    * coverage            - Number of reads aligned to each genome position
                            (total and properly paired).
    * coverage_strand     - As coverage but with forward/reverse strand counts.
    * coverage_ext        - Various additional coverage metrics, including
                            coverage for reads not properly paired (mate
                            unmapped, mate on other chromosome, ...).
    * coverage_ext_strand - As coverage_ext but with forward/reverse strand counts.
    * coverage_gc         - As coverage but also includes a column for %GC.
    * variation           - Numbers of matches, mismatches, deletions,
                            insertions, etc.
    * variation_strand    - As variation but with forward/reverse strand counts.
    * tlen                - Insert size statistics.
    * tlen_strand         - As tlen but with statistics by forward/reverse strand.
    * mapq                - Mapping quality statistics.
    * mapq_strand         - As mapq but with statistics by forward/reverse strand.
    * baseq               - Base quality statistics.
    * baseq_strand        - As baseq but with statistics by forward/reverse strand.
    * baseq_ext           - Extended base quality statistics, including qualities
                            of bases matching and mismatching reference.
    * baseq_ext_strand    - As baseq_ext but with statistics by forward/reverse strand.

Binned statistics types (each row has statistics over reads aligned starting within a genome window):

    * coverage_binned     - As coverage but binned.
    * coverage_ext_binned - As coverage_ext but binned.
    * mapq_binned         - Similar to mapq but binned.
    * alignment_binned    - Aggregated counts from cigar strings.
    * tlen_binned         - As tlen but binned.

Examples:

    pysamstats --type coverage example.bam > example.coverage.txt
    pysamstats --type coverage --chromosome Pf3D7_v3_01 --start 100000 --end 200000 example.bam > example.coverage.txt

Version: 0.15.1 (pysam 0.7.7)
```

From Python:

```python
import pysam
import pysamstats

mybam = pysam.Samfile('/path/to/your/bamfile.bam')

# iterate over statistics, one record at a time
for rec in pysamstats.stat_coverage(mybam, chrom='Pf3D7_01_v3', start=10000, end=20000):
    print rec['chrom'], rec['pos'], rec['reads_all'], rec['reads_pp']
    ...

```

For convenience, functions are provided for loading data directly into numpy arrays, e.g.:

```python
import pysam
import pysamstats
import matplotlib.pyplot as plt

mybam = pysam.Samfile('/path/to/your/bamfile.bam')
a = pysamstats.load_coverage(mybam, chrom='Pf3D7_01_v3', start=10000, end=20000)
plt.plot(a.pos, a.reads_all)
plt.show()
```

For pileup-based statistics function, note the following:

* By default a row is emitted for all genome positions covered by reads overlapping the selected region. This means rows will be emitted for positions outside the selected region, but statistics may not be accurate as not all reads overlapping that position will have been counted. To truncate output to exactly the selected region, provide a ``truncate=True`` keyword argument.
* By default a row is only emitted for genome positions covered by at least one read. To emit a row for every genome position, provide a ``pad=True`` keyword argument.
* By default the number of reads in a pileup column is limited to 8000. To increase this limit, provide a ``max_depth=100000`` keyword argument (or whatever number is suitable for your situation).

Field definitions
-----------------

The suffix **_fwd** means the field is restricted to reads mapped to
the forward strand, and **_rev** means the field is restricted to
reads mapped to the reverse strand. E.g., **reads_fwd** means the
number of reads mapped to the forward strand.

The suffix **_pp** means the field is restricted to reads flagged as
properly paired. 

* **chrom** - Chromosome name.  

* **pos** - Position within chromosome. One-based by default when
    using the command line, zero-based by default when using the
    python API.

* **reads_all** - Number of reads aligned at the position. N.b., this
    is really the total, i.e., includes reads where the mate is
    unmapped or otherwise not properly paired.

* **reads_pp** - Number of reads flagged as properly paired by the
    aligner.

* **reads_mate_unmapped** - Number of reads where the mate is
    unmapped.

* **reads_mate_other_chr** - Number of reads where the mate is mapped
    to another chromosome.

* **reads_mate_same_strand** - Number of reads where the mate is
    mapped to the same strand.

* **reads_faceaway** - Number of reads where the read and its mate are
    mapped facing away from each other.

* **reads_softclipped** - Number of reads where there is some
    softclipping at some point in the read's alignment (not
    necessarily at this position).

* **reads_duplicate** - Number of reads that are flagged as duplicate.

* **dp_normed_median** - Number of reads divided by the median number
    of reads over all positions in the specified region, or whole
    genome if no region specified.

* **dp_normed_mean** - Number of reads divided by the mean number of
    reads over all positions in the specified region, or whole genome
    if no region specified.

* **dp_percentile** - Percentile within which the number of reads falls
    considering all positions in the specified region, or whole genome
    if no region specified.

* **gc** - Percentage GC content in the reference at this position
    (depends on window length and offset specified).

* **dp_normed_median_gc** - As *dp_normed_median* but normalised by
    positions with the same percent GC composition.

* **dp_normed_mean_gc** - As *dp_normed_mean* but normalised by
    positions with the same percent GC composition.

* **dp_percentile_gc** - As *dp_percentile* but only considering
    positions with the same percent GC composition.

* **matches** - Number of reads where the aligned base matches the
    reference.

* **mismatches** - Number of reads where the aligned base does not
    match the reference (but is not a deletion).

* **deletions** - Number of reads where there is a deletion in the
    alignment at this position.

* **insertions** - Number of reads where there is an insertion in the
    alignment at this position.

* **A/C/T/G/N** - Number of reads where the aligned base is an A/C/T/G/N.

* **mean_tlen** - Mean value of outer distance between reads and their
    mates for paired reads aligned at this position. N.B., leftmost
    reads in a pair have a positive tlen, rightmost reads have a
    negative tlen, so if there is no strand bias, this value should be
    0.

* **rms_tlen** - Root-mean-square value of outer distance between
    reads and their mates for paired reads aligned at this position.

* **std_tlen** - Standard deviation of outer distance between reads
    and their mates for paired reads aligned at this position.

* **reads_mapq0** - Number of reads where mapping quality is zero.

* **rms_mapq** - Root-mean-square mapping quality for reads aligned at
    this position.

* **max_mapq** - Maximum value of mapping quality for reads aligned at
    this position.

* **rms_baseq** - Root-mean-square value of base qualities for bases
    aligned at this position.

* **rms_baseq_matches** - Root-mean-square value of base qualities for
    bases aligned at this position where the base matches the
    reference.

* **rms_baseq_mismatches** - Root-mean-square value of base qualities
    for bases aligned at this position where the base does not match
    the reference.


Release notes
-------------

Release notes are available from the github releases page: https://github.com/alimanfoo/pysamstats/releases
