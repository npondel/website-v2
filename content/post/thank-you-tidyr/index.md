---
# Documentation: https://wowchemy.com/docs/managing-content/

title: "Thank You TidyR"
subtitle: ""
summary: "A niche problem that a newer version of TidyR solved for me"
authors: [nick]
categories: [R]
date: 2020-08-21T00:00:00-05:00
lastmod: 2020-08-22T00:00:00-05:00
featured: false
draft: false

---

While I was working on my MPH program, I had to solve a problem  which at the time seemed like a complex task.  Combining and transforming some "messy" data with multiple rows per encounter into a tidy dataset for NLP analysis.   

```
messy.data
##   id                   text
## 1  1                     hi
## 2  1              some data
## 3  2                woo hoo
## 4  2 more character strings
## 5  3                help me
## 6  4                  tidyr

as.data.frame(fixed.data)
##   id                            name
## 1  1                   hi, some data
## 2  2 woo hoo, more character strings
## 3  3                         help me
## 4  4                           tidyr
```

Because I had about 3 weeks worth of experience doing "real" R work by then, I went about doing this in just about the hackiest fashion possible: hijacking some functions from [quanteda](https://quanteda.io) and using dplyr to mash the data into what I wanted.

The code looked something like this...  

```
narrative_raw <- read.csv("path/to/data", header = TRUE)

#new dataframe with only needed columns
narrative <- data.frame("KEY_CASE" = narrative_raw$KEY_CASE,
                         "text" = narrative_raw$COMMENTTEXT)

narrative$KEY_CASE <- as.character(narrative$KEY_CASE)
narrative$text <- as.character(narrative$text)

#create corpus from data, thus creating duplicate IDs
corpus1 <- corpus(narrative, docid_field = "KEY_CASE", text_field = "text")

#pull out IDs and text as separate dataframes then bind columns
text_df <- data.frame(texts(corpus1))
docid_df <- data.frame(docnames(corpus1))
duplicate_id_df <- bind_cols(docid_df, text_df)

duplicate_id_df$docnames.corpus1. <- as.character(duplicate_id_df$docnames.corpus1.)
duplicate_id_df$texts.corpus1. <- as.character(duplicate_id_df$texts.corpus1.)

#pull out extra numbers to another column
duplicate_id_df$occurances <- gsub(".*}", "", duplicate_id_df$docnames.corpus1.)

#remove periods from occurances column
duplicate_id_df$occurances <- gsub("[.]*", "", duplicate_id_df$occurances)

#remove extra numbers from ID column
duplicate_id_df$docnames.corpus1. <- gsub("(}).*","\\1",duplicate_id_df$docnames.corpus1.)

#define occurances as a numeric variable
duplicate_id_df$occurances <- as.numeric(duplicate_id_df$occurances)

#replace all blank fields with a 0
duplicate_id_df$occurances[is.na(duplicate_id_df$occurances)] <- 0

#reshaping long to wide
shape1 <- reshape(duplicate_id_df, timevar = "occurances", idvar = "docnames.corpus1.", direction = "wide")

#set NA values to blank to avoid strings of NA characters later
shape1[is.na(shape1)] <- ""

#unite columns from reshape
shape1 <- unite(shape1, combined_text, remove = TRUE, -1, sep = "  ")

#rename columns to originals
names(shape1)[names(shape1) == "docnames.corpus1."] <- "KEY_CASE"
names(shape1)[names(shape1) == "combined_text"] <- "narrative_text"
```

Gross.

This morning I ran into THE EXACT SAME PROBLEM, remembered I had done this in grad school, took one look at my old code, and knew there HAD to be a more efficient method.

Well of course there was, thanks to the pivot_wider function in [tidyr v1.0.0](https://github.com/tidyverse/tidyr/releases/tag/v1.0.0).

Time to accomplish this in 6 lines of code instead of 40…
```
# pivot_wider mutates the "long" data into a "wide" structure
# No more duplicate IDs!
messy.data %>%
  pivot_wider(
    names_from = text,
    values_from = text
  ) 

# adding a "unite" at the end collapses all columns except #1 into a single text column
messy.data %>%
  pivot_wider(
    names_from = text,
    values_from = text
  ) %>%
  unite("name", sep = ", ", -1, na.rm = T)
```

Documenting this because:  
1. It's amazing what enough time banging your head against a problem will do  
2. I'll probably run into this problem again and have forgotten how to do it

### Update 08/22/20 - "Warning: Values in 'val' are not uniquely identified..."
Ok so hitting this warning, it looks like there's an easy way to just assign row IDs and then remove them to allow the pivot to work without creating any annoying list-columns.
```
messy.data %>%
  group_by(text) %>%
  mutate(row = row_number()) %>% # assign row IDs
  pivot_wider(
    names_from = text,
    values_from = text
  ) %>%
  select(-row) %>% # remove row IDs
  unite("name", sep = ", ", -1, na.rm = T)
```