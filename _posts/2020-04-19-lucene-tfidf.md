---
layout: post
title:  "Followup - Lucene"
date:   2020-04-19
categories:
---

## Summary

To follow up Lucene, I have a simple use case: given a corpus of documents, e.g. an artist's most popular songs, a collection of notes, or a blog website such as this one, I can compute the classic TfIdf to get a quick summary of "most descriptive" words in that corpus.


## Lucene fundamental data hierarchy:

- corpus
  - 1 corpus => many documents
    - 1 document => many fields (filename, create_time, content, ...)
      - 1 field => many terms (integers, longs, tokenized/non-tokenized strings). field data can be marked as stored and/or inverted.
        - (optionally) 1 term or token => many attributes (position, offsets, flags, part-of-speech tags)


## A 4-D View of the Inverted Index

This section from the paper was especially well-written.

>The  Codec  API  presents  inverted  index  data  as  a  logical  four-dimensional  table  that  can  be  traversed  using  enumerators.  The dimensions  are:

```
field,  term,  document,  and  position
```

>that  is,  an imaginary cursor can be advanced along rows and columns of this table  in  each  dimension,  and  it  supports  both  “next  item”  and “seek  to  item”  operations,  as  well  as  retrieving  row  and  cell  data at the current position. For example, given a cursor at field **field1** and term **term1** the  cursor  can  be  advanced  along  this  posting  list  to  the data from document **doc1** (and subsequent matching docs d2, d3, ...),  where  the  in-document  frequency  for this term (TF) can be retrieved, and then positional data can be iterated to retrieve consecutive positions, offsets, and payload data at each position within this document.


## To output top words sorted by TFIDF from a corpus (lyrics, blog entries, etc.)

The steps should be simple:

1. index the corpus
2. retrieve all terms from the index where field="body"
3. compute tfidf for each term
4. sort, truncate to top 10 for example


I have updated this website header to include "top words" as a summary. Each word link to the result of a Lucene search query. The full code is at [https://github.com/warrenqi/lucene-lyrics-www](https://github.com/warrenqi/lucene-lyrics-www)