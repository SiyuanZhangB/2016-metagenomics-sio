Short read quality and trimming
===============================

Start up an instance with ami-05384865 and 500 GB of local storage
(:doc:`aws/boot`).  You should also configure your firewall
(:doc:`aws/configure-firewall`) to pass through TCP ports 8000-8888.

Then, `Log into your computer <aws/login-shell.html>`__.

---

You should now be logged into your Amazon computer!  You should see
something like this::

   ubuntu@ip-172-30-1-252:~$

this is the command prompt.

Prepping the computer
---------------------

Before we do anything else, we need to set up a place to work and
install a few things.

First, let's set up a place to work.  Here, we'll make /mnt writeable::

   sudo chmod a+rwxt /mnt

.. note::

   /mnt is the location we're going to use on Amazon computers, but
   if you're working on a local cluster, it will have a different
   location.  Talk to your local sysadmin and ask them where they
   recommend putting lots of short-term working files, i.e. the
   "scratch" space.

----

Installing some software
------------------------

Run::

  sudo apt-get -y update && \
  sudo apt-get -y install trimmomatic fastqc python-pip \
     samtools zlib1g-dev ncurses-dev python-dev

Install anaconda::

  curl -O https://repo.continuum.io/archive/Anaconda3-4.2.0-Linux-x86_64.sh
  bash Anaconda3-4.2.0-Linux-x86_64.sh

Then update your environment and install khmer::

  source ~/.bashrc

  cd
  git clone https://github.com/dib-lab/khmer.git
  cd khmer
  sudo python2 setup.py install

Running Jupyter Notebook
------------------------

Let's also run a Jupyter Notebook in /mnt. First, configure it a teensy bit
more securely, and also have it run in the background::

  jupyter notebook --generate-config
  
  cat >>/home/ubuntu/.jupyter/jupyter_notebook_config.py <<EOF
  c = get_config()
  c.NotebookApp.ip = '*'
  c.NotebookApp.open_browser = False
  c.NotebookApp.password = u'sha1:5d813e5d59a7:b4e430cf6dbd1aad04838c6e9cf684f4d76e245c'
  c.NotebookApp.port = 8000

  EOF

Now, run! ::

  cd /mnt
  jupyter notebook &

You should be able to visit port 8000 on your AWS computer and see the
Jupyter console.  (The password is 'davis'.)

Data source
-----------

We're going to be using a subset of data from `Hu et al.,
2016 <http://mbio.asm.org/content/7/1/e01669-15.full>`__. This paper
from the Banfield lab samples some relatively low diversity environments
and finds a bunch of nearly complete genomes.

