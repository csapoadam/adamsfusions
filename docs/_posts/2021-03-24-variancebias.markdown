---
layout: post
title: Bias! Variance! Lower thyselves!
date: 2021-03-24 19:20:23 +0900
category: ML
---

As we will see in this post, mathematical thinking can at times be close to philosophical thinking. The starting point and direction from which you engage with a problem matters a lot, as does your interpretation of the results you obtain.

In this post, we will consider the derivation and interpretation of the role of variance and bias in the error of a machine learning model.

# Different formulations of the error we want to minimize

Let's say we have a distribution $$ \mathbf{P} $$. Input-output value pairs belonging to the problem domain
can be sampled from this distribution: $$ (\mathbf{x}_i, y_i) \sim \mathbf{P} $$.

Also, suppose that we have a dataset that we obtained by taking $$ N $$ samples from this distribution $$\mathbf{P}$$. Let's call the dataset $$ \mathcal{X} $$.

Now, assuming that we are working on a regression problem, let's say we want to express the error of our model, $$ y = f(x) $$ in terms of some distance between a desired and obtained output in general. How do we go about doing this?

Well, based on the above framework of thinking, there are at least two ways:

1. If we wanted to express the error in general over all possible input values, we could express it as $$ Error = E_{\mathbf{P}}\left[(y_i - f(\mathbf{x}_i))\right] $$ (that's assuming that we want to use the mean squared error as our distance metric). Let's call this the **model-dependent error**, or **MDE** (bear with me for a moment before we get to why this name makes sense).

2. If we wanted to express the error in a way that focuses on the influence of the training samples we used to train the model (as opposed to other alternative training sets), we could write $$ Error = E_{\mathcal{X}}\left[(y - f_{\mathcal{X}}(\mathbf{x}))\right] $$. We can call this the **training set independent error**, or **TSIE**.

As we will see, the distinction between these two is an important one: in the first case, we are taking the weighted sum of the expression inside the brackets over all possible input-output values pairs, while considering the model, $$ f $$ to be fixed; while in the second case, we have a single input-output pair ($$ x $$ and $$ y $$), but we are taking the weighted sum of the expression over all possible models obtained via different training sets.

In the following, let's briefly focus on how the training set independent error (TSIE) can be conceived of.

# Training Set Independent Error

Let's begin by spelling out what is meant by the term expected value:

$$
\begin{align}
  TSIE &= E_{\mathcal{X}} \left[ (y - f_{\mathcal{X}}(\mathbf{x}))^2 \right] = \\
  &= \sum\limits_{i=1}^{I} p_i \left[ y - f_{\mathcal{X}_i}(\mathbf{x})) \right]^2
\end{align}
$$

where $$ f(\mathbf{x}) $$ is the output of the trained model for input $$ \mathbf{x} $$, which we will also denote as $$ \hat{y} $$; $$ y $$ is the expected ouput, and $$ p_i $$ refers to the probability of the $$ i^{th} $$ dataset $$ \mathcal{X}_i $$, supposing that we can train on $$ I $$ different datasets (if the probability distribution $$ \mathbf{P} $$ were continuous, we could use integration instead of summation).

Note that in the equation above, I've added $$ \mathcal{X}_i $$ to the subscript of $$ f $$ to show that this model will be dependent on the dataset on which it was trained -- but we can leave this subscript out for the sake of brevity from now on. As a matter of fact, we'll just be writing $$ \hat{y} $$ instead.


Now, let's transform this expression a bit, to glean some new insights:

$$ \begin{align}
TSIE &= \sum\limits_{i=1}^{I} p_i \left[ y - \hat{y}) \right]^2 = \\
&= \sum\limits_{i=1}^{I} p_i \left[ y - E(\hat{y}) + E(\hat{y}) - \hat{y}) \right]^2
\end{align}
$$

Here we have both subtracted and added the expected value of the output of our model for the given input (representing the average output of the model, if all possible training data sets are considered). By adding and subtracting this value (which, by the way, is a constant scalar value), there is no change to the overall expectation. Now we can see that we have an expression of the form $$ \sum_i p_i (a + b)^2 $$. Hence, using the classic expansion of $$ (a + b)^2$$, we can continue the derivation as follows:

