# Unit VI — Phase 8: K-means Clustering

> **Continuing from Phase 7.** All algorithms so far were **supervised**: they learned from labels. **K-means is unsupervised**: there are **no labels**, and the algorithm discovers natural **groups** in the data. We learn the distance measure, the objective (WCSS), the algorithm with a **complete hand iteration on 7 points**, initialization (k-means++), choosing K (elbow and silhouette, also worked by hand), why scaling matters, and the PySpark code with cluster interpretation.
>
> All numbers are real: the hand iteration, our own numpy check and Spark's result agree (final cost 8.525, silhouette 0.8315).

---

## Phase 8 — Roadmap

| # | Topic |
| --- | --- |
| 1 | The idea of clustering |
| 2 | Distance: Euclidean |
| 3 | The objective: WCSS (inertia) |
| 4 | The K-means algorithm |
| 5 | A complete hand-worked example |
| 6 | Convergence and local minima |
| 7 | Initialization: random, k-means++, k-means|| |
| 8 | Choosing K: the elbow method |
| 9 | Choosing K: the silhouette score |
| 10 | Why scaling matters |
| 11 | PySpark code: the 7-point example |
| 12 | PySpark code: customer segmentation |
| 13 | Choosing K for the customers |
| 14 | Interpreting the clusters |
| 15 | Scaling demonstration |
| 16 | Predicting a new point |
| 17 | Important parameters |
| 18 | Strengths and weaknesses |
| 19 | Common errors |
| 20 | Practical scenarios |
| 21 | Viva questions |
| 22 | Master revision |

---

### 1. The idea of clustering

> **Clustering groups data points so that points in the same group (cluster) are similar to each other and different from points in other groups.**

```text
 BEFORE: raw points, no labels            AFTER K-means (K = 3)
  · ·    ·   ·                              A A    B  B
 ·  ·  ·  ·    ·  ·                        A  A  B  B   C C
   ·   ·    ·  ·                             A   B    C  C
```

**Real uses:** segment customers (budget, premium…), group news articles by topic, compress images by colour, find similar documents, detect unusual points far from every cluster.

**K-means** is the most popular clustering algorithm. The "K" is the **number of clusters we ask for**, and "means" refers to each cluster being represented by the **mean (average)** of its points, called the **centroid**.

| Term | Meaning |
| --- | --- |
| **Cluster** | A group of similar points |
| **Centroid** (cluster centre) | The mean position of all points in the cluster |
| **K** | Number of clusters (a hyperparameter **we choose**) |

---

### 2. Distance: Euclidean

"Similar" is measured by **distance**. The standard choice is the **Euclidean distance** (straight-line distance).

For two points `p = (p₁, p₂, …, pₙ)` and `q = (q₁, q₂, …, qₙ)`:

```text
 d(p, q) = √[ (p₁ − q₁)² + (p₂ − q₂)² + … + (pₙ − qₙ)² ]
```

**Example.** Distance between A(1, 1) and C(3, 4):

```text
 d(A, C) = √[ (1 − 3)² + (1 − 4)² ] = √(4 + 9) = √13 = 3.606
```

The **squared Euclidean distance** `(p₁−q₁)² + …` (without the square root) is what the K-means objective and Spark's silhouette use by default.

---

### 3. The objective: WCSS (inertia)

K-means tries to make clusters **compact**. It measures this with the **Within-Cluster Sum of Squares (WCSS)**, also called **inertia**, **SSE** or Spark's **trainingCost**:

```text
 WCSS = Σ over clusters j  Σ over points x in cluster j   ‖ x − μⱼ ‖²

 μⱼ = centroid of cluster j      ‖x − μⱼ‖² = squared distance from x to its centroid
```

> **Goal:** choose the cluster assignment and centroids that **minimize WCSS**. Smaller WCSS = tighter clusters.

Finding the global minimum is very hard (NP-hard). K-means uses a simple iterative method that reliably finds a **good (local) minimum**.

---

### 4. The K-means algorithm