(See `DATA.md <https://github.com/ngs-docs/2016-metagenomics-sio/blob/work/DATA.md>`__ for a list of the data sets we're using in this tutorial.)

1. Copying in some data to work with.
-------------------------------------

We've loaded subsets of the data onto an Amazon location for you, to
make everything faster for today's work.  We're going to put the
files on your computer locally under the directory /mnt/data::

   mkdir /mnt/data

Next, let's grab part of the data set::

   cd /mnt/data
   curl -O -L https://s3-us-west-1.amazonaws.com/dib-training.ucdavis.edu/metagenomics-scripps-2016-10-12/SRR1976948_1.fastq.gz
   curl -O -L https://s3-us-west-1.amazonaws.com/dib-training.ucdavis.edu/metagenomics-scripps-2016-10-12/SRR1976948_2.fastq.gz
   
Now if you type::

   ls -l

you should see something like::

   total 346936
   -rw-rw-r-- 1 ubuntu ubuntu 169620631 Oct 11 23:37 SRR1976948_1.fastq.gz
   -rw-rw-r-- 1 ubuntu ubuntu 185636992 Oct 11 23:38 SRR1976948_2.fastq.gz

These are 1m read subsets of the original data, taken from the beginning
of the file.

One problem with these files is that they are writeable - by default, UNIX
makes things writeable by the file owner.  Let's fix that before we go
on any further::

   chmod u-w *

We'll talk about what these files are below.

1. Copying data into a working location
---------------------------------------

First, make a working directory; this will be a place where you can futz
around with a copy of the data without messing up your primary data::

   mkdir /mnt/work
   cd /mnt/work

Now, make a "virtual copy" of the data in your working directory by
linking it in -- ::

   ln -fs /mnt/data/* .

These are FASTQ files -- let's take a look at them::

   less SRR1976948_1.fastq.gz

(use the spacebar to scroll down, and type 'q' to exit 'less')

Question:

* where does the filename come from?
* why are there 1 and 2 in the file names?

Links:

* `FASTQ Format <http://en.wikipedia.org/wiki/FASTQ_format>`__

2. FastQC
---------

We're going to use `FastQC
<http://www.bioinformatics.babraham.ac.uk/projects/fastqc/>`__ to
summarize the data. We already installed 'fastqc' on our computer for
you.

Now, run FastQC on two files::

   fastqc SRR1976948_1.fastq.gz
   fastqc SRR1976948_2.fastq.gz

Now type 'ls'::

   ls -d *fastqc*

to list the files, and you should see:
::
   SRR1976948_1_fastqc.html
   SRR1976948_1_fastqc.zip
   SRR1976948_2_fastqc.html
   SRR1976948_2_fastqc.zip

You can download these files using your Jupyter Notebook console, if you like;
or you can look at these copies of them:

* `SRR1976948_1_fastqc/fastqc_report.html <http://2016-metagenomics-sio.readthedocs.io/en/work/_static/SRR1976948_1_fastqc/fastqc_report.html>`__
* `SRR1976948_2_fastqc/fastqc_report.html <http://2016-metagenomics-sio.readthedocs.io/en/work/_static/SRR1976948_2_fastqc/fastqc_report.html>`__

Questions:

* What should you pay attention to in the FastQC report?
* Which is "better", file 1 or file 2? And why?

Links:

* `FastQC <http://www.bioinformatics.babraham.ac.uk/projects/fastqc/>`__
* `FastQC tutorial video <http://www.youtube.com/watch?v=bz93ReOv87Y>`__

3. Trimmomatic
--------------

Now we're going to do some trimming!  We'll be using
`Trimmomatic <http://www.usadellab.org/cms/?page=trimmomatic>`__, which
(as with fastqc) we've already installed via apt-get.

The first thing we'll need are the adapters to trim off::

  curl -O -L http://dib-training.ucdavis.edu.s3.amazonaws.com/mRNAseq-semi-2015-03-04/TruSeq2-PE.fa

Now, to run Trimmomatic::

   TrimmomaticPE SRR1976948_1.fastq.gz \
                 SRR1976948_2.fastq.gz \
        SRR1976948_1.qc.fq.gz s1_se \
        SRR1976948_2.qc.fq.gz s2_se \
        ILLUMINACLIP:TruSeq2-PE.fa:2:40:15 \
        LEADING:2 TRAILING:2 \                            
        SLIDINGWINDOW:4:2 \
        MINLEN:25

You should see output that looks like this::

   ...
   Input Read Pairs: 1000000 Both Surviving: 885734 (88.57%) Forward Only Surviving: 114262 (11.43%) Reverse Only Surviving: 4 (0.00%) Dropped: 0 (0.00%)
   TrimmomaticPE: Completed successfully

Questions:

* How do you figure out what the parameters mean?
* How do you figure out what parameters to use?
* What adapters do you use?
* What version of Trimmomatic are we using here? (And FastQC?)
* Do you think parameters are different for RNAseq and genomic data sets?
* What's with these annoyingly long and complicated filenames?
* why are we running R1 and R2 together?

For a discussion of optimal trimming strategies, see `MacManes, 2014
<http://journal.frontiersin.org/Journal/10.3389/fgene.2014.00013/abstract>`__
-- it's about RNAseq but similar arguments should apply to metagenome
assembly.

Links:

* `Trimmomatic <http://www.usadellab.org/cms/?page=trimmomatic>`__

4. FastQC again
---------------

Run FastQC again on the trimmed files::

   fastqc SRR1976948_1.qc.fq.gz
   fastqc SRR1976948_2.qc.fq.gz

And now view my copies of these files: 

* `SRR1976948_1.qc_fastqc/fastqc_report.html <http://2016-metagenomics-sio.readthedocs.io/en/work/_static/SRR1976948_1.qc_fastqc/fastqc_report.html>`__
* `SRR1976948_2.qc_fastqc/fastqc_report.html <http://2016-metagenomics-sio.readthedocs.io/en/work/_static/SRR1976948_2.qc_fastqc/fastqc_report.html>`__

Let's take a look at the output files::

   less SRR1976948_1.qc.fq.gz

(again, use spacebar to scroll, 'q' to exit less).

Questions:

* is the quality trimmed data "better" than before?
* Does it matter that you still have adapters!?

Optional: :doc:`kmer_trimming`

Next: :doc:`assemble`
