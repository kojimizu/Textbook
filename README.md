
# Natural Language Processing

Text Mining with R from 2018 May


---
title: "Text Mining with R"
output:
  word_document: default
  html_document: default
---

Text Mining with R - A Tidy Approach
Julia Silge and David Robinson 
https://www.tidytextmining.com/

# Preface 
Tidytext (Silge and Robinson 2016) R package

## Outlines 
- Chapter 1: outlines the tidy text format and the `unnest_tokens()` function. It also introduces the gutenbergr and janaustenr packages, which provide useful literary text datasets that we'll use through this book.
- Chapter 2: shows how to perform sentiment analysis on a tidy text dataset, using `sentiments` dataset from tidytest and `inner_join` from dplyr.
- Chapter 3: describes the tf-idf statistic (term frequency times inverse document frequency), a quantity used for identifying terms that are especially important to a particular document.
- Chapter 4: introdues n-grams and how to analyze word networks in text using the windyr and ggraph packages

Text won't be tidy at all stages of analaysis, and it is important to be able to convert back and forth between tidy and non-tidy formats. 

- Chapter 5: introduces methods for tidying document-term matrices and corpus objects from the tm and quanteda packages, as well as for casting tidy text datasets into those formats. 
- Chapter 6: explores the concep of topic modeling, and uses the `tidy()`
 method to interpret and visualize the output of the topicmodels package.
 
 We conlude with several case studies that bring together multiple tidy text mining approaches we've learnt. 
 
- Chapter 7: demonstrates an application of a tidy text analysis by analyzing the author's own twitter archives. How do Dave's and Julia's tweeting habits compare? 
- Chapter 8: explores metadata from over 32,000 NASA datasets (available in JSON) by looking at how keywords from the datasets are conected to title and description fields.
- Chapter 9: analyzes a dataset of Usenet messages from a diverse set of newsgroups (focused on topics like politics, hocket, technology, atheism, and more) to understand patterns across the groups.

## Notes 
This Book does not cover following topics
- clustering, classification and prediction
- word embedding
- more complex tokenization
- language other than English

# Chapter 1: The tidy text format

Tidy text format - is a table with one-token-per-row

A token is a meaningful unit of text, such as a word
Tokenization is the process of splitting text into tokens 

Tidy datasets allow manipulation with a standard set of "tidy" tools, including popular packages such as dplyr, tidyr, ggplot2 and broom. By keeping the input and output in tidy tables, users can transition between these packages. 

At the same time, the tidytext package doesn't expect a user to keep text data in a tidy form at all times during an analysis. The packages includes functions to `tidy()` objects, such as tm and quanteda. This allows a workflow where importing, filtering and processing is done using dplyr and other tidy tools, after which the data is convered into a document-term matrix for machine learning applications. The models can the be re-converted into a tidy form for interpreation and visualization with ggplot2.


## 1.1 Contrasting tidy text with other data structure

we difine the tidy text format as being a table with one-token-per-row. Structuring text data in this way means that it conforms to tidy data principles and can be manipulated with a set of consistent tools.

- string: Text can be stored as strings (ie., character vectors)
- corpus: These types of objects typically contain raw strings annotated with additional metadata and details. 
- document-term matrix: sparse matrix describing a collection(ie. corpus) of documents with one row for each document and one column for each term. The value in the matrix is typically word count or tf-idf.

## 1.2 The `unnest_tokens` function

Emily Dickinson wrote some lovely text in her time
```{r}
text <- c("Because I could not stop for Death -",
          "He kindly stopped for me -",
          "The Carriage held but just Ourselves -",
          "and Immortality")
text
```

This is a typical character vector that we might want to analyze. In order to turn it into a tidy text dataset, we forst need to put it into a data frame.
```{r}
library(dplyr)
text_df <- data_frame(line=1:4,text=text)
text_df
```

What does it mean that this data frame has printed out as a "tibble"? A tibble is a modern class of data frame within R, available in the dplyr and tibble packages, that has a convenient print method, will not convert strings to factors, and does not use row names. Tibbles are great for use with tidy tools. 

