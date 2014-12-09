js-bv-common
============

Javascript common functionality across tabix and bai index file processing

This is a support lib of functions that are common across both local
VCF and local BAM processing.  Not intended to be used independently,
but as part of js-local-vcf or js-local-bam.

---

Provides the implementations of index reader functions:

* `bin2Ranges`
* `bin2Beg`
* `bin2End`
* `getChunks`


---

Provides the common top level bin and virtual address operations:

```javascript
function bin2Recs (binid)
```
Bin half closed half open record interval calculator.  Returns a
vector, [interval, k, l, sl, ol], where

* interval is the half closed half open record interval
* k is the corresponding binid covering the interval
* l is the level of the bin
* sl is the size of the bin (at level l)
* ol is the offset of the bin (at level l)


```javascript
function reg2bin(beg, end)
```
calculate bin given an alignment covering [beg,end) (zero-based,
half-close-half-open)


```javascript
function reg2bins(beg, end)
```
calculate the list of bins that may overlap with region [beg,end)
(zero-based)


```javascript
function rshift16 (val)
```
Right shift val 16 bits.  For a chunk beg/end value, this gives the
virtual file offset address.


```javascript
function low16 (val)
```
Get the lower 16 bits of val as an integer.  For a chunk beg/end
value, this gives the starting or ending offset into the inflated
chunk.


```javascript
var start16kbBinid = 4681;
var end16kbBinid = 37449;
var _16KB = 16384;
var _65kb = Math.pow(2, 16);
```

Various usefully named constants used in calculations


---

Provides the common 'wrapped parser' capabilities.  Generally, these
are not something a client would use directly.  But could...


```javascript
function wrappedJParser (file, start, fmt, cb)
```
Single block inflation wrapped parser.  FILE is the file to
associate with parser, START is the virtual offset for the first
block, FMT is the parser definition, and CB is the user level call
back to call with the resulting wrapped parser.

Inflates block starting at start, obtains the next two virtual
(bgzf) block offsets after this block for the initial
cur|nxt(FileOffset), creates a jParser for the buffer resulting
from the block inflation and adds wrapping attributes via
_wrapperCB (see above).

Calls the user call back CB with the resulting parser


```javascript
function wrappedRegionJParser (file, beg, end, fmt, cb)
```
Region inflation wrapped parser.  FILE is the file to associate
with parser, BEG is the starting virtual offset for the first block
in the region, END is the ending virtual offset for the last block
in the region, FMT is the parser definition, and CB is the user
level call back to call with the resulting wrapped parser.

Inflates the region of file from beg to end, uses end as the
initial curFileOffset, obtains the next virtual (bgzf) block offset
after end for the initial nxtFileOffset, creates a jParser for the
buffer resulting from the region inflation and adds wrapping
attributes via _wrapperCB (see above).

Calls the user call back CB with the resulting parser


```javascript
function nextParseRes (p, parse_rule, cb)
```

Mid level function.  Takes a jParser instance P and a parser
production / rule PARSE_RULE, a string naming a parse production in
P, and attempts to advance the current parse with parse_rule.  On
success, calls cb with the result of the parse. P must have several
attributes:

* p.theFile : the file being parsed - assumed to be a bgzf file
* p.curFileOffset : the current virtual block in theFile
* p.nxtFileOffset : the next virtual block offset in theFile
* p.offset : the current parse location offset for current parse field
* p.curBuf : the current ArrayBuffer reflecting the slice of theFile being
             parsed.

These attributes are in addition to those for jParser instances,
and must be initialized on P before the first call to nextParseRes.

On failure, checks to see if a RangeError was raised during the
parse attempt.  This indicates that the parse field(s) for rule
parse_rule extends into the next compressed blcok. If so, will
attempt to:

* Obtain and inflate the next block in the file
* Append this to p.curBuf creating an extended ArrayBuffer file slice
* Update the jDataview corresponding buffer in p.view (a jDataView)
* Update the byteLength for the view to reflect new slice size
* Create new DataView for new file slice ArrayBuffer
* Obtain next virtual offset passed the newly inflated block
* Update curFileOffset, and nxtFileOffset to reflect slice extension
* Re-seek to the beginning of failed parse
* Retry the parse by recursion.

Note that all asynchronous file actions are set to synchronize with
next parse attempt.


---

Some auxilliary utility functions.

```javascript
function log2 (x)
```
Standard log base 2.  Used in index reader format bin level calculation


```javascript
function simpleHash (items, key)
```
Simple hash for mapping items to indices.  ITEMS is a seq of objects,
each having field denoted by KEY.  Map item.key to item's index in
ITEMS.  Returns resulting association map (hash map).


```javascript
function cstg (index, buf)
```
Some of the binary encoded strings are straight C null terminated
type strings - no length attribute available in the encoding, so we
need to have a C string reader over a buffer (vector, Uint8Array)
of unsigned bytes.  NOTE: this only works on iso-latin-1 - no UTF8,
but since BAM adheres to this, that should not be an issue here.

BUF is an unsigned byte collection which can be at least 1) a
vector and 2) a Uint8Array.  INDEX is the starting index into BUF
to start a string read.

Returns a vector [newIndex, stg], where newIndex is the index of
BUF past the null terminating character of the string just read, or
the length of BUF if no null found.  STG is the string obtained.