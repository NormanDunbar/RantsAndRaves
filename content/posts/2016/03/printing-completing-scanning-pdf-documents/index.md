---
title: "Printing, Completing & Scanning PDF Documents"
date: "2016-03-29"
categories: 
  - "linux"
  - "rants-raves"
---

As a contractor I often have to fill in and sign various contract agreements. These are usually tens of pages in length, and while I only have to sign one page, I still must scan in and send back each and every page, even the untouched ones. It would help if a PDF with form filling abilities was supplied, but hey, that's only rarely the case. This is how I do it. (On Linux - Windows users mileage may vary!)

First of all, install `pdftk`. You only have to do this once.

```bash
sudo apt-get update
sudo apt-get install pdftk
```

Your commands may vary, I'm using Linux Mint 17.3 at the time of writing. Use whatever your package manager requires.

Next, create a couple of useful utility scripts to split a large PDF file into separate pages, and another to join the pages back up into one file again.

### Split Out the Pages

The script to split an input PDF file into separate pages is called `splitPDF.sh`, and is as follows. Please be aware that there is very limited error checking - I'm supposed to know what I'm doing! (Famous last words!)

```bash
ORIGINALPDF="${1}"
PAGESPDF="${ORIGINALPDF%%pdf}page_%04d.pdf"
echo "Processing \"${ORIGINALPDF}\" to \"${PAGESPDF}\""

pdftk "${ORIGINALPDF}" \
      burst \
      output "${PAGESPDF}"

echo " "
echo "Pages created."
echo "Copy/Overwrite replacement pages with same file name, then,"
echo "./joinPDF.sh \"${ORIGINALPDF}\"."
echo " "
```

What this code does is accepts the full PDF file name on the command line, wrapped in double quotes if there are spaces or other special characters, and splits it into separate pages. If, for example, the file is called `Contract.pdf`, running the script as `./splitPDF.sh Contract.pdf` will produce a number of files named `Contract.page_0001.pdf`, `Contract.page_0002.pdf` and so on until there is a separate file for each page of the original document.

### All Join Together

To join the separate pages back together again is dependent on the following script, named `joinPDF.sh`:

```bash
ORIGINALPDF="${1}"
UNCOMPRESSEDPDF="${ORIGINALPDF%%pdf}uncompressed.pdf"
COMPLETEDPDF="${ORIGINALPDF%%pdf}completed.pdf"
PAGESPDF="${ORIGINALPDF%%pdf}"

echo "Processing \"${PAGESPDF}page_*.pdf\" to \"${UNCOMPRESSEDPDF}\""

pdftk "${PAGESPDF}"page_*.pdf \
      output "${UNCOMPRESSEDPDF}"

echo "\"${UNCOMPRESSEDPDF}\" created."
echo "Compressing... "

pdftk "${UNCOMPRESSEDPDF}" \
      output "${COMPLETEDPDF}" \
      compress

echo "\"${COMPLETEDPDF}\" created."
```

Again, this is executed with the original file name used when splitting the pages. For example, with our example above, we would run the following command `./joinPDF.sh Contract.pdf`. This will take all the pages named `Contract.page_000n.pdf`, in order, and create a temporary uncompressed file (in some cases, but not all) named `Contract.uncompresses.pdf`. This file will be compressed into the final output file, named `Contract.completed.pdf`.

The scripts will not overwrite the original files, just in case, and all temporary files are left around afterwards as well. You never know.

### In Use

Using the utilities above is simple. The workflow, if it can be called such, is as follows:

- Save a copy of the master file somewhere temporary.
- Split out the individual pages.
- Print, sign, scan the pages that must be signed, etc.
- Copy the newly scanned pages over the originals.
- Join everything back together.

So, here's a worked example from my latest contract which required 13 pages to be returned with only page 12 requiring a signature.

```bash
norman@hubble  $ ### List current files:
norman@hubble  $ ls
Contract.pdf  joinPDF.sh  splitPDF.sh
```

```bash
norman@hubble  $ ### Split Contract.pdf into pages:
norman@hubble  $ ./splitPDF.sh Contract.pdf 
Processing "Contract.pdf" to "Contract.page_%04d.pdf"
 
Pages created.
```

Copy/Overwrite replacement pages with same file name, then:

```bash
./joinPDF.sh "Contract.pdf".
```

```bash
norman@hubble  $ ### Check result:
norman@hubble  $ ls
Contract.page_0001.pdf  Contract.page_0007.pdf  Contract.page_0013.pdf
Contract.page_0002.pdf  Contract.page_0008.pdf  Contract.pdf
Contract.page_0003.pdf  Contract.page_0009.pdf  doc_data.txt
Contract.page_0004.pdf  Contract.page_0010.pdf  joinPDF.sh
Contract.page_0005.pdf  Contract.page_0011.pdf  splitPDF.sh
Contract.page_0006.pdf  Contract.page_0012.pdf
```

```bash
norman@hubble  $ ### Overwrite original page 12 with signed & scanned page 12:
norman@hubble  $ cp /tmp/Contract_page_12.pdf ./Contract.page_0012.pdf
```

```bash
norman@hubble  $ ### Merge originals plus signed page:
norman@hubble  $ ./joinPDF.sh Contract.pdf
Processing "Contract.page_*.pdf" to "Contract.uncompressed.pdf"
"Contract.uncompressed.pdf" created.
Compressing... 
"Contract.completed.pdf" created.
```

```bash
norman@hubble  $ ### Check result:
norman@hubble  $ ls
Contract.completed.pdf  Contract.page_0007.pdf  Contract.pdf
Contract.page_0001.pdf  Contract.page_0008.pdf  Contract.uncompressed.pdf
Contract.page_0002.pdf  Contract.page_0009.pdf  doc_data.txt
Contract.page_0003.pdf  Contract.page_0010.pdf  joinPDF.sh
Contract.page_0004.pdf  Contract.page_0011.pdf  splitPDF.sh
Contract.page_0005.pdf  Contract.page_0012.pdf
Contract.page_0006.pdf  Contract.page_0013.pdf
```

And finally, I can email the file `Contract.completed.pdf` back to the agent and I'm ready to start work. :-)