Within our tidy text framework, we need to both break the text into individual tokens (a process called tokenization) and transform it to a tidy data structure. We use tidytext's  `unnest_tokens` function.
```{r unnest_tokens function}
# install.packages("tidytext")
library(tidytext)

# Ctl + Shift + M: %>% pipe operator
text_df
library(magrittr)
text_df$text

text_df %>% unnest_tokens(word,text)
```

The two basic arguments to `unnest_tokens` used here are column names. First, we have the output column name that will be created (`word`), and the input column that the text comes (`text`). 

After using `unnest_tokens`, we've split each row so that there is one token (word) in each row of the new data frame: the default tokenization in `unnest_tokens` is for single words, as shown here.  

## 1.3 Tidying the works of Jane Austen
We use the text of Jane Austen's six completed, published novels from the janaustenr package, and transform them into a tidy format, and also use `mutate()` to annotate a `linenumber` quantify to keep track of lines in the original format and a `chapter` to find where all the chapters are. 

```{r Tidy works of Jane Austen}
library(janeaustenr)
library(dplyr)
library(stringr)

original_books <- austen_books() %>% 
  group_by(book) %>% 
  mutate(linenumber=row_number(),
         chapter=cumsum(str_detect(text,regex("^chapter [\\divxlc]",
                                              ignore_case = T)))) %>% 
  ungroup()

head(original_books,12)
```

To work with this as a tidy dataset, we need to restructure it in the one-token-per-row format, which as we saw earlier is done with the `unnest_tokens` function.

```{r tidy_books unnest_tokens}
library(tidytext)
tidy_books <- original_books %>% 
  unnest_tokens(word,text) 
tidy_books
```

This function uses the `tokenizers` package to separate each line of text in the original data frame into tokens. The default tokenizing is for words, but other options include characters, n-grams, sentences, lines, paragraphs, or separation around a `regex pattern`.

Now that the data is in one-word-per-row format, we can manipulate it with tidy toos like dplyr. Often in text analysis, we will want to remove stop words: stop words are words that are not useful for an analysis, typically extremely common words such as "the", "of", "to" and so forth in English. 

We can remove stop words (kept in the tidytext dataset `stop_words`) with an `anti_join()`
```{r remove stop words}
# stop words dataset
data(stop_words)

# anti-join() application
tidy_books <- tidy_books %>% 
  anti_join(stop_words)
```

The `stop_words` dataset in the tidytext package contains stop words from three lexicons. We can use them all together, as we have here, or filter to only use one set of stop words.

We ca also use dplyr's count() to find the most common words in all the books as a whole.
```{r tidybooks count word}
tidy_books %>% 
  count(word,sort=TRUE)
```

Because we've been using tidy tools, our word counts are stored in a tidy data frame. This allows us to pipe this directly to the gpplot2 package, for example to create a visualization of the most common words 
```{r tidybooks visualization}
library(ggplot2)

tidy_books %>% 
  count(word,sort=T) %>% 
  filter(n>600) %>% 
  mutate(word=reorder(word,n)) %>% 
  ggplot(aes(word,n))+
  geom_col()+
  xlab(NULL)+
  coord_flip()
```

Note that the `auten_books()` function started us with exactly the text we wanted to analyze, but in other cases we may need to perform cleaning of text data, such as removing copy right headers or formattting. You'll examples of this kind of pre-processing in the case study chapters, especially Chapter 9.1.1.

## 1.4 The gutenbergr package

Now that we’ve used the janeaustenr package to explore tidying text, let’s introduce the gutenbergr package (Robinson 2016). The gutenbergr package provides access to the public domain works from the Project Gutenberg collection. The package includes tools both for downloading books (stripping out the unhelpful header/footer information), and a complete dataset of Project Gutenberg metadata that can be used to find works of interest. In this book, we will mostly use the function `gutenberg_download()` that downloads one or more works from Project Gutenberg by ID, but you can also use other functions to explore metadata, pair Gutenberg ID with title, author, language, etc., or gather information about authors.

