# The Correct Way to Calculate Variance and how Modern Database Systems do it Wrong

## Abstract
Here, we summarize our [SIGMOD RECORD 2016 paper](https://sigmodrecord.org/2017/02/07/a-closer-look-at-variance-implementations-in-modern-database-systems/) on the correct way to calculate variance and the different incorrect ones modern databases employ.

TL;DR: To calculate variance, use the old-fashioned *Two Pass* formula, if performing two passes over the data is feasible -- not the fancy one pass ones. To elaborate, if the entire dataset can fit in memory, use *Two Pass*. 
On the other hand, if we are considering a streaming environment, or if the cost of loading the data in memory in the second pass is high, then use one of the one-pass algorithms as detailed in the last paragraph of this blog.

# Introduction
Variance is a popular and often necessary component of
aggregation queries. It is typically used as a secondary
measure to ascertain statistical properties of the result
such as its error. Yet, it is more expensive to compute
than primary measures such as SUM, MEAN, and COUNT.

There exist numerous techniques to compute variance.
The definition of variance implies two passes over
the data, with the first pass calculating the sample mean.
This technique is known to provide the highest numerical precision.
However, as performing two passes was deemed too expensive, numerous complex one pass algorithms were devised.
The assumption was that reducing the number of passes over the data would reduce the computational cost as well.
However, we show in this paper that this assumption is false -- the *Two Pass* technique results in the lowest speed, with the exception of Textbook One Pass, which is highly susceptible to precision loss resulting from *catastrophic cancellation*.

We study variance implementations in
various real-world systems and find that major database
systems such as PostgreSQL and most likely System X,
a major commercial closed-source database, use Textbook One Pass.
We review literature over the past five decades on variance
calculation in both the statistics and database communities.
Interestingly, we recommend using the mathematical
formula for computing variance if two passes
over the data are acceptable due to its precision, parallelizability,
and surprisingly computation speed.

## Contributions and Outline
1. We catalog usage of different variance formulas
in various open source database systems.
2. We experiment with different closed source and
open source databases to investigate precision loss
issues. We find that precision of PostgreSQL and
System X deteriorates the most. After looking at
the PostgreSQL source code, we can verify that it
uses Textbook One Pass, and hypothesize that System
X does so as well.
3. We empirically study the accuracy of the different
representations under varying additive shifts
and dataset sizes including a hitherto unstudied one,
which we call Total Variance.
4. We recommend using Two Pass if performing
two passes over the data is acceptable,
which seems counter-intuitive, but works due to its
computational simplicity.

# Various Variance Formulations
![formulas](https://github.com/NiranjanKamat/blogs/blob/master/variance_precision/variance_formulas.png)

Table 1 presents the different variance formulas commonly used. *S* represents the sum of squares. Sample variance can be obtained by dividing it by *(n - 1)*, where *n* is the sample size.

The accuracy of Shifted One Pass depends on
that of the mean estimate. Pairwise Updating is the
only representation providing accurate results while
being highly parallelizable and requiring a single
pass. Additionally, experiments show that the
precision of Total Variance is slightly better than
that of Updating Pairwise, which has the best precision
amongst all single pass algorithms. 
Kahan summation
[5, 10] can further improve their precision.
Note that Table
2.1 of [1] succinctly enumerates the theoretical error bounds
of different formulations.

# Variance Implementations in Modern Database Systems
We looked at the code of multiple open source
databases to find their variance representations. We
also conjecture about two closed source ones through
our experiments.

<a href="url"><img src="https://github.com/NiranjanKamat/blogs/blob/master/variance_precision/database.png" width="400" ></a>

PostgreSQL uses Textbook One Pass and is thus
susceptible to precision loss. MySQL uses Knuth’s
modification [8] of Welford’s updating formula. Therefore,
it can only process a single additional data
point, and cannot avail of the possible parallelization.
Spark 1.4.1 and Impala 2.1.5 use a modified version of Updating Pairwise.
Although the source code for System X is not
available, we conjecture that it uses Textbook One
Pass as its precision behavior was similar to that
of PostgreSQL. System Y was found to have the
best precision. We hypothesize that it uses higher
precision variables, but cannot make any conjecture
about the exact representation.

# Experiments
We 
present the execution times of different algorithms
on data sizes up to 100 million tuples. The results
are the average over 100 runs. Experiments were
performed using Ubuntu 14.04.05 LTS with a 4 core,
2.4 GHz Intel CPU, with 16 GB RAM, and 256 GB
SSD storage, using a single execution thread.

In a similar vein as Tian et al. [10], we
created synthetic datasets of different sizes using
Uniform(0, 1). They were shifted
by adding values ranging from 10<sup>1</sup> to 10<sup>15</sup>.

![results](https://github.com/NiranjanKamat/blogs/blob/master/variance_precision/results.png)

## Impact of Shift
Numerical precision was evaluated using varying
additive shifts, over a dataset of size 10000 (Fig a).
With increasing shift exponent, all representations 
experience precision loss,
with Two Pass having the best precision,
and Textbook One Pass the worst.



## Impact of Data Size
Since precision errors typically accumulate, we
tried datasets of sizes from 10 to 100 million (Fig b). The
shift was set at 10<sup>5</sup>. We can see that precision
generally worsens with increasing data size. Two
Pass again outperforms other algorithms. Textbook
One Pass consistently exhibits the worst precision.

## Execution Speed
We also looked at the execution time of different
algorithms with increasing data size (Fig c).
Results with lower data sizes have not been presented
due to the computation taking minimal time.
Surprisingly, there was no discernible difference in
execution time between Two Pass and Shifted One
Pass. Only Textbook One Pass took lesser time
than Two Pass. We attribute the low execution
time of Two Pass to simplicity of its computation, 
allowing effective use of CPU caches and pipelining.

## Impact of Shift on Databases
We look at
variance precision
for the different
databases under
varying additive
shifts. We took
efforts to ensure
different systems
have similar data
types. 100 points
were chosen from a Uniform(0, 1) distribution.
Precision loss follows a similar
pattern in System X and PostgreSQL. Impala
and MySQL have a similar error profile as well.

<a href="url"><img src="https://github.com/NiranjanKamat/blogs/blob/master/variance_precision/real_database_accuracies.png" width="350" ></a>

# Conclusion & Recommendattions
Precision issues associated with Textbook One Pass
have been well documented. However, we have seen
that databases such as PostgreSQL and likely System
X still use it. We recommend from the perspective
of safety to discontinue its usage. Though
there might be arguments for its continued usage
after warning the users in certain scenarios, the arguments
against it far outweigh the speedup benefit and its ease of implementation. Although error
inherently exists in approximate query processing,
numerical precision errors are easy to eliminate and
hard to apportion and therefore should be avoided
whenever possible. Hence, we recommend to the
designers of databases, and statistics and analytics
packages, to discontinue its usage.

Previous work has recommended Pairwise Updating
from the perspective of precision, speed, and
parallelizability [1]. However, we have seen from
our experiments of up to 100 million data points,
that the most accurate algorithm, Two Pass, takes
lesser time than Updating, Updating Pairwise, and
Total Variance. Two Pass is also easy to implement
and parallelize. Therefore, in the case that
performing two passes over the data is acceptable,
Two Pass should be the preferred
algorithm. Determining whether two passes are
acceptable, however, is a nuanced decision. When
the data fits in memory, performing two passes over
the data is clearly acceptable as all representations
will incur the identical data read I/O cost. When
the data cannot fit in memory, summing up the estimated
I/O and computation times can help determine
whether Two Pass will need the least amount
of time.

In other cases, i.e., whenever Two Pass is
not estimated to require the least execution
time or we are considering a streaming environment, there does not exist a clear winner, due
to different algorithms having different strengths
and weaknesses. Updating provides faster results
at lower precision, compared with Updating Pairwise,
without needing additional memory. Updating
Pairwise is parallelizable, whereas Updating is not.
While Shifted One Pass is fast, its
accuracy is dependent on correctness of the mean 
estimate. Total Variance has good accuracy, although
it takes longer to execute, and is dependent
on the algorithm used to compute group statistics,
while also needing multiple passes. Hence, there
does not exist any algorithm that dominates every
other algorithm. A query planner that devises
hybrid formulas, while taking the data distribution,
estimated I/O and computation costs, and the overall
strengths and weaknesses of different algorithms
into consideration, appears to be an important and
ideal piece of future work.

# References
[1] Tony Chan, Gene Golub, Randall LeVeque. Algorithms for Computing the
Sample Variance: Analysis and
Recommendations. Am. Stat., 1983. http://www.dtic.mil/dtic/tr/fulltext/u2/a133112.pdf

[2] Richard. J. Hanson. Stably Updating Mean and
Standard Deviation of Data. ACM, 1975. https://dl.acm.org/citation.cfm?id=360662

[3] Joseph M. Hellerstein, Christoper R{\'e}, Florian Schoppmann, Daisy Zhe Wang, Eugene Fratkin, Aleksander Gorajek, Kee Siong Ng, Caleb Welton, Xixuan Feng, Kun Li, Arun Kumar. The MADlib Analytics Library: or
MAD Skills, the SQL. VLDB, 2012. https://arxiv.org/pdf/1208.4165.pdf

[4] Nicholas. J. Higham. Accuracy and Stability of
Numerical Algorithms. SIAM, 2002. https://dl.acm.org/citation.cfm?id=579525

[5] William Kahan. Further Remarks on Reducing
Truncation Errors. ACM, 8(1):40, 1965. https://dl.acm.org/citation.cfm?id=579525

[6] Niranjan Kamat, Arnab Nandi. A Closer Look at
Variance Implementations In Modern
Database Systems. Arxiv TR, 2016. https://arxiv.org/abs/1509.04349

[7] Sean Kandel, Ravi Parikh, Andreas Paepcke, Joseph Hellerstein, Jeffrey Heer. Profiler: Integrated Statistical
Analysis and Visualization for Data Quality
Assessment. AVI, 2012. http://vis.stanford.edu/files/2012-Profiler-AVI.pdf

[8] Donald. E. Knuth. Art of Computer Programming,
Volume 2: Seminumerical Algorithms. 2014. https://dl.acm.org/citation.cfm?id=270146

[9] Janet Rogers, James Filliben, Lisa Gill, William Guthrie, Eric Lagergren, Mark Vangel. StRD: Statistical
Reference Datasets for Testing the Numerical
Accuracy of Statistical Software, 1998. https://www.nist.gov/itl/products-services/statistical-reference-datasets

[10] Yuanyuan Tian, Shirish Tatikonda, and Berthold Reinwald.
Scalable and Numerically Stable Descriptive
Statistics in SystemML. ICDE, 2012. http://ieeexplore.ieee.org/document/6228204/

[11] B. Welford. Note on a Method for Calculating
Corrected Sums of Squares and Products.
Technometrics, 1962. http://amstat.tandfonline.com/doi/abs/10.1080/00401706.1962.10490022#.WiA4prynHCI

[12] D. West. Updating Mean and Variance
Estimates: An Improved Method. 1979. https://dl.acm.org/citation.cfm?id=359153

[13] Edward A. Youngs, Elliot M. Cramer. Some Results Relevant to
Choice of Sum and Sum-of-Product
Algorithms. Technometrics, 1971. http://www.jstor.org/stable/1267176?seq=1#page_scan_tab_contents
