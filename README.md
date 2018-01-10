# scScarTrace

Collection of scripts used to extract scar clones in scar libraries produced with scScarTrace. 
Initially, we need pair-end fastq files and the scripts produce a final table of scar-percentage per cell, where each cell is assigned to a give clone determined by scar content. This table can undergo further filtering, such as removing clones with less than a given number of cells, or merging together clones with similar scar content. 

## Initial files
Initially, we have either 8 fastq.gz files for each pair end library, that read as follows:
  * fastqfile_rootname_L001_R1.fastq.gz
  * fastqfile_rootname_L002_R1.fastq.gz
  * fastqfile_rootname_L003_R1.fastq.gz
  * fastqfile_rootname_L004_R1.fastq.gz
  * fastqfile_rootname_L001_R2.fastq.gz
  * fastqfile_rootname_L002_R2.fastq.gz
  * fastqfile_rootname_L003_R2.fastq.gz
  * fastqfile_rootname_L004_R2.fastq.gz
  
In case files from different lanes have already been merged, then we have 2 fastq files:
  * fastqfile_rootname_R1.fastq.gz
  * fastqfile_rootname_R2.fastq.gz

## Mapping
1. *mapFastq.sh fastqfile_rootname outfile_rootname path2bwa* <br/>
  This script takes three input parameters:<br/>
  a) fastqfile_rootname: common part of the name of all fastq.gz files to map <br/>
  b) outfile_rootname: desired name for the outptu files that will be generated <br/>
  c) path2bwa: path to bwa software <br/>
  The script unzips R1 and R2 fastq files and merges files from different lanes. Next, it produces a new fastq file (cbc.fastq) with reads with proper cell-specific barcodes. Finally, it maps the cbc.fastq file using bwa. 
 
2. *gzip outfile_rootname.sam* <br/>
 In order to proceed, it is required to zip the sam file.

3. *python bin/readSAMpileup.py --sam outfile_rootname.sam.gz --out outfile_rootname.pileup* <br/>
 This script goes through all the reads in the sam file, and selects those who have been mapped to the GFP in the proper strand and contain the primer used in the nested PCR for their amplification (GGCCCCGTGCTGCTGCCCGAC) with a Hamming distance equal or less than 3. Then, it counts how many times a given read/scar is seen in each cell. As an output if produces a tabular separated file (tsv) and a pickle file containing the table. 

4. *python bin/realignScars.py --picklein outfile_rootname.pileup.pickle --out outfile_rootname.scartab --th 8* <br/>
 The script remaps pileup reads using the biopython function pairwise2.align.globalms with match, mismatch, open and extend parameters equal to 1, 0.25, -1 and -0.1, respetively. This allows to correct for sequencing errors and pool together reads that belong to the same scar for the same cell. From now on, scars are defined using cigar codes, and the sequence is not used any more. As outptu files we get: <br/>
 a) outfile_rootname.scartab.txt: lists all pileuped reads and corresponding mapping and assigned cigar code <br/>
 b) outfile_rootname.scartab.pickle: pickle version of the previous list <br/>
 c) outfile_rootname.scartab.tsv: table containing number of reads per cell for each scar, denoted now as a cigar <br/>
 
