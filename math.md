Here is the mathematical foundation behind NeuralMeshLite, covering the neural computations, evolutionary operators, and swarm coordination mechanisms.

---

## 📐 Mathematical Foundations of NeuralMeshLite

### 1. Micro-Mind Neural Network Forward Pass

Each micro-mind contains a tiny feedforward network with one hidden layer.

**Input**: $\mathbf{x} \in \mathbb{R}^{d_{\text{in}}}$

**Hidden Layer**:
$$\mathbf{h} = \sigma(\mathbf{W}_1 \mathbf{x} + \mathbf{b}_1)$$
where $\mathbf{W}_1 \in \mathbb{R}^{32 \times d_{\text{in}}}$, $\mathbf{b}_1 \in \mathbb{R}^{32}$, and $\sigma$ is ReLU: $\sigma(z) = \max(0, z)$.

**Output Layer**:
$$\hat{\mathbf{y}} = \text{softmax}(\mathbf{W}_2 \mathbf{h} + \mathbf{b}_2)$$
where $\mathbf{W}_2 \in \mathbb{R}^{d_{\text{out}} \times 32}$, $\mathbf{b}_2 \in \mathbb{R}^{d_{\text{out}}}$, and $\text{softmax}(z)_i = \frac{e^{z_i}}{\sum_j e^{z_j}}$.

```elixir
# Code mapping
tensor = Nx.tensor(input)
prediction = Axon.predict(state.model, state.params, tensor)
```

---

### 2. Fitness Evaluation

Fitness $F$ measures task performance. For a batch of tasks:
$$F = \frac{1}{N}\sum_{i=1}^N \mathcal{L}(\hat{\mathbf{y}}_i, \mathbf{y}_i)$$
where $\mathcal{L}$ is task-specific (e.g., cross-entropy for classification).

```elixir
def handle_cast({:update_fitness, fitness}, state) do
  {:noreply, %{state | fitness: fitness}}
end
```

---

### 3. Tournament Selection

Tournament selection chooses parents with probability proportional to fitness rank.

For tournament size $k$, the probability of selecting the $i$-th ranked individual (out of $n$) is:
$$P(i) = \frac{(n-i+1)^k - (n-i)^k}{n^k}$$

```elixir
defp tournament_select(population, tournament_size \\ 3) do
  population
  |> Enum.take_random(tournament_size)
  |> Enum.max_by(& &1.fitness)
end
```

---

### 4. Crossover (Parameter Averaging)

Given two parent parameter sets $\boldsymbol{\theta}^{(1)}$ and $\boldsymbol{\theta}^{(2)}$, child parameters are computed via element-wise averaging:
$$\boldsymbol{\theta}^{(\text{child})} = \frac{\boldsymbol{\theta}^{(1)} + \boldsymbol{\theta}^{(2)}}{2}$$

```elixir
defp crossover(genome1, genome2) do
  params = deep_merge(genome1.params, genome2.params, fn p1, p2 -> 
    Nx.add(p1, p2) |> Nx.divide(2.0)
  end)
end
```

---

### 5. Gaussian Mutation

Parameters are perturbed with additive Gaussian noise:
$$\theta_j' = \theta_j + \epsilon, \quad \epsilon \sim \mathcal{N}(0, \sigma^2)$$
where $\sigma$ is the mutation rate.

```elixir
defp mutate(genome, rate) do
  mutated_params = Nx.Defn.grad(genome.params, fn p -> 
    Nx.add(p, Nx.random_normal(p, 0.0, rate))
  end)
end
```

---

### 6. Task Routing (Specialist Selection)

Given a task type $T$, the swarm selects a specialist mind. The probability of selecting mind $i$ is:
$$P(i | T) = \begin{cases} 
\frac{1}{N_T} & \text{if specialization}(i) = T \text{ and } N_T > 0 \\
\frac{1}{N} & \text{otherwise (fallback to any)}
\end{cases}$$
where $N_T$ is the number of minds specialized in task $T$.

```elixir
specialists = Enum.filter(state.minds, fn {_, {_, spec, _}} -> spec == task_type end)
{pid, _} = if Enum.empty?(specialists) do
  Enum.random(state.minds)
else
  Enum.random(specialists)
end
```

---

### 7. Elitism Preservation

The top $e$ individuals by fitness are guaranteed survival to the next generation:
$$\mathcal{E} = \{i \in \mathcal{P} \mid \text{rank}(i) \leq e\}$$

```elixir
sorted = Enum.sort_by(population, & &1.fitness, :desc)
elites = Enum.take(sorted, state.elite_count)
```

---

### 8. Population Dynamics

The swarm maintains a fixed size $N$. After evolution, the population is:
$$\mathcal{P}_{t+1} = \mathcal{E} \cup \mathcal{O}$$
where $\mathcal{O}$ are $N - e$ offspring generated via selection, crossover, and mutation.

---

### 9. Softmax Activation (Output Layer)

For classification tasks, the output logits $\mathbf{z}$ are transformed to probabilities:
$$\hat{y}_i = \frac{e^{z_i}}{\sum_{j=1}^{d_{\text{out}}} e^{z_j}}$$

This is handled internally by Axon's `:softmax` activation.

---

### 10. ReLU Activation (Hidden Layer)

Hidden layer non-linearity:
$$\text{ReLU}(z) = \max(0, z)$$

Its gradient is:
$$\frac{d}{dz}\text{ReLU}(z) = \begin{cases} 1 & z > 0 \\ 0 & z \leq 0 \end{cases}$$

---

## 💎 Summary Table

| Component | Mathematical Operation | Code Location |
|:---|:---|:---|
| Forward Pass | $\mathbf{h} = \text{ReLU}(\mathbf{W}_1\mathbf{x} + \mathbf{b}_1)$, $\hat{\mathbf{y}} = \text{softmax}(\mathbf{W}_2\mathbf{h} + \mathbf{b}_2)$ | `MicroMind.predict/2` |
| Fitness | $F = \frac{1}{N}\sum \mathcal{L}(\hat{\mathbf{y}}, \mathbf{y})$ | `MicroMind.update_fitness/2` |
| Tournament Selection | $P(i) \propto \text{rank}(i)^k$ | `EvolutionEngine.tournament_select/2` |
| Crossover | $\boldsymbol{\theta}^{(\text{child})} = (\boldsymbol{\theta}^{(1)} + \boldsymbol{\theta}^{(2)})/2$ | `EvolutionEngine.crossover/2` |
| Mutation | $\theta_j' = \theta_j + \mathcal{N}(0, \sigma^2)$ | `EvolutionEngine.mutate/2` |
| Task Routing | $P(i \mid T) = 1/N_T$ if specialist else $1/N$ | `SwarmManager.handle_call/3` |
| Elitism | $\mathcal{E} = \text{top}_e(\mathcal{P})$ | `EvolutionEngine.handle_cast/2` |

These mathematical foundations enable NeuralMeshLite to evolve a decentralized swarm of specialized neural networks efficiently.
