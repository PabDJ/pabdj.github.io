---
title: "Generation and Classification of Illicit Bitcoin Transactions - Behind the scenes"
date: 2023-02-15T21:45:58+05:30
description: ""
tags: [master thesis, conference paper, artificial intelligence, bitcoin, data imbalance, deep learning, generative adversarial networks, long short-term memory networks]
---

Some weeks ago I had the pleasure to be invited to [*Un podcast sobre Bitcoin*](https://www.unpodcastsobrebitcoin.com/) (A podcast about Bitcoin, for those who do not speak Spanish) which is directed by [Alberto Mera](https://twitter.com/alberto_mera).

{{< youtube 4CIVYVWDn40 >}}

During the episode I explained the whole process of this work but as I have not done it in my blog yet, it is a good opportunity to do it.

## How it started
A year ago, that is January 2022, I was finishing the first semester of my master's. As any other master, the last step is to write a master thesis, but I did not have a specific topic to work in.

Fortunately, students do not have to come up with a topic by themselves, and we are offered a list of different ones. The challenge at that moment was to find something interesting for me and to check if it was available. After making a list and thinking about the different possible outputs of each one I ended up choosing a thesis that covered Bitcoin cybercrime and artificial intelligence (AI).

Before deciding the topic, I had an interview with [Pedro Peris](https://lightweightcryptography.com/), in which I told him about my background.
I was completely honest, as my expertise in AI was null but on the other hand I was pretty familiar with Blockchain. At the end of the day I worked 8 months as an Embedded System Engineer in the development of a [hardware security module for BLS signatures](https://github.com/decentralizedsecurity/bls-hsm) in the new Ethereum Proof of Stake network. Apart from that, this work was also my bachelor's thesis, so I had to document myself a lot about Bitcoin and the whole Blockchain ecosystem.

## The work itself
### Research
How can you work on something if you do not know what has been done? Well, in order to fix this, the best way is to read, and read, and read (again) tons of papers in order to understand the *state of the art*. I am not going to lie, it is not the most entertaining task that you can do, but it gives you a precise picture of the landscape. You can observe what problems have been studied, what solutions have been proposed and what is expected to be done in the future.

I would like to mention that during this process you will be frustrated. It is not something weird, in fact, I think that most researchers have experienced this feeling. Why? I will illustrate the answer with the following tweet:

{{< tweet 1613145342681153536 >}}

As you can observe in the highlighted area, people **DO NOT SHARE THEIR CODE AND/OR DATASET**. This is sad. Really sad. First, because you cannot reproduce other people results, and second, because it avoids enhancement of previous solutions.

So, if we take into account how academia works, in general, and the lack of datasets on this topic (cybercrime in Bitcoin) we will feel overwhelmed...

## Overcoming the challenges
As I mentioned before, the lack of datasets was a big problem that I had to face. It is a key part in the pipeline of AI. Fortunately, I was able to collect some such as:
- [Elliptic Data Set](https://www.kaggle.com/datasets/ellipticco/elliptic-data-set) - 200K bitcoin transactions with 166 features (only 4,545 were illicit)
- [Ransomware in the Bitcoin Ecosystem | Dataset Extraction](https://github.com/behas/ransomware-dataset)- 7,222 Bitcoin seed addresses
- [On the Economic Significance of Ransomware Campaigns: A Bitcoin Transactions Perspective](https://spritz.math.unipd.it/datasets/btcransomware/knowledge_base.zip) - 4021 addresses
- [Bitcoin Ponzi Schemes Addresses](https://github.com/seanconeys/Bitcoin_Ponzi_ml/blob/master/datasets/final_aggregated_dataset.csv) - 2300 addresses
- [Data mining for detecting Bitcoin Ponzi schemes](https://docs.google.com/spreadsheets/d/e/2PACX-1vS8JHql9CP_XWCfJ9VG-9VXDTizHPX1wSQBwnsg9QDasPypmfelpfc_9uIXk73vaMHXlLGRk2veYXFk/pubhtml#)- 52 addresses

If we sum all together I collected more than 13,500 addresses related to illicit activity, which can be found [here](https://github.com/PabDJ/IllicitBitcoinTransactions/blob/main/suspicious_addresses.txt).

Elliptic Data Set was the result of a collaboration between MIT, IBM and Elliptic. They produced the paper entitled [*Anti-Money Laundering in Bitcoin: Experimenting with Graph Convolutional Networks for Financial Forensics*](https://arxiv.org/pdf/1908.02591.pdf) and the data set that I have previously mentioned. 

After several weeks of research, I was satisfied with everything I collected although there were some obstacles on the road. The most important one was that I had a large data set but only **23 % of the data was labelled**, and only **2 % were illicit**, which was the class of interest. How can I work with that?!

Another thing that caught my attention was the meaning of the features. They were not disclosed and the only information available was that the first 94 columns represent local information about the transaction and the remaining 72 features are aggregated features, obtained using transaction information one-hop backward/forward from the center node.

The first approach was writing the authors in order to know what was exactly behind each feature. The answer that I received was that due to intellectual property reasons the features could not be disclosed. **That hit hard.**
The second approach was writing them again, but in this case through my Master thesis supervisor, Pedro Peris. In this case we did not even receive a reply. Time to move on.

As I was not very familiar with Kaggle, I found after a while that another user deanonymized **99.5 %** of the transactions in the data set. The explanation was written [in this Russian forum,](https://habr.com/ru/post/479178/) but he did not focus on reverse-engineering the features, so I was still on the same position.
Well... not really, in fact, as the transaction hash was unveiled I could look at the date from those transactions. It is not a small thing because we only knew that the data set was spaced in 49 timespans and each of them corresponded to a period 2 two weeks. After looking at the deanonymized transactions I found out that the first transaction in the data set took place in January 1st, 2016, and the last transaction in October 2nd, 2017.

## Generation of data
Also, do you recall the addresses that were linked to illicit activity? What could I do with that data? In order to obtain valuable information that could contribute to my research, what I did was writing a [Python script](https://github.com/PabDJ/IllicitBitcoinTransactions/blob/main/get_tx_from_address.ipynb) that retrieved all the transactions from those suspicious addresses in the period of time that we were interested in (January 1st, 2016 - October 2nd, 2017). After this task, we ended up with nearly 25,000 Bitcoin transactions linked to malicious activity! To put that into numbers, we were providing **5x the former amount of illicit transactions** in the Elliptic data set.

Anyone who has dealt with artificial intelligence knows how important is to have a balanced data set. We made huge progress with the discovery of new illicit transactions, reducing the class imbalance. **The original ratio was 9.8:90.2 illicit/licit ratio. Our work changed that to a 41.2:58.8 illicit/licit ratio.** But, what is even better is to fight class imbalance through AI.

In our case, it was not an easy task. If you remember, we were working with time series data, so we could not apply traditional solutions like the ones that have been studied with data sets made by images, where simple transformations to the data like cropping, rotation, etc., generates a high amount of them. We used *Generative Adversarial Networks*, also known as GAN. This type of neural network is composed by two opposed entities, a *generator* and a *discriminator*. The final goal of the network is to generate high-quality data. At first, the generator is fed with some samples of the data set and tries to mimic it by producing its own samples. On the other side, the discriminator is taught what real data is and tries to decide whether the information sent by the generator is real or fake. After several iterations and parameter tuning, you will end up with data that looks similar to the real one. In our case, we generated 24,947 artificial samples. This number corresponds to the same amount of real illicit transactions collected previously.

## Classification of data
We have already covered the *Generation* part of our research, first with new 25,000 illicit and real transactions, and second with the study and application of GAN to our research. It is time to move to the *Classification* part. The paper written by Elliptic covered both machine learning (ML) and deep learning (DL). We also followed the same structure.

### Machine Learning
First of all, regarding ML algorithms, we experimented with Random Forest (RF) and Logistic Regression (LR) but with the balanced data sets, the one containing real transactions and the one containing artificial samples. Both experiments outperformed Elliptic results, highlighting the importance of working with more balanced data sets to reduce the bias.

| Method                   | Precision |  Recall   |    F1     |
|--------------------------|:---------:|:---------:|:---------:|
| Elliptic RF              |   0.956   |   0.670   |   0.788   |
| **RF with natural tx**   | **0.985** | **0.962** | **0.974** |
| **RF with synthetic tx** | **0.999** | **0.983** | **0.991** |
| Elliptic LR              |   0.404   |   0.593   |   0.481   |
| **LR with natural tx**   | **0.784** | **0.824** | **0.804** |
| **LR with synthetic tx** | **0.951** | **0.961** | **0.956** |

### Deep Learning
Secondly, we also wanted to study DL techniques for classification. The original research was focused on *Graph Convolutional Networks* (GCN) and some variations of it. We thought that the classification of illicit transactions could be improved by applying *Long-Short Term Memory Networks* (LSTM). The following table shows the results obtained by Elliptic with GCN and the next one the results that we obtained with LSTM. Again, we outperformed Elliptic's results.

| Method                      | Precision |  Recall   |    F1     |
|-----------------------------|:---------:|:---------:|:---------:|
| Elliptic GCN                |   0.812   |   0.512   |   0.628   |
| Elliptic Skip-GCN           |   0.812   |   0.623   |   0.705   |
| Elliptic EvolveGCN          |   0.850   |   0.624   |   0.720   |
| **LSTM with Elliptic data** | **0.908** | **0.855** | **0.868** |
| **LSTM with natural tx**    | **0.947** | **0.927** | **0.934** |
| **LSTM with synthetic tx**  | **0.991** | **0.981** | **0.985** |
-
{{< callout text="❗️For a better visual comparison, our results are highlighted in bold" >}}

## Conclusions
This research will always hold a special place in my heart. Firstly because it was my Master thesis, and it ended up being my first publication, as a conference paper. But secondly, because I was able to learn new skills like artificial intelligence (ML and DL) and at the same time I proved to myself that an unknown path can have challenges, but they can be defeated and reaching the goal will always be something satisfying.