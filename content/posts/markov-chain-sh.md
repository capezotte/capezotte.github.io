---
title: "Random Sentence Generator in Shell Script"
date: 2022-05-20T17:48:09-03:00
draft: false
---

On a Telegram chat I'm in, someone mentioned in passing "maybe we should write an AI bot that learns from our group messages". As I was bored enough, I went to the best tool for generating nonsensical, but funny and somehow fitting random messages: Markov chains.

You've likely already heard of them, it's got an initial "state", be it word, letter or color, and a probability of going to a number of other "states". For chatbots, these "states" are usually words, and going from one state to the next generates a sentence. A more accurate, but filled with jargon, description can be found on [Wikipedia](https://en.wikipedia.org/wiki/Markov_chain).

# Word pairs with probability (kind of)

For generating the word pairs, I used the following script:

```sh
#!/bin/sh
awk -F '[][[:blank:])(}{,:;.?!^$*"\\\\]+' '{
	print " " tolower($1)
	for (i = 1; i < NF; i++) {
		print tolower($i), tolower($(i+1))
	}
	print tolower($i)
}' | sed 's/[[:blank:]]*$//;/^$/d'
```

The logic is, the first word of a sentence is printed, with a leading space. After that, it iterates through each `awk` field (in this case, everything that's surrounded by spaces, punctuation and regex metacharacters), printing it and its neighbor. Finally, the last word of the sentence is printed, without any spaces. The final `sed` deletes anything involving empty fields `awk` might have found.

Therefore, this scripts generates a "database" of word pairs, so looking for lines starting with a given word and a space (`grep "^$WORD "`) and filtering them through `shuf -n1` is enough.

To generate the database, just write, for instance, `./mkdb < src.txt > src.db`. Appending to this file also works.

If it feels familiar, it's basically the same format as [this older article exploring the same idea](https://0x0f0f0f.github.io/blog/markov-bot/).

I've implemented a few additional things:
- Words that begin sentences are recorded (with a initial space).
- It does remove punctuation marks.
- For easier usage within `grep`, any regex metacharacter.
- Everything is lowercased so case sensitivity is not an issue.

# Assembling the sentences

Now the only thing that's left is taking these probabilities and assembling sentences out of them. With this format, all that is needed to get a valid word pair for a given word, with weighted probability, is:

```sh
read -r WORD1 WORD2 <<<"$(grep -- "^${WORD1} " file.db | shuf -n1)"
```

With the format we've made earlier, if `WORD2` is empty, it means we've picked a sentence ending.

And, if you're not sure about which word to start a sentence, you can pick one with

```sh
# grab a line beginning with a space, i.e. a word that started a sentence,
# and remove leading space.
word=$(grep '^ ' file.db | sed 's/^ //')
# bonus points
word=$(grep '^ ' file.db) && word=${word#' '}
```

Combining these two principles, we can write this script:

```sh
#!/bin/sh
DB="${1?No DB.}"
WORD="$2"

if [ -z "$WORD" ]; then
	WORD=$(grep -- '^ ' "$DB" | shuf -n1)
	WORD=${WORD#' '}
fi

# The paste at the end will replace every newline with a space except the last.
while [ "$WORD" ]; do
	echo "$WORD"
	# The second element of the pair will replace the WORD variable.
	# If it's empty, the loop stops.
	read -r _ WORD <<EOF
$(grep -- "^$WORD " "$DB" | shuf -n1)
EOF
done | paste -sd' '
```

So there it is, we've just written a random sentence generator with just standard command line tools, and it's not even that long. Have fun generating weird sentences.

```plain
$ ./mksentence markov-chain.sh.md.db
if you're not even that long have fun generating the logic is taking these states are usually words that it feels familiar it's got an issue
```
