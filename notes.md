## It's okay to neglect partition function

Both definition of Stein's operator and proof of connnection to KL divergence (Theorem 3.1, via Appendix A1)
work through the score function

$$s_p(x) = \nabla_x \log p(x) = \nabla_x \log \bar{p}(x) - \nabla_x \log Z = \nabla_x \log \bar{p}(x)$$

So all arguments hold using unnormalized probability for score function.

## Minibatches are a different subsampling / MC approximation than particles

Want to minimize $KL(q || p_{x \mid D})$, where $p_{x \mid D}$ is the posterior.
In this case
$$E_{x \sim q}[A_{p_{x \mid D}} k(x, \cdot)] = E_{x \sim q}[k(x, \cdot) \nabla_x \log p(x \mid D) + \nabla_x k(x, \cdot)]$$

The "particles" are performing a Monte carlo approximation of $q$, i.e. for $x_i \sim q$
$$E_{x \sim q}[k(x, \cdot) \nabla_x \log p(x \mid D) + \nabla_x k(x, \cdot)] 
\approx {1 \over N} \sum_{i=1}^N (k(x_i, \cdot) \nabla_{x_i} \log p(x_i \mid D) + \nabla_x k(x, \cdot))$$
Note that this MC approximation has nothing to do with minibatches.

Since we are focusing on scaling to large datasets, we focus on the posterior term (which is the only term involving
the data $D$):

$$\nabla_{x_i} \log p(x_i \mid D) = \nabla_{x_i} \log p(D \mid x_i) + \nabla_{x_i} \log p(x_i) - \nabla_{x_i} \log Z$$

The last term is equal to zero, and the middle term does not depend on the data. The remaining (left-most) term
is a score function.

In Bayesian treatments, the data $D$ is fixed and many models furter assume iid $D_i$. Then
$$\nabla_{x_i} \log p(D \mid x_i) = \sum_{j=1}^K \nabla_{x_i} \log p(D_j \mid x_i)$$
Minibatches can be viewed as a further subsampling of this finite sum: let $N_s \subset [K]^{b}$ be chosen
iid uar. Then
$$\nabla_{x_i} \log p(D \mid x_i) = E_{N_s} \sum_{j \in N_s} \nabla_{x_i} p(D_j \mid x_i)$$
TODO: bound variance, consider sampling without replacement.

### When $D$ is random

Other treatments may view the data $D$ as random. One way to connect SVGD to KL divergence is to take
expectations over data $D$.
Then we have (with suitable regularity conditions)
$$\nabla_\epsilon E_D KL(q || p_{x \mid D}) = E_D \nabla_\epsilon KL(q || p_{x \mid D}) = - E_D E_{x \sim q} A_{p_{x \mid D}} \phi(x)$$

For any fixed $D$, the optimal perturbation in the RKHS is still the same, so by
law of iterated expectations

$$
E_D[ E[\arg\max_{\phi} E_{x \sim q} A_{p_{x \mid D}}\phi(x) \mid D]]
= E_D[ E[E_{x \sim q}[A_{p_{x \mid D}} k(x, \cdot)] \mid D]]
= E_D E_{x \sim q} A_{p_{x \mid D}} k(x, \cdot)
$$

Note that this is the first equation of this section, but with both sides inside an expectation $E_D$. Expanding the definitions gives
$$
E_D E_{x \sim q}[A_{p_{x \mid D}} k(x, \cdot)] = E_D E_{x \sim q}[k(x, \cdot) \nabla_x \log p(x \mid D) + \nabla_x k(x, \cdot)]
$$

Since $q$ is deterministic and $x \sim q$, we have that $x \perp D$ hence
$$
E_D E_{x \sim q}[A_{p_{x \mid D}} k(x, \cdot)] = E_{x \sim q}[k(x, \cdot) E_D[\nabla_x \log p(x \mid D)] + \nabla_x k(x, \cdot)]
$$

Some remarks:
 * We already made one MC approximation when we used the empirical dataset to estimate $E_D$, but this is fairly well justified by Glivenko-Cantelli
   and inevitable since we don't have $p(D)$ nor infinite data.
 * We made another MC approximation when we used particles to approximation $E_q$.
 * Yet another MC approximation is being made to estimate the empirical sum of scores with a subsampled sum of scores.


## Other ideas for distributed computation

 * Travelling particles. Load-balancing and trajectory sampling w/ importance reweighting following http://proceedings.mlr.press/v32/ahn14.html
     * Step-size (via reweighting the local score function estimates) used to adjust bias (e.g. imbalanced datasets, more steps made on a faster shard)
     * Travelling particles means more than just minibatches; **the local particles are also random**.
        * $q$ is now a mixture distribution, with number of components scaling exponentially $O(s^l)$ (each
          component corresponds to all the shards a particle has previously been in)
        * Might be able to be smart and route particles with large pairwise kernel values to same nodes

 * Use local gradient $\sum_{j \in N_s} \nabla_{x_i} \log p(D_j \mid x_i)$ as surrogate, importance sampling reweighting
 $$
 w_j = {\rho(x_i) \over p(x_i)} = \frac{\prod_{j \in N_s} p(D_j \mid x_i)}{\prod_{s=1}^S \prod_{j \in N_s} p(D_j \mid x_i)}
 $$
   * This only needs to communicate $n$ (num particles) scalar values corresponding to likelihoods of each particle on each
     shard, which can be more efficient than communicating full gradients ($n \times d$)
   * Bias correction is different than Ahn 2014, which use dataset size on each shard as weight