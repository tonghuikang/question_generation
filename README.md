# Implementation comments from huikang

Following the full instructuions I could run this on Ubuntu on Google Cloud Platform.

## Addtional steps

Install docker
`sudo curl -sSL https://get.docker.com/ | sh`

#### To generate texts on demand

After successfully `sudo ./setup`, reinitialise the docker: 
```
sudo docker run -d --name corenlp -p 9000:9000 sld3/corenlp:3.6.0
opennmt_run_cmd="cd /root/opennmt && th tools/translation_server.lua \
  -host 0.0.0.0 -port 5556  -model /root/data/model.t7 -beam_size 12"
sudo docker run -d --name opennmt -it -p 5556:5556 -v $(pwd)/data:/root/data sld3/opennmt:780979ab_distro20e52377 bash -c "$opennmt_run_cmd"
```

If the docker is already reinitalised,
```
sudo docker start corenlp
sudo docker start opennmt
```

Process your input
```
P=$(<input.txt)
./get_qnas "$P"
```

Next time you login

#### Potential issues

- Permission denied? `sudo ...`
- Is `sudo` needed in `pip install ...` or `pip3 install ...`?
- Python2 or Python3? This may be one reason that you your script does not work.
- `The container name ... is already in use by container ... `? Run `sudo docker start ...` instead 

## My understanding of the approach
Overview: The sentence is parsed into a suitable format and is "translated" into a question.

#### CoreNLP
CoreNLP returns such a json object:
```
{'sentences': [{'index': 0,
                'parse': 'SENTENCE_SKIPPED_OR_UNPARSABLE',  # not sure what this meant
                'tokens': [{'after': ' ',                   # ???
                            'before': '',                   # ??? 
                            'characterOffsetBegin': 0,      # starting character
                            'characterOffsetEnd': 9,        # ending character, inclusive
                            'index': 1,                     # token number, starts with one
                            'lemma': 'Singapore',           # root word
                            'ner': 'LOCATION',              # entity detected
                            'originalText': 'Singapore',    # 
                            'pos': 'NNP',                   # proper noun
                            'word': 'Singapore'},           # ?
                           {'after': ' ',                   # 
                            'before': ' ',                  #
                            'characterOffsetBegin': 10,     #
                            'characterOffsetEnd': 13,       #
                            'index': 2,                     #
                            'lemma': 'be',                  #
                            'ner': 'O',                     # no entity detected
                            'originalText': 'was',          # root word
                            'pos': 'VBD',                   # verb past tense
                            'word': 'was'},                 # 
                            ...
```

#### Conversion into `opennmt` format
`convert_text_to_opennmt_format.py` translates the CoreNLP into a form suitable for training: 
```
singapore￨B￨UP￨NNP￨LOCATION was￨O￨LOW￨VBD￨O declared￨O￨LOW￨VBN￨O independent￨O￨LOW￨JJ￨O on￨O￨LOW￨IN￨O 9￨O￨LOW￨CD￨DATE august￨O￨UP￨NNP￨DATE 1965￨O￨LOW￨CD￨DATE .￨O￨LOW￨.￨O
singapore￨O￨UP￨NNP￨LOCATION was￨O￨LOW￨VBD￨O declared￨O￨LOW￨VBN￨O independent￨O￨LOW￨JJ￨O on￨O￨LOW￨IN￨O 9￨B￨LOW￨CD￨DATE august￨I￨UP￨NNP￨DATE 1965￨I￨LOW￨CD￨DATE .￨O￨LOW￨.￨O
```
One line is generated for one possibble answer. In the above sentence there are two answers - `singapore` and `9 august 1965`.

The five elements refer to (in order):
```
token: word in lowercase
ans_tag: answer tag, B is for the first word/token, I for the subsequent tokens
case_tag: was it capitalised?
pos_tag: part-of-speech
ner: entity tag, for those in data/ner_features
```
The above result is passed into opennmt. Probably in the docker there is some additional custom processing.

#### `opennmt` output
This is the output from opennmt:
```
[[{'attn': (an mxn array),                 # not sure how the dimensions is derived
   'n_best': 1,                            # ?
   'pred_score': -4.4853830337524,         # "some score on the confidence of the translation?"
   'src': 'singapore￨B￨UP￨NNP￨LOCATION was￨O￨LOW￨VBD￨O declared￨O￨LOW￨VBN￨O '
          'independent￨O￨LOW￨JJ￨O on￨O￨LOW￨IN￨O 9￨O￨LOW￨CD￨DATE '
          'august￨O￨UP￨NNP￨DATE 1965￨O￨LOW￨CD￨DATE .￨O￨LOW￨.￨O',
   'tgt': 'who was declared independent on august august ?'   # result of the "translation"
   }],
   ...
```

The answer is re-extracted from `src`, the question is obtained from `tgt` and score is obtained from `pred_score`.

# Description

It is a question-generator model. It takes text and an answer as input
and outputs a question.

Question generator model trained in seq2seq setup by using http://opennmt.net.

# Environment

- Docker ver. 17.03+:

   - Ubuntu: https://docs.docker.com/engine/installation/linux/ubuntu/#install-using-the-repository
   - Mac: https://download.docker.com/mac/stable/Docker.dmg

- Docker-compose ver. 1.13.0+: https://docs.docker.com/compose/install/
- Python 3
- pyzmq dependencies: Ubuntu `sudo apt-get install libzmq3-dev` or for Mac `brew install zeromq --with-libpgm`

# Setup

- run `./setup`.
This script downloads torch question generation model,
installs python requirements, pulls docker images and runs
opennmt and corenlp servers.


# Usage

`./get_qnas "<text>"` - takes as input text and outputs tsv.
- First column is a question,
- second column is an answer,
- third column is a score.

## Example

Input:

```
./get_qnas "Waiting had its world premiere at the \
  Dubai International Film Festival on 11 December 2015 to positive reviews \
  from critics. It was also screened at the closing gala of the London Asian \
  Film Festival, where Menon won the Best Director Award."
```

Output:

```
who won the best director award ? menon -2.38472032547
when was the location premiere ?  11 december 2015  -6.1178450584412
```


# Notes

- First model feeding may take a long time because of CoreNLP modules loading.
- Do not forget to install pyzmq dependencies.