(Also called **Lloyd's algorithm**.)

```text
 1. CHOOSE K.
 2. INITIALIZE K centroids (pick K starting points).
 3. REPEAT until nothing changes:
      a. ASSIGN each point to its NEAREST centroid.            (E-step)
      b. UPDATE each centroid = MEAN of the points assigned to it. (M-step)
 4. STOP when assignments no longer change (or maxIter reached).
```

```text
   init centroids
         │
         ▼
   ┌─▶ assign points to nearest centroid
   │        │
   │        ▼
   │   recompute centroids (means)
   │        │
   └── changed? ──yes──┘
            │ no
            ▼
          DONE
```

**Why it works:** each step can only **decrease (or keep) WCSS**: assigning to the nearest centroid cannot increase it, and the mean is the point that minimizes the sum of squared distances. WCSS can never go below 0, so the process must settle.

---

### 5. A complete hand-worked example

Seven points (K = 2):

| Point | A | B | C | D | E | F | G |
| --- | --- | --- | --- | --- | --- | --- | --- |
| (x, y) | (1, 1) | (1.5, 2) | (3, 4) | (5, 7) | (3.5, 5) | (4.5, 5) | (3.5, 4.5) |

```text
 y
 7 │                  D
 6 │
 5 │            E   F
 4 │        C  G
 3 │
 2 │   B
 1 │ A
   └──────────────────── x
     1  2  3  4  5
```

**Step 0: initial centroids.** Use the first and last points: `c₁ = A (1, 1)`, `c₂ = D (5, 7)`.

#### Iteration 1

**Assign** each point to the nearer centroid (Euclidean distances):

| Point | d to c₁ (1,1) | d to c₂ (5,7) | Cluster |
| --- | --- | --- | --- |
| A (1, 1) | 0.00 | 7.21 | **1** |
| B (1.5, 2) | 1.12 | 6.10 | **1** |
| C (3, 4) | 3.61 | 3.61 | tie → we assign **1** |
| D (5, 7) | 7.21 | 0.00 | **2** |
| E (3.5, 5) | 4.72 | 2.50 | **2** |
| F (4.5, 5) | 5.32 | 2.06 | **2** |
| G (3.5, 4.5) | 4.30 | 2.92 | **2** |

(C is exactly equidistant: √13 from both. Ties are broken arbitrarily. The final answer would be the same either way.)

**Update** the centroids:

```text
 c₁ = mean of A, B, C = ( (1 + 1.5 + 3)/3 , (1 + 2 + 4)/3 ) = (1.833, 2.333)
 c₂ = mean of D, E, F, G = ( (5 + 3.5 + 4.5 + 3.5)/4 , (7 + 5 + 5 + 4.5)/4 ) = (4.125, 5.375)

 WCSS after iteration 1 = 12.208
```

#### Iteration 2

**Assign** using the new centroids:

| Point | d to c₁ (1.833, 2.333) | d to c₂ (4.125, 5.375) | Cluster |
| --- | --- | --- | --- |
| A | 1.57 | 5.38 | 1 |
| B | 0.47 | 4.28 | 1 |
| **C** | **2.03** | **1.78** | **2** ← moved! |
| D | 5.64 | 1.85 | 2 |
| E | 3.14 | 0.73 | 2 |
| F | 3.77 | 0.53 | 2 |
| G | 2.73 | 1.08 | 2 |

Point **C changed clusters** (it is now closer to c₂).

**Update:**

```text
 c₁ = mean of A, B       = ( (1 + 1.5)/2 , (1 + 2)/2 )             = (1.25, 1.5)
 c₂ = mean of C, D, E, F, G = ( (3+5+3.5+4.5+3.5)/5 , (4+7+5+5+4.5)/5 ) = (3.9, 5.1)

 WCSS = 8.525
```

#### Iteration 3

Assign again with `c₁ = (1.25, 1.5)` and `c₂ = (3.9, 5.1)`: the assignments are **identical** (A, B → 1; C, D, E, F, G → 2). Centroids do not move → **converged**.

#### Verify the final WCSS by hand

```text
 Cluster 1 centre (1.25, 1.5):
   A (1, 1):     (1−1.25)² + (1−1.5)²   = 0.0625 + 0.25  = 0.3125
   B (1.5, 2):   (1.5−1.25)² + (2−1.5)² = 0.0625 + 0.25  = 0.3125                → 0.625

 Cluster 2 centre (3.9, 5.1):
   C (3, 4):     0.81 + 1.21 = 2.02
   D (5, 7):     1.21 + 3.61 = 4.82
   E (3.5, 5):   0.16 + 0.01 = 0.17
   F (4.5, 5):   0.36 + 0.01 = 0.37
   G (3.5, 4.5): 0.16 + 0.36 = 0.52                                              → 7.90

 WCSS = 0.625 + 7.90 = 8.525   ✔
```

**Summary of the run**

| Iteration | Cluster 1 | Cluster 2 | WCSS |
| --- | --- | --- | --- |
| Start | centroid A (1, 1) | centroid D (5, 7) | – |
| 1 | {A, B, C}, centroid (1.833, 2.333) | {D, E, F, G}, centroid (4.125, 5.375) | 12.208 |
| 2 | {A, B}, centroid (1.25, 1.5) | {C, D, E, F, G}, centroid (3.9, 5.1) | **8.525** |
| 3 | no change → **converged** | | 8.525 |

WCSS decreased every iteration: 12.208 → 8.525 → (same).

---

### 6. Convergence and local minima

K-means **always converges**, but possibly to a **local** minimum depending on the starting centroids.

```text
 Good start                         Bad start
  ● ●     ● ●                        ●●●●●●  ← two centroids start here
  clusters found correctly           one real cluster gets split, two others merged
```

**Defences**

1. **Smart initialization** (k-means++, next section).
2. **Run several times** with different seeds and keep the lowest WCSS (the standard practice).
3. Set a `seed` so that results are reproducible.

---

### 7. Initialization: random, k-means++, k-means||

| Method | How the K starting centroids are chosen | Quality |
| --- | --- | --- |
| **Random** | K random data points | Can be poor |
| **k-means++** | First centre random; each next centre chosen with probability proportional to the **squared distance** from the nearest existing centre (so centres spread out) | Much better |
| **k-means|| (Spark default)** | A **parallel** version of k-means++ for big distributed data | Good and scalable |

```python
KMeans(k=4, initMode="k-means||")     # default
KMeans(k=4, initMode="random")        # simple random start
```

> **Viva line:** Spark uses **k-means||**, the parallel version of k-means++, by default, which spreads the initial centres apart and avoids bad local minima.

---

### 8. Choosing K: the elbow method

K must be chosen by us. A larger K **always** lowers WCSS (with K = number of points, WCSS = 0), so we look for the point where the improvement **stops being worthwhile**.

```text
 WCSS
  │ ●
  │   ●
  │      ●______   ← the ELBOW: curve bends here → good K
  │            ●───●───●
  └────────────────────── K
    1   2   3   4   5   6
```

**Procedure:** run K-means for K = 2, 3, 4, …, record WCSS, plot, pick the elbow.

(Verified numbers appear in Section 13.)

---

### 9. Choosing K: the silhouette score

The **silhouette** measures how well each point fits its **own** cluster compared with the **nearest other** cluster.

For each point `i`:

```text
 a(i) = average distance from i to the other points in its OWN cluster      (cohesion)
 b(i) = average distance from i to the points in the NEAREST OTHER cluster  (separation)

            b(i) − a(i)
 s(i) = ─────────────────        s(i) ∈ [−1, +1]
         max( a(i), b(i) )
```

The overall silhouette is the **average of s(i) over all points**.

| Silhouette | Meaning |
| --- | --- |
| **close to +1** | Point is far from other clusters, close to its own: **well clustered** |
| **about 0** | Point is on the boundary between two clusters |
| **negative** | Point is probably in the **wrong** cluster |

| Average silhouette | Interpretation |
| --- | --- |
| 0.71 to 1.0 | Strong structure |
| 0.51 to 0.70 | Reasonable |
| 0.26 to 0.50 | Weak |
| ≤ 0.25 | No real structure |

**Hand calculation for our 7-point result** (Spark's default uses **squared** Euclidean distances). Clusters: {A, B} and {C, D, E, F, G}.

For point **A (1, 1)**:

```text
 a(A) = squared distance to B       = (1−1.5)² + (1−2)²  = 1.25
 b(A) = average squared distance to C, D, E, F, G
      = (13 + 52 + 22.25 + 28.25 + 18.5) / 5 = 134 / 5 = 26.8

 s(A) = (26.8 − 1.25) / 26.8 = 0.9534
```

All seven points (computed the same way):

| Point | a | b | s |
| --- | --- | --- | --- |
| A | 1.250 | 26.800 | 0.9534 |
| B | 1.250 | 16.950 | 0.9263 |
| C | 4.500 | 9.625 | 0.5325 |
| D | 8.000 | 44.625 | 0.8207 |
| E | 2.188 | 17.625 | 0.8759 |
| F | 2.438 | 23.125 | 0.8946 |
| G | 2.625 | 14.375 | 0.8174 |

```text
 Average silhouette = (0.9534 + 0.9263 + 0.5325 + 0.8207 + 0.8759 + 0.8946 + 0.8174) / 7 = 0.8315
```

**Point C has the lowest score (0.53)**: it sits on the edge between the clusters, the same point that tied in iteration 1. ✔ The overall **0.8315** is exactly what Spark reports (Section 11).

**Elbow vs silhouette**

| Elbow | Silhouette |
| --- | --- |
| Look for a bend in WCSS (a judgement call) | Pick the K with the **highest** score (more objective) |
| Always decreases with K | Rises then falls |

Use both, plus common sense about what the clusters *mean*.

---

### 10. Why scaling matters

K-means uses **distance**. If one feature has a much larger range, it **dominates** the distance and the other features are ignored.

**Example.** Customers described by annual income and spending score:

| | Income | Spending score (1 to 100) |
| --- | --- | --- |
| Customer 1 | ₹30,000 | 20 |
| Customer 2 | ₹31,000 | 90 |

```text
 d = √[ (31000 − 30000)² + (90 − 20)² ] = √(1,000,000 + 4,900) ≈ 1002.4
```

The big score difference (70) barely matters, because income differences are in thousands. The algorithm effectively clusters **by income only**.

> **Rule:** standardize features (`StandardScaler`) before K-means whenever they are on different scales. A demonstration is in Section 15.

---

### 11. PySpark code: the 7-point example

```python
from pyspark.ml.feature import VectorAssembler
from pyspark.ml.clustering import KMeans
from pyspark.ml.evaluation import ClusteringEvaluator

pts = [("A", 1.0, 1.0), ("B", 1.5, 2.0), ("C", 3.0, 4.0), ("D", 5.0, 7.0),
       ("E", 3.5, 5.0), ("F", 4.5, 5.0), ("G", 3.5, 4.5)]

df = spark.createDataFrame(pts, ["name", "x", "y"])
fv = VectorAssembler(inputCols=["x", "y"], outputCol="features").transform(df)

km = KMeans(k=2, seed=1, maxIter=20)
model = km.fit(fv)

print("centres:", model.clusterCenters())
print("cost   :", model.summary.trainingCost)
print("sizes  :", model.summary.clusterSizes)
print("iters  :", model.summary.numIter)
```

```text
centres: [array([3.9, 5.1]), array([1.25, 1.5])]
cost   : 8.525
sizes  : [5, 2]
iters  : 2
```

✔ **Exactly our hand result:** centres `(3.9, 5.1)` and `(1.25, 1.5)`, cost **8.525**, cluster sizes 5 and 2. (Spark numbers its clusters arbitrarily; here cluster 0 is the big one.)

```python
model.transform(fv).select("name", "x", "y", "prediction").orderBy("name").show()
```

```text
+----+---+---+----------+
|name|  x|  y|prediction|
+----+---+---+----------+
|   A|1.0|1.0|         1|
|   B|1.5|2.0|         1|
|   C|3.0|4.0|         0|
|   D|5.0|7.0|         0|
|   E|3.5|5.0|         0|
|   F|4.5|5.0|         0|
|   G|3.5|4.5|         0|
+----+---+---+----------+
```

✔ A, B form one cluster; C, D, E, F, G the other, as in our hand iteration.

```python
evaluator = ClusteringEvaluator(predictionCol="prediction", featuresCol="features",
                                metricName="silhouette", distanceMeasure="squaredEuclidean")
print("silhouette:", evaluator.evaluate(model.transform(fv)))
```

```text
silhouette: 0.8315257348314337
```

✔ Matches our hand-computed **0.8315**.

---

### 12. PySpark code: customer segmentation

A marketing team has 16 customers with **annual income (₹ thousands)** and **spending score (1 to 100)**. They want customer segments.

```python
cust = [(1, 15, 20), (2, 18, 25), (3, 20, 15), (4, 22, 22),
        (5, 18, 80), (6, 20, 85), (7, 24, 78), (8, 22, 82),
        (9, 80, 18), (10, 85, 22), (11, 88, 15), (12, 90, 20),
        (13, 82, 80), (14, 86, 85), (15, 90, 78), (16, 88, 82)]

cdf = spark.createDataFrame(cust, ["id", "income", "score"])
cv = VectorAssembler(inputCols=["income", "score"], outputCol="features").transform(cdf)
```

```text
 score
 100 │ ●●●●(low income, high spend)        ●●●●(high income, high spend)
     │
  50 │
     │
   0 │ ●●●●(low income, low spend)         ●●●●(high income, low spend)
     └─────────────────────────────────────────────── income
          20                                  90
```

---

### 13. Choosing K for the customers

Loop over several K values, recording both measures (the **elbow table**):

```python
ev = ClusteringEvaluator(predictionCol="prediction", featuresCol="features",
                         metricName="silhouette", distanceMeasure="squaredEuclidean")

for k in range(2, 7):
    m = KMeans(k=k, seed=42, maxIter=50).fit(cv)
    print("k", k, "WSSSE", round(m.summary.trainingCost, 1),
          "silhouette", round(ev.evaluate(m.transform(cv)), 4))
```

```text
k 2  WSSSE 15476.6  silhouette 0.6496
k 3  WSSSE 8085.4   silhouette 0.7343
k 4  WSSSE 271.8    silhouette 0.988
k 5  WSSSE 223.2    silhouette 0.8486
k 6  WSSSE 180.8    silhouette 0.6716
```

**Reading the table**

```text
 WSSSE
 15476 │ ●
  8085 │    ●
       │
   272 │          ●           ← the ELBOW at K = 4 (huge drop 8085 → 272)
   223 │               ●
   181 │                   ●  ← then only tiny improvements
       └──────────────────── K
         2   3   4   5   6
```

| K | WSSSE | Drop from previous | Silhouette |
| --- | --- | --- | --- |
| 2 | 15476.6 | – | 0.6496 |
| 3 | 8085.4 | −7391 | 0.7343 |
| **4** | **271.8** | **−7814 (huge)** | **0.9880 ← best** |
| 5 | 223.2 | −49 (tiny) | 0.8486 |
| 6 | 180.8 | −42 (tiny) | 0.6716 |

Both methods agree: **K = 4**. (WSSSE = within-set sum of squared errors, the same as WCSS.)

> The data was built from four visible groups, so the answer is clear. Real data is rarely this tidy, and the elbow is then a judgement call.

---

### 14. Interpreting the clusters

```python
model4 = KMeans(k=4, seed=42, maxIter=50).fit(cv)

for i, c in enumerate(model4.clusterCenters()):
    print("cluster", i, "centre [income, score] =", [round(float(v), 1) for v in c])
print("sizes:", model4.summary.clusterSizes)
```

```text
cluster 0 centre [income, score] = [18.8, 20.5]
cluster 1 centre [income, score] = [85.8, 18.8]
cluster 2 centre [income, score] = [21.0, 81.2]
cluster 3 centre [income, score] = [86.5, 81.2]
sizes: [4, 4, 4, 4]
```

```python
model4.transform(cv).orderBy("id").select("id", "income", "score", "prediction").show(16)
```

```text
+---+------+-----+----------+
| id|income|score|prediction|
+---+------+-----+----------+
|  1|    15|   20|         0|
|  2|    18|   25|         0|
|  3|    20|   15|         0|
|  4|    22|   22|         0|
|  5|    18|   80|         2|
|  6|    20|   85|         2|
|  7|    24|   78|         2|
|  8|    22|   82|         2|
|  9|    80|   18|         1|
| 10|    85|   22|         1|
| 11|    88|   15|         1|
| 12|    90|   20|         1|
| 13|    82|   80|         3|
| 14|    86|   85|         3|
| 15|    90|   78|         3|
| 16|    88|   82|         3|
+---+------+-----+----------+
```

**Naming the segments (the human step!)**

| Cluster | Centre (income, score) | Segment name | Business action |
| --- | --- | --- | --- |
| 0 | (18.8, 20.5) | **Budget savers**: low income, low spend | Discounts, value offers |
| 1 | (85.8, 18.8) | **Cautious high earners**: high income, low spend | Build trust, premium previews |
| 2 | (21.0, 81.2) | **Enthusiastic spenders**: low income, high spend | Loyalty and instalment plans |
| 3 | (86.5, 81.2) | **Premium customers**: high income, high spend | VIP service, exclusive launches |

> **Key point:** K-means only produces **numbers** (cluster 0, 1, 2, 3). The *meaning* of each cluster comes from **studying its centre and members**. Cluster numbers have no order and may change between runs.

**Summary of the fitted model**

| `summary` attribute | Meaning |
| --- | --- |
| `clusterSizes` | Number of points per cluster |
| `trainingCost` | WSSSE / WCSS |
| `numIter` | Iterations used |
| `predictions` | DataFrame with each point's cluster |
| `model.clusterCenters()` | List of centroid arrays |

---

### 15. Scaling demonstration

Put income in **rupees** (×1000) while the score stays 1 to 100, and cluster again with K = 4:

```python
cust2 = [(i, inc * 1000, sc) for (i, inc, sc) in cust]
c2 = VectorAssembler(inputCols=["income", "score"], outputCol="features") \
        .transform(spark.createDataFrame(cust2, ["id", "income", "score"]))

m_raw = KMeans(k=4, seed=42, maxIter=50).fit(c2)
print("unscaled sizes:", m_raw.summary.clusterSizes)
```

```text
unscaled sizes: [5, 6, 3, 2]
```

The expected groups are **4 + 4 + 4 + 4**, but the unscaled run gives **5, 6, 3, 2**: the groups are **wrong**, because the algorithm effectively clusters by income alone (income differences of tens of thousands drown out the 0 to 100 scores).

**Fix: standardize first**

```python
from pyspark.ml.feature import StandardScaler

scaler = StandardScaler(inputCol="features", outputCol="scaled", withMean=True, withStd=True).fit(c2)
scaled = scaler.transform(c2).drop("features").withColumnRenamed("scaled", "features")

m_scaled = KMeans(k=4, seed=42, maxIter=50).fit(scaled)
print("scaled sizes:", m_scaled.summary.clusterSizes)
print("silhouette  :", round(ev.evaluate(m_scaled.transform(scaled)), 4))
```

```text
scaled sizes: [4, 4, 4, 4]
silhouette  : 0.9882
```

| Run | Cluster sizes | Result |
| --- | --- | --- |
| Raw rupees + score | [5, 6, 3, 2] | ❌ wrong grouping |
| **Scaled** | **[4, 4, 4, 4]** | ✅ correct, silhouette 0.988 |

> **Always scale before K-means** when features have different units. (A Pipeline with `VectorAssembler → StandardScaler → KMeans` does this cleanly.)

---

### 16. Predicting a new point

A new customer earns ₹85k and has a spending score of 90. Which segment?

```python
from pyspark.ml.linalg import Vectors
print(model4.predict(Vectors.dense([85.0, 90.0])))
```

```text
3
```

Cluster 3 is the **Premium** segment ✔. (`predict` assigns the point to the **nearest centroid**.)

**Initialization check:** training with `initMode="random"` also reached the cost **271.8** here, because the groups are clear.

---

### 17. Important parameters

| Parameter | Default | Meaning |
| --- | --- | --- |
| `k` | 2 | Number of clusters |
| `maxIter` | 20 | Maximum iterations |
| `tol` | 1e-4 | Convergence tolerance (centroid movement) |
| `initMode` | `k-means\|\|` | `k-means\|\|` or `random` |
| `initSteps` | 2 | Steps of k-means\|\| |
| `distanceMeasure` | `euclidean` | `euclidean` or `cosine` |
| `seed` | random | Reproducibility |
| `weightCol` | none | Row weights |
| `featuresCol`, `predictionCol` | `features`, `prediction` | Column names |

> `distanceMeasure="cosine"` is useful for text vectors, where direction matters more than length.

---

### 18. Strengths and weaknesses

| Strengths | Weaknesses |
| --- | --- |
| **Simple** and easy to understand | You must **choose K** in advance |
| **Fast** and scalable (parallel in Spark) | Sensitive to **initialization** (local minima) |
| Works well on compact, round clusters | Assumes **spherical, similar-sized** clusters |
| Output (centroids) is easy to interpret | Sensitive to **outliers** (they pull the means) |
| One of the most used unsupervised methods | Needs **numeric**, **scaled** features |
| | Fails on elongated or ring-shaped clusters |

**Alternatives in Spark:** `BisectingKMeans` (hierarchical splitting), `GaussianMixture` (soft, elliptical clusters), `LDA` (topic modelling).

---

### 19. Common errors

| Symptom | Cause | Fix |
| --- | --- | --- |
| Clusters dominated by one feature | Unscaled features | `StandardScaler` |
| Different clusters each run | No seed / random init | Set `seed`; use k-means|| |
| Cluster numbers change between runs | Labels are arbitrary | Interpret by centroids, not numbers |
| `Column features must be of type VectorUDT` | No assembler | `VectorAssembler` |
| WCSS keeps falling and you pick a huge K | WCSS always falls | Use elbow/silhouette, not the lowest WCSS |
| Poor clusters with outliers | Means are pulled by outliers | Remove/cap outliers |
| Using a label column as a feature | Defeats unsupervised learning | Leave labels out |
| `ClusteringEvaluator` error on columns | Wrong column names | Set `predictionCol`, `featuresCol` |
| Categorical text in features | K-means needs numbers | One-hot encode (or use another method) |
| Expecting "accuracy" | There are no labels | Use WSSSE and silhouette |

---

### 20. Phase 8 — practical scenarios

1. Compute the **Euclidean distance** between (1, 1) and (5, 7).
2. Perform **one full K-means iteration by hand** for the 7 points with starting centroids A and D.
3. Compute the **WCSS** of the final clusters by hand (8.525).
4. Run the 7-point example in **Spark** and compare centres and cost.
5. Compute the **silhouette of point A** by hand and compare with 0.9534.
6. Cluster the **16 customers** with K = 4; print centres and sizes.
7. Produce the **elbow table** for K = 2 to 6 and pick K.
8. Compute the **silhouette** for each K and pick K; do both methods agree?
9. **Name the four segments** and propose a marketing action for each.
10. Multiply income by 1000 and show how **unscaled** K-means fails; fix it with `StandardScaler`.
11. **Predict** the cluster of a new customer (₹25k, score 80).
12. Compare **`initMode="random"`** with the default and with different seeds.

---

### 21. Phase 8 — viva questions and answers

#### Q1. What is clustering and what kind of learning is it?

Grouping similar data points together without labels; it is unsupervised learning.

#### Q2. What is K-means?

An algorithm that partitions data into K clusters, each represented by the mean (centroid) of its points, by iteratively assigning points to the nearest centroid and recomputing centroids.

#### Q3. What is a centroid?

The mean position of all points in a cluster.

#### Q4. Describe the steps of the K-means algorithm.

Choose K, initialize K centroids, assign each point to the nearest centroid, update each centroid to the mean of its points, and repeat until assignments stop changing.

#### Q5. What distance does K-means use?

Euclidean distance (squared Euclidean in the objective); cosine is also available in Spark.

#### Q6. What is the objective function of K-means?

Minimize the within-cluster sum of squares (WCSS): the sum of squared distances of points to their cluster centroids.

#### Q7. Does K-means always converge? Does it find the global minimum?

It always converges, but only to a local minimum that depends on initialization.

#### Q8. Why does the mean update lower WCSS?

The mean is the point that minimizes the sum of squared distances to a set of points, and nearest-centroid assignment can only reduce distances.

#### Q9. What is k-means++ and what does Spark use?

A smarter initialization that spreads starting centres using squared-distance weighting; Spark uses k-means||, its parallel version, by default.

#### Q10. How do you choose K?

With the elbow method (look for the bend in WCSS vs K), the silhouette score (highest is best), and domain knowledge.

#### Q11. Explain the elbow method.

Plot WCSS against K; WCSS always decreases, but beyond the right K the improvement is small, producing a bend (elbow).

#### Q12. What is the silhouette score and its range?

`s = (b − a) / max(a, b)`, averaged over points, where a is the mean distance to the own cluster and b to the nearest other cluster; range −1 to +1, higher is better.

#### Q13. Why must features be scaled before K-means?

Because it is distance-based; a feature with a large range would dominate the distance.

#### Q14. How does K-means treat outliers?

Poorly: they pull centroids, because means are sensitive to extreme values.

#### Q15. What assumptions does K-means make about clusters?

That they are roughly spherical, similar in size and well separated.

#### Q16. Which Spark attribute gives WCSS?

`model.summary.trainingCost`.

#### Q17. How can you evaluate clustering when there are no labels?

Use internal measures such as WCSS and the silhouette score (ClusteringEvaluator).

#### Q18. What does `model.predict(vector)` do?

Assigns the point to the nearest cluster centre.

#### Q19. Why are cluster numbers (0, 1, 2…) arbitrary?

K-means has no inherent ordering of clusters; meaning must be derived from the centres.

#### Q20. In our customer example, which K was best and why?

K = 4: the WSSSE dropped sharply to 271.8 (the elbow) and the silhouette peaked at 0.988.

---

### 22. Phase 8 — final master recap

```text
K-MEANS      unsupervised clustering into K groups ; each cluster = its centroid (mean)
DISTANCE     d(p,q) = √Σ(pᵢ − qᵢ)²        (A→C: √13 = 3.606)
OBJECTIVE    WCSS = Σ_clusters Σ_points ‖x − μ‖²      (Spark: trainingCost)
ALGORITHM    init K centroids → [ assign to nearest → recompute means ] → until no change
HAND RUN     7 points: WCSS 12.208 → 8.525 ; centroids (1.25,1.5) & (3.9,5.1) ; Spark cost 8.525
INIT         random | k-means++ | k-means|| (Spark default, parallel k-means++)
CHOOSING K   elbow (bend in WCSS)  +  silhouette s=(b−a)/max(a,b) (higher is better)
CUSTOMERS    K=4 : WSSSE 271.8 (elbow), silhouette 0.988 ; segments Budget/Cautious/Spenders/Premium
SCALING      unscaled sizes [5,6,3,2] ✗  →  scaled [4,4,4,4] ✓
PYSPARK      KMeans(k, seed, maxIter).fit(df) → centers · transform · summary.trainingCost · ClusteringEvaluator
```

**Seven facts to memorise**

1. K-means is **unsupervised**: no labels, you choose **K**.
2. Two repeating steps: **assign to nearest centroid**, **recompute centroid = mean**.
3. It minimizes **WCSS** and always converges (to a **local** minimum).
4. Spark initializes with **k-means||** (parallel k-means++).
5. Pick K using the **elbow** and the **silhouette** (higher = better).
6. **Scale** features first (distance-based).
7. Cluster numbers are arbitrary; **interpret clusters by their centres**.

### Phase 8 completed

---

> **Next → Phase 9: Support Vector Machine (SVM)** (maximum-margin classifier, hyperplane and margin geometry, hinge loss and soft margin, the C / regParam trade-off, the dual and kernel idea, `LinearSVC` in Spark, One-vs-Rest for multiclass, scaling, and worked numbers). File: `9_Unit6_SVM.md`