## Filtering 
1. *python bin/findThresholds-QCplots.py outfile_rootname.scartab.tsv* <br/>
 This script takes as an input the file *outfile_rootname.scartab.tsv* and provides possible thresholds to filter out cells where scScarTrace did not work and noisy scars that arise due to sequencing and mapping artifacts. The script provides several output files (examples for all these files can be found in the folder "examples"):<br/>
  * ![sumXcell.txt](https://github.com/anna-alemany/scScarTrace/tree/master/examples/sumXcell.txt): text file with the total number of reads detected per cell. If a cell is absent from the list this implies that no read at all was detected. <br/>
  * ![sumXcell.pdf](https://github.com/anna-alemany/scScarTrace/tree/master/examples/sumXcell.pdf): barplot of the total number of reads detected per cell.<br/>
  * ![sumXcell.hst](https://github.com/anna-alemany/scScarTrace/tree/master/examples/sumXcell.hst): normalized histogram of the total number of reads detected per cell.<br/>
  * ![sumXcell.fit](https://github.com/anna-alemany/scScarTrace/tree/master/examples/sumXcell.fit): text file with results from fitting the normalized histogram of the total number of reads per cell with a double Gaussian function. <br/>
  * ![sumXcell-histo.pdf](https://github.com/anna-alemany/scScarTrace/tree/master/examples/sumXcell-histo.pdf): plot showing the histogram of the total number of detected reads per cell and the correspoinding fit to a double Gaussian function. The x-axis is shown in log10 scale. The plot gives a value _x_ for the minimum of the double Gaussian function, which can be used as a threshold to filter out cells with less reads than _10^x_.<br/>
  * ![sumXscar.txt](https://github.com/anna-alemany/scScarTrace/tree/master/examples/sumXscar.txt): text file with number of reads for each detected scar, identified with the cigar code. <br/>
  * ![sumXscar.hst](https://github.com/anna-alemany/scScarTrace/tree/master/examples/sumXscar.hst): text file with the histogram of the number of reads per scar.<br/>
  * ![sumXscar.fit](https://github.com/anna-alemany/scScarTrace/tree/master/examples/sumXscar.fit): text file with results from fitting the histogram of total number of reads per scar to a three domain piecewise-defined linear function. The third domain corresponds to scars that are highly observed and therefore are real scars; the second domain contains scars whose sequence is a few nucleotides away from scars that are present in the third domain, hence these scars are due to sequencing errors of read scars; the first domain corresponds to scars that are hardly observed and therefore are generated by sequencing noise. The border between the second and the third domain _y_ provides a reasonable threshold to filter out scars that are seen with less that _10^y_ reads.<br/>
  * ![sumXscar.pdf](https://github.com/anna-alemany/scScarTrace/tree/master/examples/sumXscar.pdf): plot showing the sorted number of reads for each scar.<br/>
  * ![sumXscar-histo.pdf](https://github.com/anna-alemany/scScarTrace/tree/master/examples/sumXscar-histo.pdf): plot showing the normalized histogram of number of reads per scar and the fit to the three domain piecewise-defined linear function. This profile is general for all scar libraries. 
 
2. *python bin/filter-normalize.py outfile_rootname.scartab.tsv 10^x 10^y outfile_rname* <br/>
 This script takes as an input the scartab.tsv file, followed by the read threshold for cells, the read threshold for scars, and a root name for the output file. As a guide, we recommend to use _~10^x_ as a threshold to filter out cells with less reads, and _~10^y_ as a threshold to filter out scars with less reads. However, the user can explore different thresholds. As an output, the script generates two files:
 * outfile_rname_filter.txt: text file with a table of raw reads for each scar per cell.
 * outfile_rname_norm-filter.txt: text file with a table of normalized reads for each scar per cell. Normalized reads for each cell have been obtained by dividing the number of reads per scar by the total number of reads per cell, and multiplying the result by 100 to have a percentage. 

3. *python bin/scarPurityHistogram.py outfile_rname_norm-filter.txt outfile_rootname.scartab.txt outscardir_name*
 The script takes as an input the outfile_rname_norm-filter.txt file and the outfile_rootname.scartab.txt in order to provide an histogram of sequence content per each cigar. Hence, we can check how many reads for each nucleotide we have in each position of the read. Therefore, when in a given position there are two nucleotides observed with a 50-50 frequency, we know that we have to be careful with this cigar since it codes two different scars. All histograms are stored in the *outscardir_name* directory (which the script creates in case it does not exist). 

4. *python bin/cleanScarErrors.py outfile_rname_norm-filter.txt outscardir_name outfile2_rname*
 For each cell, the script compares pairwise the sequence of cigars/scars that are observed with a percentage above a scarthreshold set to 0.5 by default. In case the distance between two scar sequencies is lower than a given threshold hdthreshold set to 5 by default, the script assumes that these correspond to the same scar and merges them. 
 As an output, the script generates two files:
 * outfile2_rname.log: log files that details which scars are considered to be sequencing errors of others and which have been pooled for each cell. 
 * outfile2_rname.txt: text file with the table of new scar percentage per cell. 

## Clone extraction
0. *python bin/mergeDF.py n file_1 label_1 file_2 label_2 ... file_n label_n output_merged_rname*
 Until now, all scripts are run independently for each library. The following scripts can be run on merged tables, produced after running the _bin/cleanScarErrors.py_ script. To merge this tables in _outfile2_rname.txt_ one can run this script, giving first the total number of files one wants to merge, followed by the file names (with the path, when necessary), _file_1, file_2, ..., file_n_, and the corresponding labels to append at the column names of each file. The are two output files:
 * output_merged_rname.tsv: table with the percentage of each scar per cell will contain all columns from _file_1, file_2, ..., file_n_ with the appended corresponding label, and merged scars for all files as rows. 
 * output_merged_rname.log: summary of files merged and corresponding labels.  

1. *python bin/hierarchicalClustering.py output_merged_rname.tsv n output3_rname method_label pdfplot_label*
 As a first approach, we cluster cells based on scar pattern using hierarchical or agglomerative clustering. The scripts takes for input parameters: 
 * _output_merged_rname.tsv_: file with input scar percentage table
 * _n_: Number of clusters. This will have an impact for the next filtering step. In this step of the analysis it is not very important to get the correct number of clusters, and it is actually better to set the number of clusters a bit higher than what it really is. 
 * _output3_rname_: root name of output files.
 * _method_label_: label to choose the clustering algorithm approach: "hcl" for hierarchical clustering and "acl" for agglomerative clustering. 
 * _pdfplot_label_: label to activate the generation of a plot with the barplot of scar percentage per cell when set to "y". When this label is equal to "y" the code becomes a bit slower. 
 
 As an output, the script produces four files: 
 * _output3_rname_df.txt_: text file with the scar percentage table which is now transposed and contains a new column with the cluster identity for each cell. 
 * _output3_rname_clust.txt_: text file with the list of cluster assigned to each cell.
 * _output3_rname_centroid.txt_: text file with table of mean scar expression over all cells assigned to each cluster.
 * _output3_rname.gpl_: script to generate scar barplot in ![gnuplot](http://www.gnuplot.info/).
 * _output3_rname_barplot.pdf_: barplot displaying scar percentage per cell, with cells sorted by clone identity. This plot is only generated when _pdfplot_label_ is set to _y_. For the sake of speed, it is better to produce this plot with gnuplot using the script _output3_rname.gpl_. 
 
3. *python bin/cleanCellScarfraction.py output3_rname_df.txt threshold output4_fullname*
 From the ![barplot](https://github.com/anna-alemany/scScarTrace/blob/master/examples/example_raw_barplot.pdf) it can be seen that all cells have residual scars (with a tiny percentage) that consist of scars leaking from other cells. The present script will allow to remove these scars. The script takes, as input parameters:
 * _output3_rname_df.txt_: text file with input scar table, obtained by running the previous script. 
 * _threshold_: percentage threshold, so that scars in a cell that are present less than _threshold_ will be removed in that cell. A common threshold is 3.5. 
 * _output4_fullname_: name for the output file. 
 As an output, the script produces a text file named _output4_fullname_ that contains the new filtered table. 

4. *python bin/HDclustering.py _output4_fullname_ output5_rname pdfplot*
 This script clusters together cells sharing the identical scar profie, without taking into account which is the percentage of each scar in each cell. The script takes as an input: 
 * _output4_fullname_: text file with the table of scar percentage per cell. It does not necessarily have to be the one produced in the previous script, but can also _output3_rname_df.txt_. However, _output4_fullname_ will produced a cleaner output. 
 * _output5_rname_: root for the name of output files.
 * _pdfplot_: label to activate the generation of a plot with the barplot of scar percentage per cell when set to "y". Cells are sorted according to their assigned cluster (clone). The generationo f the plot makes the script slower. 
 As an output, the script produces the following plots:
 * _output5_rname_HD.txt_: text file with the scar percentage table which contains a column with the assigned cluster identity for each cell. This is final.
 * _output5_rname_HDcentroids.txt_: text file with table of mean scar expression over all cells assigned to each cluster.
 * _output5_rname.gpl_: script to generate scar barplot in ![gnuplot](http://www.gnuplot.info/).
  * _output5_rname_HDbarplot.pdf_: barplot displaying scar percentage per cell, with cells sorted by clone identity. This plot is only generated when _pdfplot_ is set to _y_. For the sake of speed, it is better to produce this plot with gnuplot using the script _output5_rname.gpl_. 
 
