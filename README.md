
Reed-Solomon Python extension module
Copyright 2005 Shane Hathaway
Latest revision at http://hathawaymix.org/Software/ReedSolomon


Introduction
============

Reed-Solomon (RS) codes add efficient redundancy to data streams.  RS
codes provide an answer to two similar questions: how do you transmit
digital data over noisy and lossy channels, and how do you store data
on media that is sure to degrade?  RS codes generate a configurable
number parity bits corresponding with the data bits.  On the receiving
end, only a portion of the correct data and parity bits is sufficient
for recovering all of the original data.

RS codes can replace simple duplication.  For example, consider a
large file that needs to be stored for a long time.  To ensure its
longevity, you could make three copies of the file and store each copy
in a different place.  Each time a hard drive containing the file
fails, you make a new copy on a different drive.  If you have a lot of
large files, keeping so many copies is a burden.

Using RS codes, you can consume only a little more than the size of
the file while improving recoverability.  If you divide the file into
20 chunks, add 5 RS-encoded parity chunks, and store each chunk on a
different hard drive, RS coding guarantees you can still recover the
whole file even if any 5 hard drives fail.  At the same time, you're
only consuming 25% more space than the size of the file.  Compare that
with the previous solution: with mirroring, you consume 200% more than
the size of the file and risk losing data if 3 hard drives fail at the
same time.

RS codes have been in use for a long time.  Compact Disc technology,
for example, uses RS codes to recover data corrupted by scratches and
other defects.  RS coding is one of many forward error correction
(FEC) codes.  RS coding has some advantages over other codes:

  - RS coding reliably recovers erasures.  (An erasure is a missing
  chunk of data.)  If no corruption is involved, RS codes can recover
  the data if the number of missing chunks is less than or equal to
  the number of parity chunks.

  - Since RS codes were invented 45 years ago, any patents surrounding
  RS codes have most likely expired already.

RS coding, however, is computationally intensive.  There is a good,
pure-Python implementation of RS codes by Emin Martinian [1]_, but it
is too slow for most practical uses.  This extension uses the fast, GPL
Reed-Solomon library by Phil Karn [2]_.

.. [1] http://www.csua.berkeley.edu/~emin/source_code/py_ecc/
.. [2] http://www.ka9q.net/code/fec/


Installation
============

If you're using GCC, before you build the module, set up compiler
optimizations.  This code runs twice as fast if -O2 is set, compared
with no optimization.  It also runs 20% faster if you compile for a
specific processor architecture.  Here is a good setting for an Athlon
XP::

  CFLAGS='-O2 -march=athlon-xp -Wall'
  export CFLAGS

See 'man gcc' for a list of all processor architectures GCC can
optimize for.

Compile and install using the standard Python distutils method:

  python setup.py build
  python setup.py install


Usage
=====

Once the module is installed, you can find reference documentation by
typing "pydoc reedsolomon" at the system command line, or by typing
"import reedsolomon; help(reedsolomon)" in the Python interpreter.
What follows is a brief tutorial.

Start a Python interactive interpreter.  Import the reedsolomon module
and create a Codec object::

  >>> from reedsolomon import Codec
  >>> c = Codec(7, 5)

This codec creates seven-byte code words from five-byte input strings.
In each code word generated by this codec, the first five bytes
consist of the original data and the last two bytes are parity.  This
codec can recover from the loss of any two bytes or the corruption of
any single byte.  Encode a five-byte sequence::

  >>> encoded = c.encode('abcde')
  >>> encoded
  'abcde\x94m'

The encoded string contains the original data followed by two parity
bytes (note that '\x94' is a single character.)  Decode that string::

  >>> c.decode('abcde\x94m')
  ('abcde', [])

The original data has been restored.  The second part of the result
lists the corrections made by the library.  Since there were no errors
in the data, the list of corrections is empty.  Introduce one error::

  >>> c.decode('aZcde\x94m')
  ('abcde', [1])

Using the data and parity bytes, the RS codec restored the byte at
index 1 and reported the correction.  This codec can also recover from
a single corrupt parity byte::

  >>> c.decode('abcdeZm')
  ('abcde', [5])

