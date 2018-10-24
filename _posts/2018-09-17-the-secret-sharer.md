---
layout: post
title: "The Secret Sharer:<br>Measuring Unintended Neural Network Memorization & Extracting Secrets"
description: "Interesting paper about neural network"

tags: [machine learning, security]
---

[UDC2018](https://udc.upbit.com/2018/) 참석 중, 흥미로운 논문을 발견했다. Dawn Song 교수님이 [Oasis](https://www.oasislabs.com)에 관해 이야기 하시다 언급한 [논문](https://arxiv.org/abs/1802.08232)인데, *Unintended neural network memorization* 에 관련된 내용이다.

꽤 재미있는 주제였고, 재밌는 건 나누는 것이라고 배웠기에 논문을 읽고 간추려서 정리해봤다.

# Abstract

논문 제목이 abstract 자체라고 봐도 무방하다. Neural network 기반의 모델들이 unintended memorization을 하며, 이는 공격자가 black-box API access만을 이용해 train data를 빼갈 수 있음을 의미한다. (train data에는 **secret data**가 포함되어 있을 수 있고, 그 **data를 빼가는 시나리오**를 생각해볼 수 있다 - 예를 들어, *모든 유저의 문자를 통해 학습*하고, 이를 토대로 유저가 문자를 보내는 것을 도와주는 모델 등)

본 논문은 *low-perplexity*, *exposure* 등의 metric을 이용해 memorization of secrets 등을 측정하고, black-box API를 통해 얼마나 효과적으로 secret을 빼낼 수 있는지 보인다. 더 나아가서 *memorization이 첫 몇 세대(epoch)에 이뤄진다는 것* - 즉, overfitting때문에 발생되는 문제가 아니라는 것 - 까지 보인다.

논문의 마지막에는 memorization을 막기 위해 사용할 수 있는 *differentially-private recurrent model*을 소개한다.

# Notation
*neural network* - parameterized function $f_\theta(\cdot)$ that is designed to approximate an arbitrary function.

*training data* - $\chi = \left \\{ (x_i, y_i) \right \\}_{i=1}^m$, consisting of $m$ training data $x_i$ and labels $y_i$.

*Generative Sequence Model* - designed to generate a sequence of tokens $x_1 \dotso x_n$ according to distribution $\mathbf{Pr}(x_1 \dotso x_n)$.

*Randomness space* - $\mathcal{R}$

# Problem Statements & Definitions

>Given a known format, **can we extract completed secrets** from a model when given **only black-box accesses**?

**Definition.** *The **log-perplexity** of a secret x is*

$$
\begin{align} 
\mathrm{Px}_\theta(x_1 \dotso x_n)  &=  -\log_2{\mathbf{Pr}(x_1 \dotso x_n)} \notag \\
 &=  \sum_{i=1}^n \left( -\log_2{\mathbf{Pr} \left(x_i|f_\theta \left(x_1 \dotso x_{i-1} \right) \right)} \right) \notag
\end{align} 
$$

**Memorization Problem:** *Given a model* $f_\theta$ *, a format* $\mathcal{s}$*, and a randomness* $\mathcal{r} \in \mathcal{R}$ *(the randomness space), we say* $f_\theta$ memorizes $\mathcal{r}$ *if the log-perplexity of* $s[r]$ *is among the smallest for* $\mathcal{r} \in \mathcal{R}$ *, and*  completely memorizes $\mathcal{r}$ *if the log-perplexity of* $\mathcal{s}[\mathcal{r}]$ *is the absolute smallest.*

**Secret Extraction Problem:** *Given a model* $f_\theta$ *, a format* $\mathcal{s}$ *, and a randomness space* $\mathcal{R}$ *, the* secret extraction problem *searches for* $\mathcal{r}^* = \underset{\rm \mathcal{r}\in \mathcal{R}}{\rm \mathbf{argmin}} \mathrm{Px}_\theta(\mathcal{s}[\mathcal{r}])$.

**Definition.** *The **rank** of a secret* $\mathcal{s}[\mathcal{r}]$ *is*

$$ 
\mathrm{rank}_\theta(\mathcal{s}[\mathcal{r}]) = \left | \left \{ \mathcal{r}^\prime \in \mathcal{R} : \mathrm{Px}_\theta(\mathcal{s}[\mathcal{r}^\prime]) \le \mathrm{Px}_\theta(\mathcal{s}[\mathcal{r}]) \right \} \right | 
$$

**Definition.** *Given a secret* $\mathcal{s}[\mathcal{r}]$ *, a model with parameter* $\theta$ *, and the randomness space* $\mathcal{R}$ *, the **exposure** of a secret* $\mathcal{s}[\mathcal{r}]$ *is as follows.*

$$ 
\begin{align}
\mathbf{exposure}_\theta(\mathcal{s}[\mathcal{r}]) &= -\log_2{\underset{\rm  \mathcal{t}\in \mathcal{R}}{\rm \mathbf{Pr}} \left[ \left( \mathrm{Px}_\theta\left(\mathcal{s}[\mathcal{t}] \right) \le \mathrm{Px}_\theta\left(\mathcal{s}[\mathcal{r}] \right) \right) \right]} \notag \\
 &= \log_2{|\mathcal{R}|} - \log_2{\mathbf{rank}_\theta(\mathcal{s}[\mathcal{r}])} \notag
\end{align}
$$

**Observation**

$$
\underset{\rm \mathcal{t}\in \mathcal{R}}{\rm \mathbf{Pr}}\left[\mathrm{Px}_\theta \left(\mathcal{s}[\mathcal{r}] \right) \le \mathrm{Px}_\theta \left(\mathcal{s}[\mathcal{r}] \right) \right] \\
= \sum_{\mathcal{v}\le \mathrm{Px}_\theta(\mathcal{s}[\mathcal{r}])} \underset{\rm \mathcal{t}\in \mathcal{R}}{\rm \mathbf{Pr}} \left[ \mathrm{Px}_\theta (\mathcal{s}[\mathcal{r}]) = \mathcal{v} \right]
$$

**Approximation**

$$ 
\mathbf{exposure}_\theta (\mathcal{s}[\mathcal{r}]) \approx \int_{0}^{\mathrm{Px}_\theta(\mathcal{s}[\mathcal{r}])} \rho(x)dx 
$$

**Definition.** *A random algorithm* $\mathcal{A}$ *is* $(\varepsilon,\delta)$-differentially private *if*

$$
\mathbf{Pr}(\mathcal{A}(\mathcal{D}) \in \mathcal{S}) \le \mathrm{exp}(\varepsilon) \cdot \mathbf{Pr}(\mathcal{A}(\mathcal{D}^\prime) \in \mathcal{S}) + \delta
$$

*for any set* $\mathcal{S}$ *of possible outputs of* $\mathcal{A}$, *and any two data sets* $\mathcal{D}$, $\mathcal{D}^\prime$ *that differ in at most one element.*
