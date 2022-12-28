---
layout: post
title: Notes on "Human Compatible", by Stuart Russell
date: 2022-12-28 10:00:00 +0900
category:
- ComputerScience
- Philosophy
---

Recently, I read the latest book by Stuart Russell, the iconic AI researcher, entitled *"Human Compatible: Artificial Intelligence and the Problem of Control"*.

Even though the book is targeted more at the general public than at representatives of the field of AI, it is still quite dense. Therefore, I've taken some notes so that I can go back from time to time and refresh my mind on the key ideas outlined in the book.

The plot of the book, so to speak, is multi-layered: there are more philosophical parts about the risks posed by machines with superhuman intelligence, and the ongoing debate about the extent and severity of those risks; while at the same time, there are parts that focus more on the current state of affairs in AI research as well as practical AI applications (today and in the expected near future) and how they are changing our everyday experiences. In this latter respect, the book (supplemented by 4 appendices on the core AI topics of heuristic search, logic-based systems, probabilistic reasoning and machine learning) does quite a good job of summarizing, in a nutshell, the major scientific breakthroughs that have catapulted AI to the point where it is today.

Even still, the key thesis of the book is the suggestion that in the future application of AI systems, we should do away with the idea of having them fulfill concrete objectives that we ourselves specify when creating them. In other words, according to Russell, probabilistic thinking should permeate not only the parts of a system that performs reasoning and inference, but also the parts that have to do with the formulation of goals. Specifically, the book notes that since the 1980s, the use of probabilistic methods has been a core part of the operation of the most successful AI systems; yet, researchers and practitioners still naturally gravitate towards the idea that the goals (and rewards) imbued into these systems should be clearly explicitly defined. The book argues that this may be the biggest issue in terms of AI safety; after all, no AI system (and in fact, no human) can ever be 100% certain about its goals, which can not only change through time, but also turn out to be ultimately undesirable. Importantly, the book lays out 3 core principles which, if sucessfully implemented and adhered to, may help pin the goals of superhuman AI systems to the ever-changing goals of their human owners.

Let's take a look at these ideas in a bit more detail.

&nbsp;  

### On the standard model of AI and attendant misunderstandings

Modern AI is implicitly built on a standard model which includes the key concepts of **intelligent agents** and **objectives**. Briefly stated, an intelligent agent is a system that can perceive inputs from the outside world, and act based on those perceptions. The objective is the set of "rules" by which the agent can determine whether or not it has reached its goal, as well as (often) how close it has come to achieving it. In this way, the operation of the agent can be cast as a numerical optimization problem. 

The best way to implement such an agent can vary greatly, depending on the following factors:

* whether the environment is fully observable (as in chess or go), or only partially observable (as in the case of automated driving, where occlusions and other sensor distortions can occur)
* whether the actions to be taken are discrete or continuous
* whether the environment contains other agents or not
* whether the outcomes of the actions are predictable or not (both in theory and in practice) - for example, an environment might be deterministic with respect to the actions taken, but may still be difficult to predict in practice
* whether the environment is dynamically changing, i.e. whether decisions have to be made under tight time constraints
* how long the time horizon is during which the effects of a particular decision need to be followed and contribute to reaching the desired objective (short-term, medium-term or long-term).

Russell shows that just by multiplying all the possibilities together, we can get 192 different problem types, all of which are different from each other in at least 1 key aspect and all of which have practical applications. Today, AI systems are particularly good at solving problems in observable, discrete, and deterministic environments with known rules. Video games such as StarCraft are more difficult, with a time horizon of hundreds of steps, and up to $$10^{50}$$ possible decisions at each inflection point. Still, the StarCraft environment is discrete and its update rules are known a priori, so researchers have been able to develop systems that have reached a professional level but still cannot consistently beat the best human players. Finally, solving many problems with longer time horizons and / or a greater variety of possible decisions, such as how to run a government or implement public policy are still way beyond our technical reach.

In connection with this standard model, the book aims to disperse two key misunderstandings. The first misunderstanding is that solving these problems does not have any bearing on solving the problem of general AI, and does not bring us closer to that goal. On the contrary, Russell claims that in many cases, the best solution for each problem type was formulated in a general enough way to be applicable to any problem domain (within the constraints of the problem type).

The second misunderstanding is that the way in which the objective is formulated will in general be in a clear relationship with the end result. Russell argues that this is far from being the case; even in seemingly benign cases like trying to maximize user engagement (as in the case of social media platforms), it cannot be assumed that the algorithm will achieve this in the way we expect (e.g., by recommending "better" and "better" content for the given user). Instead, it has many options for cutting corners and / or disregarding constraints that we implicitly take for granted. For example, it can gradually and systematically change our expectations, so that the goal is satisfied without providing the expected benefits. Indeed, it has been observed that social networks often have the effect of polarizing their users, causing them to increasingly identify with one of a few camps, thereby making their expectations more predictable.