https://ropensci.org/tutorials/gutenbergr_tutorial/

```{r}
library(gutenbergr)
library(dplyr)
library(magrittr)

gutenberg_metadata

# find Gutenberg ID
gutenberg_metadata %>% 
  filter(title=="Wuthering Heights")
```

In many analyses, you may want to filter just for English works, avoid duplicates, and 

## 1.5 Word frequencies

A common task in text mining is to look at word frequencies, just like we have done above for Jane Austen’s novels, and to compare frequencies across different texts. We can do this intuitively and smoothly using tidy data principles. We already have Jane Austen’s works; let’s get two more sets of texts to compare to. First, let’s look at some science fiction and fantasy novels by H.G. Wells, who lived in the late 19th and early 20th centuries. Let’s get The Time Machine, The War of the Worlds, The Invisible Man, and The Island of Doctor Moreau. We can access these works using `gutenberg_download()` and the Project Gutenberg ID numbers for each novel.

```{r}
library(dplyr)
library(magrittr)
library(tidyverse)
library(tidyr)
library(gutenbergr)
```


```{r 1.5 word frequency 1}
# install.packages("gutenbergr")
hgwells <- gutenberg_download(c(35, 36, 5230, 159))
```

```{r 1.5 word frequency 2}
library(magrittr)
tidy_hgwells <- hgwells %>% 
  unnest_tokens(word,text) %>% 
  anti_join(stop_words)
```

Just for kicks, what are the most common words in these novels of H.G. Wells?
```{r 1.5 word frequency 3}
tidy_hgwells %>% 
  count(word,sort=T)
```

Now let’s get some well-known works of the Brontë sisters, whose lives overlapped with Jane Austen’s somewhat but who wrote in a rather different style. Let’s get Jane Eyre, Wuthering Heights, The Tenant of Wildfell Hall, Villette, and Agnes Grey. We will again use the Project Gutenberg ID numbers for each novel and access the texts using gutenberg_download().
```{r word frequency 4}
bronte <- gutenberg_download(c(1260, 768, 969, 9182, 767))

tidy_bronte <- bronte %>% 
  unnest_tokens(word,text) %>% 
  anti_join(stop_words)
```

What are the most common words in these novels of the Brontë sisters?
```{r 1.5 word frequency 5}
tidy_bronte %>% 
  count(word,sort=T)
```

Interesting that “time”, “eyes”, and “hand” are in the top 10 for both H.G. Wells and the Brontë sisters.

Now, let’s calculate the frequency for each word for the works of Jane Austen, the Brontë sisters, and H.G. Wells by binding the data frames together. We can use spread and gather from tidyr to reshape our dataframe so that it is just what we need for plotting and comparing the three sets of novels.

```{r 1.5 word frequency 6}

library(tidyr)
frequency <- bind_rows(mutate(tidy_bronte, author = "Brontë Sisters"),
                       mutate(tidy_hgwells, author = "H.G. Wells"), 
                       mutate(tidy_books, author = "Jane Austen")) %>% 
  mutate(word = str_extract(word, "[a-z']+")) %>%
  count(author, word) %>%
  group_by(author) %>%
  mutate(proportion = n / sum(n)) %>% 
  select(-n) %>% 
  spread(author, proportion) %>% 
  gather(author, proportion, `Brontë Sisters`:`H.G. Wells`)```
```

We use `str_extract()` here because the UTF-8 encoded texts from Project Gutenberg have some examples of words with underscores around them to indicate emphasis (like italics). The tokenizer treated these as words, but we don’t want to count “_any_” separately from “any” as we saw in our initial data exploration before choosing to use `str_extract()`.

Now let’s plot (Figure 1.3).
```{r 1.5 word frequency 7}
library(scales)
# expect a warning about rows with missing values being removed
ggplot(frequency, aes(x = proportion, y = `Jane Austen`, color = abs(`Jane Austen` - proportion))) +
  geom_abline(color = "gray40", lty = 2) +
  geom_jitter(alpha = 0.1, size = 2.5, width = 0.3, height = 0.3) +
  geom_text(aes(label = word), check_overlap = TRUE, vjust = 1.5) +
  scale_x_log10(labels = percent_format()) +
  scale_y_log10(labels = percent_format()) +
  scale_color_gradient(limits = c(0, 0.001), low = "darkslategray4", high = "gray75") +
  facet_wrap(~author, ncol = 2) +
  theme(legend.position="none") +
  labs(y = "Jane Austen", x = NULL)


