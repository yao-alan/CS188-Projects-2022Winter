---
layout: post
comments: true
title: Sheet Music Recognition
author: Ning Wang and Alan Yao
date: 2022-01-26
---


> Sheet Music Recognition is a difficult task. [Zaragoza et al.](URL 'https://www.mdpi.com/2076-3417/8/4/606') devised a method for recognizing monophonic scores (one staff). We extend this functionality for piano sheet music (grand staff) that have are monophonic in each staff (treble and bass).


<!--more-->
{: class="table-of-content"}
* TOC
{:toc}

## Main Content
This project was inspired by [Zaragoza et al.](URL 'https://www.mdpi.com/2076-3417/8/4/606'). We extend the monophonic score reader by parsing grand staves from piano sheet music. Thus, we add a stage in the pipeline to first identify any grand staves before separating them into treble and bass. Each individual staff is then feed into the current pipeline. 

# An End-to-End Approach to Optical Music Recognition

## What is Optical Music Recognition (OMR)?

As its name suggests, OMR is the field of research focused on training computers to automatically read music. Despite its similarities to other processes such as optical character recognition (OCR), OMR is substantially more difficult. Unlike words on a line, the spatial positioning of notes contributes to the overall semantics; for instance, in the treble clef, a note on the bottommost line in a staff is an ```E```, while a note on the line just above it is a ```G```. Furthermore, robust solutions must be able to successfully interpret the same note differently depending on clef, tempo, and a variety of other factors. Other issues include staff lines adding visual noise and hindering note classification, difficulties in finding bars used to delimit staves, and the varying sizes of musical notation (a musical slur may span multiple measures, in contrast to the tiny dots lengthening the duration of a note).

### Classical approaches

Traditional methods, not based in deep learning, break down OMR into multiple stages. Among other things, an initial pass will clean up the image by removing staff lines, find the bars that break a stave into measures, and detect the clefs that affect the interpretation of notes in pitches. Next, symbols are isolated and run through a classifier. Finally, the individual components are reassembled and reinterpreted within the overall semantics of the piece.

Unfortunately, such a piecewise approach typically fails to generalize **ELABORATE**.

## A Deep-Learning Solution

### The PrIMuS dataset

Neural networks require large amounts of data upon which to train; to facilitate learning, the authors introduced the *Printed Images of Music Staves* (PrIMuS) dataset.

In PrIMuS, 87,678 sequences of notes, written upon a single staff, are converted into five different representations, these being

1. Plaine and Easie code **insert image 1a.**
2. Rendered images **insert image 1b.**
3. Music Encoding Initiative format **insert image 1c.**
4. Semantic encoding **insert image 1d.**
5. Agnostic encoding **insert image 1e.**

Note that in the final setting, many of the rendered images are distorted using GraphicsMagick, so as to emulate pictures taken with a bad camera.

The major accomplishment of PrIMuS, besides its scale and diversity, lies in the distinction between semantic and agnostic encodings. As their names suggest, the semantic encoding includes real musical meaning that the agnostic version lacks. For example, note the two "sharp" symbols in the rendered image above. In the agnostic encoding, these are labelled as
```
accidental.sharp-L5, accidental.sharp-S3
```
that is, sharps on "line 5" (the uppermost line) and "space 3" (the third space between two lines). In a real musical context however, these two sharps denote the D-Major key signature, and are appropriately designated
```
keySignature-DM
```
in the semantic encoding.

As another example, notes that are linked together (for instance, sixteenth notes) have this visual linkage designated with `.beamed` in the agnostic representation; in the semantic version, they are correctly labelled `.sixteenth`.

In essence, the agnostic encoding merely notes the 2D plane positions of musical elements, while the semantic encoding interprets these elements in the context of the greater piece.

### A quick primer on CTC networks

Sequential data usually fails to come in set widths. As an analogy, suppose we had to split a sentence, typed in 20 pt monospace font, into fixed-width letters. This would be easy: all we have to do is box each letter in a rectangle of 20-pt width. Now imagine if we were to use these same boxes to split a second sentence, one written in 25-pt sans-serif. Suddenly, some boxes might contain just a fraction of a character; we would be completely unable to split such a sentence! **Insert example image for reference**.

How can we fix this? Note that the main problem here is the lack of alignment; a single letter might span two boxes. If we were to independently read each box without the context of the other, we would be unable to make out the greater letter. Thus we need to find some way to span contexts.

*Connectionist temporal classification* (CTC) accomplishes just this in the context of RNNs. Unlike traditional networks, CTC networks continue predicting the same token when the current context is not yet finished; for instance, in our example above, given **insert image of an A split across three boxes**, a network might learn to predict ```AAA```, because the network correctly believes that the latter two boxes still span the A.

Note that with this current system, we are unable to distinguish whether ```AAA``` corresponds to `A`, `AA`, or a true `AAA`. With CTC, the solution is to add a special "blank" character `-` that delimits tokens. Therefore given **insert image of two A's**, a CTC network might predict `A-A`.