$$ \begin{align}
TSIE &= \sum\limits_{i=1}^{I} p_i \left[ (y - E(\hat{y})) + (E(\hat{y}) - \hat{y}) \right]^2 = \\
&= \sum\limits_{i=1}^{I} p_i \left[y - E(\hat{y})\right]^2 + \\
&+ \sum\limits_{i=1}^{I} p_i \left[E(\hat{y}) - \hat{y}\right]^2 +\\
&+ 2\sum\limits_{i=1}^{I} p_i (y - E(\hat{y})) (E(\hat{y}) - \hat{y})
\end{align}
$$

(We could do this because the expected value of a sum is the sum of expected values.)

### Dissecting the three terms above
1. First, let's take a look at the first term. This is the expected value of: the square of the difference between the desired output and the expected value of the output of our model (for this particular input, $$ \mathbf{x} $$). But what distribution are we conditioning this expected value on? On the distribution of training sets.
Needless to say, the squared difference in the expression does not depend on the training set we are using at all, for two reasons:
  * since we are looking at a particular case (by having fixed the input, and hence the expected output at the beginning, regardless of the training dataset), $$ y$$ does not really depend on the probability distribution over training datasets -- it is just a random output value corresponding to a random input value.
  * If we were considering $$ \hat{y} $$ instead of $$ E(\hat{y}) $$ as the second operand, then the choice of training dataset would have an effect on this expression -- but as it is in this case, we are just dealing with a constant value (i.e., the weighted average of all model outputs to this particular input, $$ f(\mathbf{x}) $$ considering all models trained across different potential datasets).
Therefore, the expression $$ \sum\limits_{i=1}^{I} p_i \left[y - E(\hat{y})\right]^2 $$ is actually equal to $$ \left[y - E(\hat{y})\right]^2 $$. This, in turn, is referred to as the **squared bias** of our model. Simply put, the bias expresses the distance between the desired output and the average output of the model for the given input.

2. Second, the expected value taken in the second term is truly the expected value of a random variable: $$ \hat{y} $$ really does depend on the training set used to train our model. In this case, the expected value represents a variance term -- the variance of the output of the model for the given input, computed over the probability distribution of the different training sets. This term will be large if, depending on the training set that is used, the model will report very different outputs to the same input.

3. Finally, the last term, $$ 2\sum\limits_{i=1}^{I} p_i (y - E(\hat{y})) (E(\hat{y}) - \hat{y}) $$ is equal to 0. Recall that the first factor, $$ y - E(\hat{y}) $$ is independent of the distribution we are looking at, so it figures as a constant in this expression and can be moved to before the summation:
$$
2(y - E(\hat{y})) \sum\limits_{i=1}^{I} p_i (E(\hat{y}) - \hat{y})
$$
Now, what remains within the summation is clearly $$ 0 $$, since the expected value of the difference between a variable and its expected value will always be 0 (this is also known as the [first central moment](https://en.wikipedia.org/wiki/Central_moment))

# Why is there a tradeoff between bias and variance?

So, the only way in which we can effectively reduce the training set independent error of our model (by influencing the model) is to minimize either the bias or variance of the model -- or, if possible, both.

To see whether there is a tradeoff, let's consider what it would mean to reduce the variance and bias of a model.

Reducing the variance of a model would mean that regardless of the training set that is used, there is almost no variation in the output of the model given a specific input. In the extreme case, 0 variance would mean that the result would be the same, regardless of the training data set. Although this is an exaggeration, it shows that the variance of a model is low when its performance is not very sensitive to the training data.

However, if this is the case, then what happens to the bias, in general? Since the performance of the model in this case is not very sensitive to the training data, there is no real way to improve its performance (i.e. to reduce the difference between the desired output and the averaged output of the model for the given input) by getting better training data.

From a different perspective, the tradeoff between variance and bias can be understood based on these two statments:

> I, bias, want to lower myself; thus I am concerned with listening to the data 100%, so that if I get a good training sample (which reflects the actual distribution behind the problem domain), I can get as close to the desired output as possible. I don't have fixed ideas etched in stone, I just want to listen to the data.

> I, variance, want to lower myself; thus I am focused on producing behaviors that are not so dependent on the data. I want to produce behaviors that would be basically the same even if the training samples were quite different.

So, in general, the mechanisms that can reduce variance and reduce bias are at odds, because reducing variance entails reducing (training data dependent) model flexibility, whereas to reduce the bias of the model, we actually want to make the model more flexible (i.e. less biased). The colloquial meaning of the term 'bias' and the relationship between bias and model flexibility have clear parallels.

