# Datasets

warp ships no data. Every dataset below is a description: where it came from,
who made it, under what terms, which files it consists of, and how to read the
format once a file is on disk. Fetching is the user's step, and so is agreeing
to whatever the publisher asks.

## The digests are not pinned

`src/datasets.tw` carries a `sha256` field per file and every one of them is
currently the empty string, which `verify` treats as a failure. That is the
honest state of this repository and not an oversight.

A digest copied from a third-party page without checking it against the real
file is worse than no digest: it looks like verification, it will match whatever
that third party is serving, and it silences the one check that would have
caught a substitution. Pinning them needs the files in hand and a machine that
downloaded them from the publisher.

Until then `verify` refuses and prints the digest it computed, in the form:

```
train-images-idx3-ubyte.gz has no pinned digest in warp. Its sha256 is <hex>.
Record it in src/datasets.tw and in docs/datasets.md before training on it.
```

Refusing rather than warning is deliberate. A warning in a training log is not
read, and the cost of a false refusal is a minute of someone's attention while
the cost of a false pass is an experiment run on the wrong bytes.

**Open item:** download each file from the URL in `src/datasets.tw`, verify it
against the publisher's own checksum where one is published, and record the
digest in both places.

Running `verify` and pasting back what it printed does not close this. That
records what warp read off whatever the mirror served, which is the thing the
digest is supposed to be independent of. The README shows the refusal, digest
and all, precisely because that number is not yet worth pinning.

**Half of that is now one command.** `fetch` downloads a dataset and `digests`
reads out what is on disk, so "the files in hand" no longer means doing it by
hand (`docs/needs.md` entry 10, delivered by twill's process interface). What
they cannot supply is the independent half: the number `digests` prints came
from the same download it is meant to check. It was run on 2026-09-01 against
MNIST and Fashion-MNIST, every file the size the table already expected, and
the digests are deliberately **not** recorded here: a second source has to
agree with them first, and that is what the paragraph above says and still
means.

Expect it to be slow. `std/hash` is SHA-256 written in twill, and MNIST's
9.9 MB training file took about four minutes on the machine that produced the
line in the README. `verify` checking the size first is what keeps that cost
off the common failure.

## Fetching

```bash
twill run tools/fetch.tw mnist ./data            # fetch, then verify
twill run tools/fetch.tw --digests mnist ./data  # read back what is on disk
```

From twill, the same two: `ds.fetch(ds.mnist(), "./data")` and
`ds.digests(ds.mnist(), "./data")`.

`fetch` downloads the files a dataset names, into a directory, and then puts
them through `verify`, including the refusal above, which is why a first fetch
of anything ends in "has no pinned digest". A file that is already there is left
alone; a transfer that fails deletes its partial file so the next run's size
check is not answering about a fragment.

Three things it deliberately is not:

- **Automatic.** No loader calls it. A data-loading library that silently
  downloads a hundred and seventy megabytes surprises people, which is the
  argument `docs/needs.md` entry 10 made against wanting this at all. It is
  something a person types, and it prints each file's size before fetching it.
- **A network stack.** It drives `curl`, which the user already has, through
  twill's `run`. `run` takes an argument vector and never a shell, so a URL
  reaches curl as one argument and cannot become a command, and the URL is
  warp's own, out of the table in `src/datasets.tw`, never a caller's.
- **A decompressor.** `docs/needs.md` entry 9 is still open: the `.gz` files
  stay compressed and the manual `gunzip` step stays in the getting-started
  guide. Shelling out to `gunzip` is now possible and has not been done, because
  which files warp is willing to write is a bigger decision than which it is
  willing to read.

`curl --fail` is what makes an HTTP error an exit status rather than a saved
error page, and `--location` follows the redirect these URLs actually serve.
curl missing is reported as itself rather than as a failed download.

## The size check comes first

`verify` compares the file size before it computes a digest. Reading 170
megabytes to discover that the download was an HTML error page is a slow way to
learn it, and a wrong size is the overwhelmingly common failure.

## MNIST

- **Origin.** Yann LeCun and Corinna Cortes. Derived from NIST Special
  Databases 1 and 3.
- **Cite.** LeCun, Bottou, Bengio and Haffner, Gradient-based learning applied
  to document recognition, Proc. IEEE 86(11), 1998.
- **Licence.** CC BY-SA 3.0, as stated on the distribution page. The underlying
  NIST databases are US government work; the split and normalisation are the
  authors'.
- **Format.** IDX, gzipped. 60,000 training and 10,000 test images, 28 by 28,
  one channel, unsigned bytes. Labels 0 to 9.
- **Note.** The original `yann.lecun.com` host has been unreliable for years.
  warp points at the CVDF mirror on Google Cloud Storage, which serves the same
  four files. The digest, once pinned, is what makes that mirror safe to use.

## Fashion-MNIST

- **Origin.** Zalando Research.
- **Cite.** Xiao, Rasul and Vollgraf, Fashion-MNIST: a novel image dataset for
  benchmarking machine learning algorithms, arXiv:1708.07747, 2017.
- **Licence.** MIT.
- **Format.** Identical to MNIST, deliberately, so the same reader handles both.
- **Note.** Ten clothing categories, not digits. It exists because MNIST is too
  easy to distinguish between methods, and swapping one for the other is a
  one-line change here for exactly that reason.

## CIFAR-10

- **Origin.** Alex Krizhevsky, Vinod Nair and Geoffrey Hinton, University of
  Toronto. A labelled subset of the 80 Million Tiny Images collection.
- **Cite.** Krizhevsky, Learning Multiple Layers of Features from Tiny Images,
  technical report, University of Toronto, 2009.
- **Licence.** **Not stated by the authors.** They ask that the technical report
  be cited. That is a request, not a licence, and warp says so rather than
  inventing one. Check the terms before redistributing anything derived from it.
- **Format.** Binary, in a tar archive. 50,000 training and 10,000 test images,
  32 by 32, three channels, one label byte per record.
- **Note.** The parent collection, 80 Million Tiny Images, was withdrawn by its
  authors in 2020 over offensive labels and imagery in the unfiltered set.
  CIFAR-10's ten classes are hand-verified and are not implicated, but anyone
  citing the lineage should know it.

## Iris

- **Origin.** Ronald Fisher, 1936, from measurements by Edgar Anderson.
  Distributed by the UCI Machine Learning Repository.
- **Cite.** Fisher, The use of multiple measurements in taxonomic problems,
  Annals of Eugenics 7(2), 1936.
- **Licence.** CC BY 4.0 as distributed by UCI.
- **Format.** CSV, 150 rows, four features, three classes.
- **Note on the download.** The URL used to be UCI's zip while the file warp
  names and sizes is `iris.data`, the CSV inside it, so the row described a
  fetch that could not succeed and the answer here was "unzip first". It now
  names the CSV directly, which returns exactly the 4551 bytes the row already
  expected: the size the table carried is what says the new URL is the right
  file. Iris is the one dataset here that needs no decompression step at all.
- **Note.** Published in a eugenics journal by an author who was a proponent of
  it. It is included because it is everywhere and small enough to be useful in a
  test, and the provenance is recorded because people should know what they are
  citing.

## Adding a dataset

Four things, and none of them is the data:

1. A `Dataset` in `src/datasets.tw` with the origin, the citation and the
   licence position filled in. An empty licence field fails the tests.
2. A `File` per artefact with its URL and expected size.
3. A digest, verified against a file you downloaded yourself.
4. A section here, including anything a user should know before publishing a
   result on it.