To formalize the above, a CTC network uses an RNN to map from a $T$-length sequence $\textbf{x}$ to a $T$-length target sequence $\textbf{z}$. Each element $x_i \in \mathcal{X}$, the $i$th element of $\textbf{x}$, is an $m$-dimensional real-valued vector. Given a finite alphabet $\Sigma$, consisting of all possible token labels, $z_i \in \Sigma^* = \Sigma \cup \{\text{blank}\}$. Each output unit in the network calculates the probability $y_k^t$ of observing the corresponding label at time $t$; as it is assumed all such probabilities are independent among the $T$ indices,
then given an output sequence $\pi$, the probability $p$ of getting output sequence $\pi$ given input sequence $\textbf{x}$ is
$$
p(\pi \mid \textbf{x}) = \prod_{t=1}^T y_{\pi_t}^t.
$$
All blanks and duplicates are removed after final processing, so sequences such as ```--lll-m--aa-o``` and ```lm-ao``` both map to ```lmao```. Therefore the probability of getting a final labelling $\textbf{l}$ is
$$
p(\textbf{l}\mid \textbf{x}) = \sum_{\pi \in \mathcal{B}^{-1}(\textbf{l})}p(\pi \mid \textbf{x}).
$$
The predicted sequence is that with the highest probability, or
$$
\arg \max_{\textbf{l} \in L^{\leq T}} p(\textbf{l} \mid \textbf{x}).
$$

### The deep-learning approach

Since music is a prime example of sequential data without fixed widths, the authors have chosen to create a *Convolutional Recurrent Neural Network* (CRNN) leveraging CTC loss. To simplify the process, only single staffs are run through the network (so, for instance, the network cannot simultaneously process the two staffs---one for each hand---of a piano piece).

The model structure is as follows:
**Insert Figure 4 of main paper**.

Specifically, we have
<center>

| Input ($128 \times W \times 1$)                |
| :----:                                         |
| **Convolutional Block**                        |
| Conv($32, 3 \times 3$),  MaxPool($2 \times 2$) |
| Conv($64, 3 \times 3$),  MaxPool($2 \times 2$) |
| Conv($128, 3 \times 3$), MaxPool($2 \times 2$) |
| Conv($256, 3 \times 3$), MaxPool($2 \times 2$) |
| **Recurrent Block**                            |
| Bidirectional LSTM($256$)                      |
| Bidirectional LSTM($256$)                      |
| Linear($\|\Sigma\| + 1$)                       |
| Softmax                                        |

</center>

where the input is given in $(h, w, c)$ format. Batch normalization is used for every convolutional layer, and the outputs of said layers are passed through a standard ReLU. Note that the linear layer has $|\Sigma| + 1$ output units because of the extra blank label needed with CTC. Bidirectional LSTMs are used because later information about notes can help inform previous frames.

In training, mini-batches of 16 samples and the Adadelta learning rate optimizer were used. While there are better ways to decode the final sequence (i.e. beam search), a simple greedy decoding is used.

### Results



## Improving Upon the Paper

### Multi-staff detection with YOLOv3


## Basic Syntax
### Image
Please create a folder with the name of your team id under /assets/images/, put all your images into the folder and reference the images in your main content.

You can add an image to your survey like this:
![YOLO]({{ '/assets/images/UCLAdeepvision/object_detection.png' | relative_url }})
{: style="width: 400px; max-width: 100%;"}
*Fig 1. YOLO: An object detection method in computer vision* [1].

Please cite the image if it is taken from other people's work.


### Table
Here is an example for creating tables, including alignment syntax.

|             | column 1    |  column 2     |
| :---        |    :----:   |          ---: |
| row1        | Text        | Text          |
| row2        | Text        | Text          |



### Code Block
```
# This is a sample code block
import torch
print (torch.__version__)
```


### Formula
Please use latex to generate formulas, such as:

$$
\tilde{\mathbf{z}}^{(t)}_i = \frac{\alpha \tilde{\mathbf{z}}^{(t-1)}_i + (1-\alpha) \mathbf{z}_i}{1-\alpha^t}
$$

or you can write in-text formula $$y = wx + b$$.

### More Markdown Syntax
You can find more Markdown syntax at [this page](https://www.markdownguide.org/basic-syntax/).

## Reference
[1] Calvo-Zaragoza, J., Rizo, D.: End-to-end neural optical music recognition of monophonic scores. Appl. Sci. 8(4), 606 (2018)

[2] Calvo-Zaragoza, J., Rizo, D.: Camera-PrIMuS: neural end-to-end optical music recognition on realistic monophonic scores. In: Proceedings of the 19th International Society for Music Information Retrieval Conference, ISMIR 2018, Paris, France, 23–27 September 2018, pp. 248–255 (2018)

[3] Alfaro-Contreras M., Calvo-Zaragoza J., Iñesta J.M. (2019) Approaching End-to-End Optical Music Recognition for Homophonic Scores. In: Morales A., Fierrez J., Sánchez J., Ribeiro B. (eds) Pattern Recognition and Image Analysis. IbPRIA 2019. Lecture Notes in Computer Science, vol 11868. Springer, Cham.


---