```

Words that are close to the line in these plots have similar frequencies in both sets of texts, for example, in both Austen and Brontë texts (“miss”, “time”, “day” at the upper frequency end) or in both Austen and Wells texts (“time”, “day”, “brother” at the high frequency end). Words that are far from the line are words that are found more in one set of texts than another. For example, in the Austen-Brontë panel, words like “elizabeth”, “emma”, and “fanny” (all proper nouns) are found in Austen’s texts but not much in the Brontë texts, while words like “arthur” and “dog” are found in the Brontë texts but not the Austen texts. In comparing H.G. Wells with Jane Austen, Wells uses words like “beast”, “guns”, “feet”, and “black” that Austen does not, while Austen uses words like “family”, “friend”, “letter”, and “dear” that Wells does not.

Overall, notice in Figure 1.3 that the words in the Austen-Brontë panel are closer to the zero-slope line than in the Austen-Wells panel. Also notice that the words extend to lower frequencies in the Austen-Brontë panel; there is empty space in the Austen-Wells panel at low frequency. These characteristics indicate that Austen and the Brontë sisters use more similar words than Austen and H.G. Wells. Also, we see that not all the words are found in all three sets of texts and there are fewer data points in the panel for Austen and H.G. Wells.

Let’s quantify how similar and different these sets of word frequencies are using a correlation test. How correlated are the word frequencies between Austen and the Brontë sisters, and between Austen and Wells?

```{r}
cor.test(data=frequency[frequency$author == "Brontë Sisters",],
         ~proportion+"Jane Austen")

cor.test(data = frequency[frequency$author == "H.G. Wells",], 
         ~ proportion + `Jane Austen`)
