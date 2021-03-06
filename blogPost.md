# Author Style Transfer Using Recurrent Neural Nets (Blog Post)

First, let me note the names of the entire team that contributed to the project: Mukundan Kuthalam, Yuriy Minin, Zohaib Imam, and Ram Muthukumar.

For every paper that we mention in this post, we have a link to the paper denoted with a superscript number, so click on those if you would like to read the papers. Now if you would like a link to the GitHub with our model and datasets, look [here](https://github.com/yursky/art-of-text)

## Intro

There have been artists who try to copy others. In art class in elementary school, you might have tried to draw a picture in the style of
Van Gogh or someone like that. The art of transforming an existing image into another author's style is an idea that has already been
explored using neural networks. However, this problem is an interesting one when it comes to the realm of Natural Language Processing (NLP).
How do we rate how close author's styles are? What about the target author's text do we want to capture so that we can transform input text to match the target's style? These questions are the sort of things that we explore in our project: Author Style Transfer Using Recurrent Neural Networks.

## Prior Work

First, though, let's take a look at what has already been done. In "Applying Artistic Style Transfer to Natural Language," Edirisoorya and
Tenney[<sup>1</sup>](https://github.com/yursky/art-of-text/blob/master/papers/neural-style-natural-language.pdf) make the point that using style transfer in the realm of NLP is a relatively new technique. Here, literary text was replaced with embedding ID's and fed into a GRU based RNN for identifying the author. The network for the style transfer had an encoder and decoder that would enable them to define a loss in content and style. Before we go on, we need to define their content loss and style loss.

Their Seq2Seq model pooled (averaged) consecutive word vector inputs along with word vector outputs, found the difference between these
vectors, and labeled the result as content loss. For style loss, it was a little more complex. Suppose an image was fed into the network. Then, at every hidden layer, a vector will be generated. Now suppose we input the text whose style we want to transform. We can compare the vectors that it generates at every layer and the vectors from earlier, and create a style loss function from there. Thus, we get a cost function that is a linear combination of content loss and style loss and then create a network that minimizes this function.

The project did not work too well and there were a few possible improvements that the paper mentioned. However, we will be discussing a paper[<sup>2</sup>](https://github.com/yursky/art-of-text/blob/master/papers/style-transfer-from-non-parallel-text-by-cross-alignment.pdf) from NIPS 2017 that gets a lot better results and became the basis for our experiments. Thus, we used this paper to get a good understanding of the problem of Author Style Transfer and what a model to solve this problem would look like.

## The Data

So let's talk about where the data is coming from. Kaggle has a really nice dataset that it provided for the "Spooky Author Identification"[<sup>3</sup>](https://www.kaggle.com/c/spooky-author-identification/data) competition. The best part is that this data is clean and pre-processed very well, so we can focus on word embeddings using just this data. However, data science is all about the data, so some code was written to try and process the data that came from Project Gutenberg, and that taught us a few things about dealing with data.

Pre-processing takes a lot of effort, and it can many times come from the fact that file formatting is inconsistent across files. A text file for one book can space out paragraphs and even sentences in a completely different way than another and this made writing the pre-processing code quite a task in and of itself. Also, people generally don't think about how a piece of software will read information, so you can find random content in between paragraphs and even sentences in paragraphs can be hard to split (we thought [NLTK](http://www.nltk.org/) would work magic and more importantly, save time, but even that library had a tough time getting it right). Note for the future: Expect to spend considerable time just getting data ready.

Nonetheless, we also had the Kaggle dataset, so we had 5 authors worth of data to work with. The Dickens and Tolstoy books were thought to be processed fairly well, but turned out to not be as clean as we would have liked, while the team still has issues with Twain's works, probably because there are still encoding issues that we need to wrestle with more. Unfortunately, we could not add the Project Gutenberg dataset to the overall dataset for these reasons. Oh, and we should mention that our great professors were kind enough to allow us to transcribe their lectures, but since we could not get a large enough number of sentences from those, we could not use those for the dataset either.

So our complete dataset came from Kaggle, a corpus of Shakespeare text that we found, and a corpus of Yelp reviews that ended up giving us enough data for some interesting results. Okay, on to the model!

## The Model

Alright, here we are. It is nice to talk about the algorithms that others have tried and what we did to prepare our data, but at the end of the day, we just want to see what our model can do. But to understand that, we need to know what the model is doing, and that means a lot of math.

The general idea is that any piece of text is made up of two parts: style and content. We can also say that while content may be shared across pieces of text (they may talk about the same characters or about the same topics), each author has their own style that they apply to this content that makes their writing unique to them. This is exactly the sort of idea this model tries to exploit. That is, any piece of text can be encoded into a "content-based" representation. Then, to mimic the style of another writer, this text can be fed to a decoder that applies a "style" to the text, resulting in a piece of text that looks like it was written by a specific author.

To put this in more mathematical terms, every corpora of text is the input to an encoder that converts the corpora to its "content representation". Once this is done, the style-dependent decoder layer can covert this representation to something that looks like it was written by the author that has that particular style. Thus, there are three parts to a text: the style of the variable (let us call this variable y), the content of the variable (z), and the corpora of text itself that is generated from the distribution of y and z (this is x). Now for a discussion of the probability behind the model.

First, it is important to know that since we know the author styles (and also the corpora of text that we are analyzing), we know p(x1|y1) and p(x2|y2). However, we really need to find the style transfer functions between them \[p(x1|x2;y1,y2) and p(x2|x1;y1,y2)\]. This involves recovering the joint distribution p(x1,x2|y1,y2) which can be done by factoring in z. That is, if we know p(x1|y1,z) and p(x2|y2,z), then p(x1,x2|y1,y2) is the integration of p(x1|y1,z) * p(x2|y2,z) * p(z) over z assuming that each of the x's are generated from "distinct enough" styles (otherwise, there would naturally not be a well-defined style transfer function).

The kicker of all this is that to transfer x1 into the style of x2, we need an encoder that learned the content of x2 \[p(z|x2,y2)\] and a decoder layer that adds style to this content representation \[p(x|y,z)\]. In case you have not noticed yet, since we already know the styles of both authors, the big challenge is preserving the content of the text that we are transforming. The more content we preserve, the closer it sounds like another author wrote the same corpora of text in their words. Then the loss function becomes what parameters "theta_E" and "theta_G" must be used for E and G so that the encoder E learns the content of the target content as well and possible and G generates x as well as possible from the given style y and content z. For those who like numbers and symbols: ![alt text](https://github.com/yursky/art-of-text/blob/master/language-style-transfer/img/LossFunc.png)

This is basically where we get to the point of tuning the neural networks that were built to see if we can get the best thetas. There are two variants of this model. One called an auto-aligned encoder has a discriminator D that tries to "align" each content distribution with the right z ~ p(xi|yi) distribution (where i is some integer). By "align," I mean the content distributions that are generated are transformed to match the true content distribution of the other corpora. The second type of encoder (cross-aligned auto-encoder) looks at the corporas directly. It takes the transformed corporas and aligns those with the true values generated from the target style. This is the last layer of the network.

Phew, that's a lot of math! However, you now hopefully have a better idea of what sort of thing this model is trying to do. Now, without further ado, the next discussion talks about what was achieved by applying this model to the data.

## The Results
Well, not all Data Science projects give you fantastic results and that was painfully obvious here. Limited by time, computational resources, and data, author style transfer proved quite complex. This could have been due to a number of reasons including lack of training time and a more diverse vocabulary. Interestingly enough, the sentiment transfer using Yelp reviews gave us some really good results, where we turned negative Yelp reviews into more positive ones. Enough from me, see for yourself:

# Sentiment Transfer Results:  
```
Original: horrible service was .   
Transfer: great service !  
```

In the first example, the core subject in the original sentance (the service) is preserved in the transformed sentence. "Horrible" was transformed into "great," which shows the model is able to reverse the sentiment. An interesting thing to note in this example is the period became an exclamation point, showing the model understands the concept of enthusiasm. 

```
Original: i would never recommend this place to anyone and it 's me .  
Transfer: i will definitely recommend this place to go and everyone !  
```

Okay so maybe neither of these sentences are grammatically correct, but there are interesting results. Again, we see the model seemingly correlating exclamation marks with positive sentiment. That said, the structure of the sentence is preserved and the sentiment reversal was spot on. The next step was to try and apply the model to switching styles from horror authors (HP Lovecraft, Mary Shelley, and Edgar Allen Poe) to Shakespeare.

# Author Transfer Results:  
(Horror Authors to Shakespeare)

```
Original: the <unk> of the <unk> of the <unk> and the <unk> of the <unk> and the <unk> of the <unk>  
Transfer: And , , , , , , , , , , 
```

The data was preprocessed to omit proper nouns and numbers, hence the <unk>'s. In contrast to the model trained to transfer sentiment, the model trained on author style transfer was not very successful. In the first example above, the model understood Shakespeare loved to use commas, but the translated result preserves no content from the original. 

```
Original: i was the <unk> of the <unk>.   
Transfer: And , the king of the world  
```

The second example shows a more interesting result. The model inferred what word Shakespeare would use in place of the unknowns, and chose grandiose language such as "king" and "world," fitting language Shakespeare would use.

```
Original: i had been <unk> and i had not be   
Transfer: And , I 'll not be to be .  
```

In this third example, the transferred sentence preserves no content, but created a phrase similar to "to be or not to be," a line accreditted to Shakespeare. 

## Moving Forward  
Training a neural network takes a very long time and our team found this out the hard way. Literary authors also have quite a diverse vocabulary, which is harder for the model to learn. For these reasons, access to more time would likely improve our results. In addition to taking a while to train, the model was also a very "hungry" one. That is, it required a lot of data to train the model (the sentiment data set had 150,000 sentences!) so the more data we could feed it, the better. Due to the long runtime of the model, hyperparameter tuning (i.e. the learning rate, gamma, etc.) was somewhat ignored and that is something the team would love to look into in the future. Of course, making architectural changes is always on the table and the idea of the Siamese network is something we could try...

## Conclusion
This problem is a difficult one to solve and the models that exist today are certainly nothing short of hungry and hard to train. However, the project was great for learning about neural networks and also how the results of Data Science experiments can be, perhaps, not as desirable as the scientists would have wished. This project was quite a learning experience and we look forward to working on more things like this.

Remember, any and all hyperlinks are either the superscripted numbers or the GitHub repo link at the top. Thanks for reading!
