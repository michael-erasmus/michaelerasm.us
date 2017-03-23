+++
title = "What is TF-IDF? The 10 minute guide"
draft = false
date = "2017-03-17T19:03:27-07:00"

+++

I recently started reading up a bit on <a href="/web/20160731164955/https://en.wikipedia.org/wiki/Tf%E2%80%93idf">tf-idf</a>, which stands for <em>term frequency-inverse document frequency</em>. Tf-idf is a simple, but surprisingly powerful technique which can be used to figure out what a document is 'about'. It's often used in the fields of information retrieval and text mining.

### Documents?

First, let's just define what I mean with document. For our purposes, a document can be thought of all the words in a piece of text, broken down by how frequently each word appears in the text.

Say for example, you had a very simple document such as this quote:</p>

<blockquote>
 <p>Just the fact that some geniuses were laughed at does not imply that all who are laughed at are geniuses. They laughed at Columbus, they laughed at Fulton, they laughed at the Wright brothers. But they also laughed at Bozo the Clown - Carl Sagan</p>
</blockquote>

If we looked at the top frequency of words in the text we get this:

<table style="width:100%">  
 <tr>
   <td><b>Word</b></td>
   <td><b>Frequency</b></td>
  </th>
  <tr>
    <td>laughed</td>
    <td>6</td>
  </tr>
 <tr>
    <td>at</td>
    <td>6</td>
  </tr>
  <tr>
    <td>the</td>
    <td>3</td>
  </tr>
  <tr>
    <td>that</td>
    <td>2</td>
  </tr>
  <tr>
    <td>geniuses</td>
    <td>2</td>
  </tr>
  <tr>
    <td>are</td>
    <td>2</td>
  </tr>
  <tr>
    <td>wright</td>
    <td>1</td>
  </tr>
  <tr>
    <td>bozo</td>
    <td>1</td>
  </tr>
  <tr>
    <td>who</td>
    <td>1</td>
  </tr>
  <tr>
    <td colspan=2 style='center'>...</td>
  </tr>
 </table>

This structure is also often referred to as a Bag of Words. Although we care about how many times a word appear in a document, we ignore the order in which words appear.

### Term Frequency

Now let's think of a way to figure out what this text is about. What are the main themes? What would be great terms to use if I wanted to search for this document? Well a very simple way of course would just to get the highest frequency words, and declare those as 'important'.

The word 'laughed' and 'at' appears a lot, and seems to be quite representative of the text, but what about 'the' and 'that'?

At the same time the words 'geniuses' and 'wright' appears less frequently, but seems to be really relevant as well.

Without any more information, it's quite difficult to only use term frequency as measure of how important a word is to describe a document.

### (Inverse) Document Frequency

But what if we had a whole set of documents to our disposal? This can also be called a *Corpus*.

Using structure we can gleam from how frequently words appear in different documents within a corpus can give us the additional information about how common words are in general.

Once we know how common or rare certain words are in general, we can weigh their importance in any specific document.

So if we see a high frequency of words like 'the' and 'that' in a lot of documents, we can deduce that these are common words and thus consider them to be less important in any particular document. </p>

On the flip side, the words 'geniuses', 'wright' and 'bozo' will be more rare, and thus can better represent our document. This is called the document frequency, the number of documents in corpus that particular a word occurs in.

Since we want rarer words to have a higher weight, we calculate the inverse of the document frequency. All of this is expressed quite simply as a ratio:

```number_of_document_in_corpus / term_document_frequency```

### Putting it all together

Once we have the term frequency and inverse document frequency statistics for any given word, we can calculate it's tf-idf weight simply as product of the two stats.

```tf_idf = term_frequency * inverse_document_frequency```

If we wanted to grab the most important word in a particular document, we can simply calculate a tf-idf score for each word and use the top scores. And that's it, that's all you need to use tf-idf!

### Oh, and just one more thing (dampening)

