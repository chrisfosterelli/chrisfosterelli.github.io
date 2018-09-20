---
layout: post
has_math: true
permalink: entropy-loss-for-reinforcement-learning
title: Entropy loss for reinforcement learning
image: https://fosterelli.co/image/entropy-loss-for-reinforcement-learning/social.png
---

Reinforcement learning agents are notoriously unstable to train compared to 
other types of machine learning algorithms. One of the ways that a reinforcement
learning algorithm can underperform is by becoming stuck during training on a 
strategy that is neither a good solution nor the absolute worst solution. We 
generally refer to this phenomenon as reaching a "local minimum" in the space
of game strategies. 

<!-- Content Breaker -->

[Local minima] under traditional supervised learning occurs when gradient descent
finds a parameter set that results in lower error than expected, and the near
parameters all result in an increase of error. Although there is a better set of
parameters in a different location of the parameter space, gradient descent 
isn't able to find it and becomes stuck in the local minimum. Some methods, such
as training with momentum, help with this.

In reinforcement learning, a similar situation can occur if the agent discovers
a strategy that results in a reward that is better than it was receiving when it
first started, but very far from a strategy that would result in optimal reward.
This might often manifest as the agent deciding to take a single move, over and
over. Entropy loss, I believe first discussed in a [1991 paper], is an 
additional loss parameter that can be added to a model to help with local minima
in the same way that momentum might and to provide encouragement for the agent 
to take a variety of moves and explore the set of strategies more.

For example, take a game like [snake]. In this environment, the snake must learn
to navigate the environment it exists in without crashing into itself or a wall.
The agent will have four moves available to it: up, right, down, and left. The
agent receives +1 reward for every step it takes successfully and -1 reward for
each time it crashes into either a wall or a snake's tail. We ignore food in 
this simple example.

In this environment, the randomly initialized weights will yield immediate death 
1/4 of the time when the snake "moves back" onto its own tail. One of the first
things the agent should learn is to avoid doing this. So lets train it and see a
few example runs!

<video autoplay loop muted width="300">
  <source
    src="/file/entropy-loss-for-reinforcement-learning/before.mp4"
    type="video/mp4">
  Your browser does not support this video.
</video>

It learns this very well! Unfortunately, it learns to avoid its tail by 
restricting its movement in only one or two directions. By moving in that 
fashion it does ensure the snake will never go back on its tail, but this 
strategy is clearly suboptimal as the agent only lasts a short period before
crashing into the wall.

As the agent progresses learning, its average move distribution will grow
closer and closer to predictions where either a single move is guarenteed 
(`[ 1 0 0 0 ]`) or two moves are split equally (`[ 0.5 0.5 0 0 ]`). With a very 
low or zero probability, it's unlikely to ever explore moving a different 
direction when faced with a wall. Because it reaches this local minima state,
the agent never learns to understand how to avoid itself or that it must avoid
walls.

To solve this, we will encourage the agent to vary its moves by adding a new
loss parameter based on the entropy of its predicted moves. The equation for
entropy here is:

$$
H(x) = -\sum_{i=1}^n {\mathrm{P}(x_i) \log_e \mathrm{P}(x_i)}
$$

At each timestep, we will calculate the entropy of the softmax prediction output
from the policy network. When we calculate the training loss at the end of the 
epoch, we'll subtract the entropy loss over all of the timesteps from the 
reward-based policy and value losses. 

```python
beta = 0.05 # Hyperparameter that controls the influence of entropy loss
entropy_loss = torch.stack(entropy_losses).sum() # Sum of H(X) from all steps
policy_loss = torch.stack(policy_losses).sum() # Sum of reward policy losses
value_loss = torch.stack(value_losses).sum() # Sum of reward value losses
loss = policy_loss + value_loss - (beta * entropy_loss) # Adjusted loss function
loss.backward() # Standard gradient update
```

Intuitively, a lot of predictions such as `[ 1 0 0 0 ]` are going to have very 
low entropy, and our entropy loss component will be more positive. This 
encourages the network to only make strong predictions if it is _highly_ 
confident in them, that means that the actor critic network will have to have a
low enough loss from the policy and value loss parameters to offset the more 
positive entropy loss sufficiently so that the global loss remains low.

<video autoplay loop muted width="300">
  <source
    src="/file/entropy-loss-for-reinforcement-learning/after.mp4"
    type="video/mp4">
  Your browser does not support this video.
</video>

When we train the same network with this new entropy loss component, we get an
agent that learns a significantly more diverse strategy and can survive without 
hitting walls!

Entropy loss is a clever and simple mechanism to encourage the agent to explore
by providing a loss parameter that teaches the network to avoid very confident 
predictions. As the distribution of the predictions becomes more spread out, the
network will sample those moves more often and learn that they can lead to 
greater reward.

While this is discussed briefly by the [Atari deep RL paper], I found that 
entropy loss can be critical to having effective agents in some circumstances 
and applies to more than just the network design mentioned in the paper. I 
couldn't find much explanaining the intution behind entropy loss, though it pops
up in lots of places, so hopefully helps elaborate on a real example of how it 
can take an agent from "not learning anything" to "solved the game" in some 
environments.

[snake]: https://en.wikipedia.org/wiki/Snake_(video_game_genre)
[local minima]: https://en.wikipedia.org/wiki/Maxima_and_minima
[1991 paper]: http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.54.3433&rep=rep1&type=pdf
[Atari deep RL paper]: https://www.cs.toronto.edu/~vmnih/docs/dqn.pdf