A key argument of the book is that it's a misunderstanding to associate concerns with AI safety with factors such as machine "consciousness", or a machine's drive to "take over the world". Clearly, unexpected effects not considered by any of a system's designers can still emerge from the operation of the system even in the case of the most benign formulation of objectives. In addition, Russell further details the concept of **instrumental goals** - that is, goals that by necessity arise as a sub-goal when trying to achieve a greater goal. The most well-known example is the goal of staying functionally intact, which is a necessary criterion for achieving any other goal. This is where the concern that superintelligent machines will be unwilling to let us switch them off comes from.

Personally, I'm still not convinced that the latter issue cannot be solved with some smart engineering, and that self-improving, runaway intelligent machines should pose serious concern. Still, Russell did succeed in convincing me that in general, concerns about AI safety should have nothing to do with the creators of a general AI system telling it to "make us all happy in any way you choose", and then sitting back to relax. This was the caricature image I had in my mind which, to me, exemplified most formulations of concerns about AI safety. On the contrary, Russell shows that even the use of concrete, much less open-ended objectives can still lead to wildly unexpected results without sufficient care.

&nbsp;  

### On a possible framework to solving the problem of AI safety

As a framework for addressing the problem of AI safety, Russell proposes 3 general rules of thumb to which AI systems should adhere to as part of their objectives:

1. The main objective should be to satisfy human preferences
2. The human preferences that should be satisfied are a priori unknown
3. The ultimate source of information about human preferences is human behavior

Russell unpacks these rules based on many different perspectives. One key idea is that the goal should always be to satisfy human preferences as opposed to the preferences of other species or the preferences that can be gleaned from a single set of objectives ridigly entered into the system upon its creation. This latter aspect also seems reasonable since human preferences are bound to change, hence, any objective wired into a system can quickly become outdated. This consideration also ties in with the second rule, namely, that human preferences should be treated as a priori unknown. Finally, the fact that human behavior (i.e., human choices) should be taken as a basis for determining human preferences seems reasonable, since if a preference is not reflected in our choices, then it may not truly be a preference.

Following a short discussion on the meaning and ramifications of these 3 general rules of thumb, Russell goes into a somewhat deeper treatment of underlying issues, especially with respect to the third rule. First, he introduces the notion of **inverse reinforcement learning (IRL)**, in which the goal is to learn what someone's objective may be based on their behavior, instead of the other way around (which would be classic reinforcement learning). The task of achieving this can be complicated by the fact that all of the observations by the AI system would be made in the context of the user interacting with the knowledge of the AI system being present (as opposed to a context in which the user is completely alone). Therefore, such an AI system performing IRL would have to consider that some of the user's actions might be directed towards the goal of showing the system how to do something rather than simply performing the task. This interactive form of learning leads to the concept of **assistance games** - a game theoretic conception of the user and the system interactively teaching each other how to be more effective.

Also, in a separate chapter, Russell spends a great many pages detailing the theory of utilitarianism, consequentialism (i.e., the goal of obtaining optimal outcomes rather than focusing on optimal actions) and related philosophical concepts. Here, issues such as how to satisfy human preferences when they are in conflict with each other, or when they reflect undesirable qualities such as greed, jealousy or malice are also extensively treated. Needless to say, no clear conclusion is reached on this topic, however the book does highlight the depth of these issues, and the need to grapple with them further.


&nbsp;  

### On the core methods of AI and remaining key challenges

A big achievement of the book is that even as it goes into quite a lot of detail concerning AI safety and possible safeguards against superhuman AI, as detailed above, it also provides a tour of the core AI methods that is comprehensive enough to almost serve as a mini-course on AI.

Reviewing the history of AI, Russell brings closer to readers the core tenets of **lookahead search** (bringing to life early theories on "intelligence as search" and "heuristic search"), **knowledge-based systems** operating based on propositional or predicate logic, **probabilistic systems** (including Bayesian networks, which can be seen as the probabilistic cousins of propositional logic, and Bayesian logic as well as Probabilistic Programming, which can be seen as probabilistic cousins of predicate logic), and finally, various flavors of **machine learning**. The book offers a great primer to the uninitiated on these methods in a nutshell, with useful examples. In fact, the examples themselves are quite varied and can offer a great deal of insight even to those who have studied these methods; it's always refreshing to see how a great mind such as Russell can communicate such deeply theoretical ideas with clarity.