Now consider the case of erasures.  If you tell the library which
bytes are missing from the input, this codec can handle up to two
erasures.  decode() accepts a second argument that lists the erased
indexes::

  >>> c.decode('a0cde\x94m', [1])
  ('abcde', [1])
  >>> c.decode('a00de\x94m', [1, 2])
  ('abcde', [1, 2])
  >>> c.decode('abcd0\x940', [4, 6])
  ('abcde', [4, 6])

If `P` is the number of parity symbols, `R` is the number of errors
(corrupted bytes), and `S` is the number of erasures, RS coding
guarantees recoverability if the following equation is true::

  2R + S <= P

Here is what happens if there are too many erasures::

  >>> c.decode('a000e\x94m', [1, 2, 3])
  Traceback (most recent call last):
    File "<stdin>", line 1, in ?
  reedsolomon.UncorrectableError: Too many errors or erasures in input

The codec reliably throws an exception when there is an excess of
erasures.  However, if there are too many errors, the codec may fail
to detect the problem and produce an incorrect result.  Here is one
possible result of trying to decode two corrupted bytes using a codec
not designed to handle that much corruption:

  >>> c.decode('aXXde\x94m')
  Traceback (most recent call last):
    File "<stdin>", line 1, in ?
  reedsolomon.UncorrectableError: Corrupted input

In this case, it was fortunate that the corruption was detected.
However, some corruption will go undetected and may even garble the
result::

  >>> c.decode('aX\x0bde\x94m')
  ('aX\x0bd\xfe', [4])

The codec decided that the 'e' at index 4 was the incorrect byte and
changed it to '\xfe'.  In order to handle 2 errors correctly, you need
a codec with 4 parity bytes::

  >>> c = Codec(9, 5)
  >>> encoded = c.encode('abcde')
  >>> encoded
  'abcde\xec\x02\x98\xf8'
  
Now introduce the same errors at indexes 1 and 2 and decode::

  >>> c.decode('aX\x0bde\xec\x02\x98\xf8')
  ('abcde', [1, 2])

With this codec, you can also recover from the loss of any 4 symbols::

  >>> c.decode('a0c0e0\x020\xf8', [1, 3, 5, 7])
  ('abcde', [1, 3, 5, 7])

In some applications, it makes sense to layer a stronger hashing
scheme, such as an MD5 sum, over RS coding.  While RS codes provide no
guarantee that corruption will be detected, it is very improbable that
an unintentionally corrupt data stream will pass an MD5 check.

In RS coding, there is a relationship between the bits per symbol and
the maximum length of each code word::

  n <= 2 ** symsize - 1

With a symbol size of 8 bits, code words can be up to 255 symbols
(bytes) long.  If you don't need all 8 bits, you can change the symbol
size by passing the optional 'symsize' argument to the Codec
constructor.

Speed depends on the codec.  The shorter the code word, the faster
this extension runs.  Running on a 1.7 GHz Pentium M laptop,
'speedtest.py' can encode over 11 MB/s with a (10, 8) codec, but only
3 MB/s with the popular (255, 223) codec.

There are methods for encoding and decoding interleaved chunks of
data: encodechunks(), decodechunks(), and updatechunk().  These are
useful when the chunks of data are reliable but you may be missing
chunks, and you'd like to recover the missing chunks using RS codes.
For example, when transmitting packets over unreliable packet-based
digital networks, the packets received are generally complete and
correct, but you might not receive all the packets.  Transmitted
packets should interleave data from mutliple source packets, enabling
you to recover all of the source data if enough interleaved packets
arrive.  Without interleaving, RS coding could correct bad rows, but
would provide no way to recover missing rows.  See `pydoc reedsolomon`
and `tests/test.py` for more information on using these methods.

The module also provides IntegerCodec, which lets you use symbols with
more than 8 bits.  IntegerCodec currently exists only because the
underlying library has functions for working with integer arrays
rather than character arrays.  To simplify maintenance, IntegerCodec
may be removed from the module in the future.

More background on RS coding can be found at the following sites::

  http://www.4i2i.com/reed_solomon_codes.htm
  http://www.siam.org/siamnews/mtc/mtc193.htm