There is one minor detail I failed to mention, which is that both the term frequency and inverse document frequency is often scaled logarithmically.

Why would we need to do that? Well let's say you have a document that contains the word 'cat' twice. Using tf-idf you might be well find that the document is about cats. But let's say your corpus also contains a document in which the word 'cat' appears 36 times.

Both these documents might be about cats, but could we say the one is 18 times 'more' about cats?

If you think about it, term frequency isn't a linear indicator of how important a word is in a document. As the number of times a word appears in a document goes up, it's relative importance starts to flatten out.

A good way to represent this is by using the log10 of the term frequency instead. Since the log of 1 is zero and the log of zero is infinity, we actually use

```log(tf) + 1```


We also use the log of the inverse document frequency for the same reason, since we don't want super rare terms to be weighted too highly.

### Can you show me some code?

Yeah of course! Here is a quick and dirty implementation of tf-idf in python, using pandas. I'm using a dataset that comes baked into scikit-learn. The <a href="/web/20160731164955/http://scikit-learn.org/stable/datasets/twenty_newsgroups.html"><code>fetch_20newsgroups</code></a> function will return a dataset of posts from 20 usenet newsgroups.

For performance reasons, I'm using using a small subset of the dataset.

```python

import os
import math
import re
import pandas as pd
from collections import Counter
from sklearn.datasets import fetch_20newsgroups

#get a subset of the dataset

categories = [
        'alt.atheism',
        'talk.religion.misc',
        'comp.graphics',
        'sci.space',
    ]
docs_data = fetch_20newsgroups(subset='train', categories=categories,
                                shuffle=True, random_state=42,
                                remove=('headers', 'footers', 'quotes'))

#build a pandas dataframe using the filename and data of each post
docs =  pd.DataFrame({
            'filename' : docs_data.filenames,
            'data': docs_data.data
})

#grab the corpus size(we'll use this later for IDF)
corpus_size = len(docs)

#no let's do some basic cleaning up of the text, make everything lower case and strip out all non-letters
docs['words'] = docs.data.apply(lambda doc: re.sub("[\W\d]", " ", doc.lower().strip()).split())

#let's calculate the word frequencies for each document (Bag of words)
docs['frequencies'] = docs.words.apply(lambda words: Counter(words))

#cool, now we can calculate TF, the log+1 of the frequency of each word
docs['log_frequencies'] = docs.frequencies.apply(lambda d: dict([(k,math.log(v) + 1) for k, v in d.iteritems()]))

#now let's build up a lookup list of document frequencies
#first we build a vocabulary for our corpus(set of unique words)
corpus_vocab = set([word for words in docs.words for word in words])

#now use the vocabulary to find the document frequency for each word
df = lambda word: len(docs[docs.words.apply(lambda w: word in w)])
corpus_vocab_dfs = dict([(word,math.log(corpus_size / df(word))) for word in corpus_vocab])

#phew! no let's put it all together. let's calculate tf*idf for each term
tfidf = lambda tfs: dict([(k,v * corpus_vocab_dfs[k]) for k, v  in tfs.iteritems()])
docs['tfidf'] = docs.log_frequencies.apply(tfidf)

#finally we can grab the top 5 weighted terms to get keywords for each document
sorted(docs.tfidf[0], key=docs.tfidf[0].get, reverse=True)[0:5]
docs['keywords'] = docs.tfidf.apply(lambda t: sorted(t, key=t.get, reverse=True)[0:5])
```

[Gist](https://gist.github.com/michael-erasmus/ad16c57cf48eb95a4b63)

The code is reasonably commented, so I hope its clear enough. Let me know if you had any questions.

### And that's a wrap!

That's my short and hopefully clear explanation of tf-idf. The <a href="/web/20160731164955/https://en.wikipedia.org/wiki/Tf%E2%80%93idf">wikipedia page</a> has a lot more useful information and theory to dig into, if that kind of stuff tickles your fancy!