```

Just as we saw in the plots, the word frequencies are more correlated between the Austen and Brontë novels than between Austen and H.G. Wells.

## 1.6 Summary
In this chapter, we explored what we mean by tidy data when it comes to text, and how tidy data principles can be applied to natural language processing. When text is organized in a format with one token per row, tasks like removing stop words or calculating word frequencies are natural applications of familiar operations within the tidy tool ecosystem. The one-token-per-row framework can be extended from single words to n-grams and other meaningful units of text, as well as to many other analysis priorities that we will consider in this book.

# Reference

Wickham, Hadley. 2014. “Tidy Data.” Journal of Statistical Software 59 (1): 1–23. doi:10.18637/jss.v059.i10.

Wickham, Hadley, and Romain Francois. 2016. dplyr: A Grammar of Data Manipulation. https://CRAN.R-project.org/package=dplyr.

Wickham, Hadley. 2016. tidyr: Easily Tidy Data with  
spread()and gather()
  Functions. https://CRAN.R-project.org/package=tidyr.

Wickham, Hadley. 2009. ggplot2: Elegant Graphics for Data Analysis. Springer-Verlag New York. http://ggplot2.org.

Robinson, David. 2017. broom: Convert Statistical Analysis Objects into Tidy Data Frames. https://CRAN.R-project.org/package=broom.

Ingo Feinerer, Kurt Hornik, and David Meyer. 2008. “Text Mining Infrastructure in R.” Journal of Statistical Software 25 (5): 1–54. http://www.jstatsoft.org/v25/i05/.

Benoit, Kenneth, and Paul Nulty. 2016. quanteda: Quantitative Analysis of Textual Data. https://CRAN.R-project.org/package=quanteda.

Silge, Julia. 2016. janeaustenr: Jane Austen’s Complete Novels. https://CRAN.R-project.org/package=janeaustenr.

Robinson, David. 2016. gutenbergr: Download and Process Public Domain Works from Project Gutenberg. https://cran.rstudio.com/package=gutenbergr.

# Chapter 2: Sentiment analysis with tidy data
In the previous chapter, we explored in depth what we mean by the tidy text format and showed how this format can be used to approach questions about word frequency. This allowed us to analyze which words are used most frequently in documents and to compare documents, but now let’s investigate a different topic. Let’s address the topic of opinion mining or sentiment analysis. When human readers approach a text, we use our understanding of the emotional intent of words to infer whether a section of text is positive or negative, or perhaps characterized by some other more nuanced emotion like surprise or disgust. We can use the tools of text mining to approach the emotional content of text programmatically, as shown in Figure 2.1.

One way to analyze the sentiment of a text is to consider the text as a combination of its individual words and the sentiment content of the whole text as the sum of the sentiment content of the individual words. This isn’t the only way to approach sentiment analysis, but it is an often-used approach, and an approach that naturally takes advantage of the tidy tool ecosystem

## 2.1 The `sentiment` dataset
As discussed above, there are a variety of methods and dictionaries that exist for evaluating the opinion or emotion in text. The tidtext package contains severail sentiment lexicons in the several sentiment lexicons in the `sentiments` dataset.
```{r}
library(tidytext)
sentiments
```

The three greneral-purpose lexicons are
- `AFINN` from from Finn Årup Nielsen,
- `bing`Bing Liu and collaborators, and
- `nrc` from Saif Mohammad and Peter Turney.

All three of these lexicons are based on unigrams, i.e., single words. These lexicons contain many English words and the words are assigned scores for positive/negative sentiment, and also possibly emotions like joy, anger, sadness, and so forth. 

The `nrc` lexicon categorizes words in a binary fashion (“yes”/“no”) into categories of positive, negative, anger, anticipation, disgust, fear, joy, sadness, surprise, and trust. The `bing` lexicon categorizes words in a binary fashion into positive and negative categories. The `AFINN` lexicon assigns words with a score that runs between -5 and 5, with negative scores indicating negative sentiment and positive scores indicating positive sentiment. All of this information is tabulated in the sentiments dataset, and tidytext provides a function `get_sentiments()` to get specific sentiment lexicons without the columns that are not used in that lexicon.
```{r}
get_sentiments("afinn")
```

```{r}
get_sentiments("bing")
```

```{r}
get_sentiments("nrc")
```

How were these sentiment lexicons put together and validated? They were constructed via either crowdsourcing (using, for example, Amazon Mechanical Turk) or by the labor of one of the authors, and were validated using some combination of crowdsourcing again, restaurant or movie reviews, or Twitter data. 

Given this information, we may hesitate to apply these sentiment lexicons to styles of text dramatically different from what they were validated on, such as narrative fiction from 200 years ago. While it is true that using these sentiment lexicons with, for example, Jane Austen’s novels may give us less accurate results than with tweets sent by a contemporary writer, we still can measure the sentiment content for words that are shared across the lexicon and the text.

here are also some domain-specific sentiment lexicons available, constructed to be used with text from a specific content area. Section 5.3.1 explores an analysis using a sentiment lexicon specifically for finance.

*Dictionary-based methods like the ones we are discussing find the total sentiment of a piece of text by adding up the individual sentiment scores for each word in the text.

Not every English word is in the lexicons because many English words are pretty neutral. It is important to keep in mind that these methods do not take into account qualifiers before a word, such as in “no good” or “not true”; a lexicon-based method like this is based on unigrams only. For many kinds of text (like the narrative examples below), there are not sustained sections of sarcasm or negated text, so this is not an important effect. Also, we can use a tidy text approach to begin to understand what kinds of negation words are important in a given text; see Chapter 9 for an extended example of such an analysis.

One last caveat is that the size of the chunk of text that we use to add up unigram sentiment scores can have an effect on an analysis. A text the size of many paragraphs can often have positive and negative sentiment averaged out to about zero, while sentence-sized or paragraph-sized text often works better.

## 2.2 Sentiment analysis with inner join







