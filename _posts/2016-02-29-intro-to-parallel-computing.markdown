---
layout: post
title:  "A Talk on Parallel Computing to the Biostatisticians"
date:   2016-02-29 18:36:18 -0500
categories: talk
---

So I was asked to give a small introductory talk on parallel computing 
technologies to a few biostatisticians enrolled in BSTA 787 at 
University of Pennsylvania. [Here](https://penno365-my.sharepoint.com/personal/jiacheng_upenn_edu/_layouts/15/guestaccess.aspx?guestaccesstoken=imk6OhLxPqy3%2fnrySPNOzEMp%2fWOHN5RNpWiz7djjnh8%3d&docid=08f456f8f4e4f461ba0d05e53309b9240) is the slides I used for this talk. 
I used several pieces of code to showcase different 
parallel computing technologies, openMP, openMPI and Apache Spark. The code
is also published here on this page.
Some of the slides were borrowed from the lecture notes of CIS 505 by Dr. 
Boon Thau Loo, a class I thoroughly enjoyed and would recommend to anyone.
Some of the concepts in the talk will be egregiously inaccurate (or even wrong)
in order to make them approachable/understandable for those who do not
have a solid CS background.
 

The following Java code was used to showcase the interweaving of execution 
by different threads.
{% highlight java %}
public class Main {
    static class HelloThread extends Thread {
        public void run() {
            for (int i = 0; i < 10; i++) {
                System.out.print(Integer.toString(i)+"\t");
            }
        }
    }

    public static void main(String args[]) {
    
        int numRepeats = 5;
        // first print sequentially
        for (int i = 0; i < numRepeats; i++) {
            for (int j = 0; j < 10; j++) {
                System.out.print(Integer.toString(j)+"\t");
            }
        }

        System.out.println();

        // then print concurrently
        Thread[] pool = new Thread[numRepeats];
        for (int i = 0; i < numRepeats; i++) {
            pool[i] = new HelloThread();
        }

        for (int i = 0; i < numRepeats; i++) {
            pool[i].start();
        }
        
    }
}
{% endhighlight %}


The following C/C++ code was used to showcase the openMP technology.
{% highlight cpp %}
#include <iostream>
#include <omp.h>
#include <cmath>
#include <vector>

bool is_prime( long p ){
    if( p == 2 ) return true;
    else if( p <= 1 || p % 2 == 0 ) return false;
    else {
        bool prime = true;
        const int to = sqrt(p);
        int i;
        for(i = 3; i <= to; i+=2)
            if (!(prime = p % i))break;
        return prime;
    }
}

int main(){

    long max_size = 10000000;
    std::vector<long> nums(max_size);

    for (int j = 0; j < max_size; ++j) {
        nums[j] = j+1;
    }

    omp_set_num_threads(8);

#pragma omp parallel for
    for (long i=1; i<max_size; i++){
        is_prime(nums[i]);
    }

    return 0;
}
{% endhighlight %}

The following C/C++ code was used to showcase the openMPI technology.
{% highlight cpp %}
#include <iostream>
#include <cmath>
#include <vector>
#include <mpi.h>

bool is_prime( long p ){
    if( p == 2 ) return true;
    else if( p <= 1 || p % 2 == 0 ) return false;
    else {
        bool prime = true;
        const long to = sqrt(p);
        int i;
        for(i = 3; i <= to; i+=2)
            if (!(prime = p % i))break;
        return prime;
    }
}

long *create_sequence(size_t size){
    long *seq = (long *) malloc(sizeof(long)*size);
    for (int i = 0; i < size; ++i) {
        seq[i] = i+1;
    }
    return seq;
}

bool *compute_primality(long * values, size_t size){
    bool *results = (bool*) malloc(sizeof(bool) *size);
    for (int i = 0; i < size; ++i) {
        long val = *(values+i);
        results[i] = is_prime(val);
    }
    return results;
}

int main() {
    long *all_integers = NULL;

    // Initialize the MPI environment
    MPI_Init(NULL, NULL);

    // Get the number of processes
    int world_size;
    MPI_Comm_size(MPI_COMM_WORLD, &world_size);

    // Get the rank of the process
    int world_rank;
    MPI_Comm_rank(MPI_COMM_WORLD, &world_rank);

    int elements_per_proc = 10000000;
    if (world_rank == 0) {
        all_integers = create_sequence(elements_per_proc * world_size);
    }

// Create a buffer that will hold a subset of the passed sequence
    long *subseq_seq = (long *) malloc(sizeof(long) * elements_per_proc);

// Scatter the sequence to all processes
    MPI_Scatter(all_integers, elements_per_proc, MPI_LONG, subseq_seq,
                elements_per_proc, MPI_LONG, 0, MPI_COMM_WORLD);

// Compute the primality of your subset
    bool * sub_primality = compute_primality(subseq_seq, elements_per_proc);

// Gather all primality to root process
    bool * all_primality = NULL;
    if (world_rank == 0) {
        all_primality = (bool *) malloc(sizeof(bool) * world_size*elements_per_proc);
    }
    MPI_Gather(sub_primality, elements_per_proc, MPI_C_BOOL, all_primality, 
    elements_per_proc, MPI_C_BOOL, 0, MPI_COMM_WORLD);

// Print the prime numbers found
    if (world_rank == 0) {
        for (int i = 0; i < world_size*elements_per_proc; ++i) {
            if (all_primality[i]){
                std::cout << (i+1) << ",";
            }
        }
    }

    // Finalize the MPI environment.
    MPI_Finalize();

    return 0;

}
{% endhighlight %}

**Attention statisticians, the following message is critical to ALL OF YOU
if you ever want to perform parallel computing using R.**

So many statisticians nowadays are trying to perform parallel computing with R,
using the bajillion parallel packages out there for R. Off the top of my head,
doParallel, doMC, doSNOW, parallel, snow, multicore, doMPI... There are two things
statisticians almost always get confused about.

1. What package to use under what circumstances.
2. What to set n.cores.

These two questions are not that easily answered without a bit of knowledge in 
distributed systems. Unfortunately as statisticians, many do not wish to know the nitty-gritty
details of the distributed systems. I will hereby give some rules of thumb to 
question 1, and some **really important** guidelines for question 2.

1. What package to use under what circumstances.

   All R parallel packages use process-level parallelism. Some packages
   such as `doMC` and `multicore` use the "fork" system call on *NIX systems to make duplicates
   of the current R processes. Sometimes, this exhausts the memory fairly quickly,
   especially if you have loaded some large R objects. But it is a very efficient
   way of executing in parallel if you are running an R process on a *NIX system.
   On Windows machines, `doMC` or `multicore` will not run in parallel (because Windows
   does not provide the "fork" system call). In this case, you should run doSNOW, snow 
   or doParalle, which uses doSNOW by default. doSNOW uses three different interfaces
   to communicate with its "workers", PVM (Parallel Virtual Machine), MPI(Message Passing Interface)
   or raw "sockets". They should in theory work in all operating systems, and 
   across different nodes on a computing cluster, if PVM or MPI is available.
   doMPI is a relatively new package using the Rmpi interface for parallelization
   with MPI. I am not particularly sure about the differences between doMPI and doSNOW 
   when it comes to MPI, since both use Rmpi as their back end.
   The rule of thumb is, **if you want to work under Windows, use doParallel.
   If you want to work on a computing cluster, use doMPI.
   If you want to work on a *NIX machine (Mac, Linux, FreeBSD), use doMC.**
   
2. How many cores to use.

   I have used some packages that handle this issue very poorly. 
   For example, one package sets default of n.cores = 12. This will inevitably
   slows down any computer with less than 12 cores to a grinding halt. Some packages
   automatically detect the number of cores of the machine and set the n.cores parameter
   to be that number, as if there would never be two copies of the same function
   being executed simultaneously. Neither of the above practices are desirable. 
   What should be done is this, whenever you write a function in a package that
   may require parallel computing, **ask the user for the number of cores to use, set
   the default n.cores = 1, and do your best to honor the n.cores parameter.**

The following code was used to showcase the doParallel/doMPI package and foreach pacakge.

{% highlight R %}
library(doParallel)

# create the dataset
data <- seq(-10,10,length.out = 1000)

# initialize the cluster environment
cl <- makeCluster(4)
registerDoParallel(cl)

# perform the computation, notice the .combine option and .packages option. 
# .combine tells foreach how to combine the results. In this case it's c().
# .packages tells foreach what packages are used in the parenthesized code. 
results <- foreach(i=data, .combine = c, .packages = c('boot')) %dopar% {
  inv.logit(i)
}

# stop the cluster
stopCluster(cl)

# plot the results
plot(data,results,type='l',xlab='X',ylab='Y')

{% endhighlight %}

Similar code can be used for doMPI and foreach.
{% highlight R %}
data <- seq(-10,10,length.out = 1000)

library(doMPI)
library(foreach)

cl <- startMPIcluster(count = 4)
registerDoMPI(cl)

results <- foreach(i=data, .combine = c, .packages = c('boot')) %dopar% {
  inv.logit(i)
}

closeCluster(cl)

plot(data,results,type='l',xlab='X',ylab='Y')
{% endhighlight %}

If you ever need to write some parallel R code, just copy and paste the above
code and replace the body of the foreach function with the computations
of your choosing. Make sure to set up `.combine` and `.packages` parameters
correctly.

The following code was used to showcase the Apache Spark functions.

{% highlight scala %}
object summaryStatistics {
  def main(args: Array[String]) {
    import org.apache.spark.SparkConf
    import org.apache.spark.SparkContext
    import org.apache.spark.sql.SQLContext
    import org.apache.spark.mllib.stat.Statistics
    import org.apache.spark.storage.StorageLevel

    val conf = new SparkConf().setAppName("summaryStats").setMaster("local[4]")
    val sc = new SparkContext(conf)
    val sqlContext = new SQLContext(sc)

    val yelpRev = sqlContext.read.json("yelp_academic_dataset_review.json")
    val reviewLengths = yelpRev.map(x => x.getString(4).split("\\W+")
      .length.toDouble).persist(StorageLevel.MEMORY_ONLY)

    yelpRev.describe("stars").show()
    println(reviewLengths.mean())
    println(reviewLengths.stdev())
    println(Statistics.corr(yelpRev.map(x => x.getLong(3).toDouble), 
      reviewLengths, 
      "spearman"))
  }
}
{% endhighlight %}

How to set up an IDE for Spark development and the link to download the yelp review data can be found in [this post]({% post_url 2016-02-23-how-to-setup-intellij-for-spark%})
