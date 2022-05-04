---
layout: post
title: High Performance Programming with MPI and OpenMP
date: 2022-5-4
description: Parallel programming and high performance programming has been the way people trying to enhance computer performance after the development of hardware slows down. The major two systems researchers have been using are MPI and OpenMP. This spring, I had the chance to explore both of them and worked on a research project to perform a histogramming sort among distributed memories.
tags: [technology, computer science, article]
---

During the past six months, I have been learning high performance programming and parallel programming with MPI and OpenMP. It is always a fascinating process whenever I have the chance to learn something new. I arrived in this course with barely any knowledge about parallel programming, and comes out with the ability to do research on high performance programming analysis. This might be the beauty of going to schools, you never know what you don’t know until you know.

## The Raise of Distributed Memory

Among recent years, distributed memory has evolved quickly as the development of hardware is reaching its own bottleneck. People need more powerful and faster computers, but no one wants a computer with bigger size than they have already had. As smaller and smaller hardware components go, there it comes to a limit where nothing can be ever smaller to be put together without burning the whole chip down. Therefore, researchers start finding another angle to improve the overall performance. The question is, how can we enhance the performance given the hardware we currently have? The answer has to be within the theoretical and mathematical part of computer science. 

Currently, we have multi-cores personal computers available for purchase everywhere, but we barely use all cores together. Thus, the issue is how we can achieve better performance by utilizing all of its cores’ computing power together on one task. We used to have everything been executed in a sequential order, which we call it sequential programming. In sequential programming, everything works one by one according to the line of execution order. In this case, we ignore the power that multi cores can in fact run altogether for a single process and produce a correct result[^1] in the end. This possibility will increase the overall performance since we distribute the work among all processors to make them work altogether and faster. 

However, if we distribute the work among all processors, they would have to know each other’s progress to work together properly and correctly. For example, as we see the code below, if we loop through an array and sum up each entry together to a variable called `total_sum`, it is fairly easy to get it correct under sequential programming where we just add each element one by one. This is not the case in parallel programming. 


```c
int total_sum = 0;
for (int i = 0; i < arr_len; i++) {
    total_sum += arr[i];
}
// this will not print out the correct sum in parallel programming
printf(“the total sum is: %d\n”, total_sum); 
```


Under parallel programming, each processor is running this code block independently, and they will read and write the global variable `total_sum` at anytime. When some processors access `total_sum` for the first time, the value stored in it might have been altered by some other processors; and when they are trying to write it back, they might overwrite the result produced by other processors. Thus, we need a way to have our processors communicate with each other and sync together properly. This is where MPI and OpenMP come in to play. 

## MPI and OpenMP

By combining MPI and OpenMP, we can fully utilize the power of supercomputers clusters. Generally, MPI will be in charge of the communication between ranks, and OpenMP will utilize all cores within each rank to work together. There are three major ways for MPI ranks to communicate together, point-to-point communication, one-side messaging, and all-to-all collective. Point-to-point communication happens between two specific ranks and uses `MPI_Send/Recv`. One-side messaging, `MPI_Put/Get`, lets the receiver open a window which works as a shared memory slot where data can be put into that window by other ranks. All-to-all collective, such as `MPI_Bcast` and `MPI_Allreduce`, works among all ranks within a communicator where each rank will get the same result stored after the MPI call. OpenMP is a more robust method comparing to MPI. It works well for loops and small tasks within each rank of MPI to achieve one single task among multi cores. A great example for hybridizing these two powerful tools is the final research project I finished for this course, a histogramming sort[^2] within distributed memory environment. 

## Histogramming Sort in Parallel

Sorting is comparable hard under distributed memory considering we have to globally sort all data entries. A histogramming sort targets against this problem. It has three phases, rebalance, histogramming, and move data. The general idea for this algorithm is distributing all data entries among all processors, then find the splitters for each histogram bar, in the end move data to the correct processor. 


### Rebalance

The first phase, rebalance, will evenly redistribute all data points among all processors. If one processor has more data entries than the average, it will move data out. On the other hand, it will take data in if it has less than average data entries. MPI all-to-all is used to get the total data entries among all processors, and MPI one-side messaging is used to transfer data. The reason we have to rebalance before histogramming is simple, we want each processor to work on roughly same amount of data during histogramming phase. In this phase, data is not globally sorted, but locally sorted on each of processor at the end. 


```c
void rebalance(const dist_sort_t *const data, const dist_sort_size_t myDataCount,
    dist_sort_t **rebalancedData, dist_sort_size_t *rCount) {

	// find total data size
	MPI_Allreduce(...);

	// find average and error tolerance
	dist_sort_t avg = total_data / nprocs;
	dist_sort_t upper_bound = ...
	dist_sort_t lower_bound = ...

	// span window
	MPI_Win_creat(...);

	// rebalance process
	if (myDataCount > upper_bound) {
		MPI_Put(...);
	}

	// prep the result array
	*rebalancedData = (dist_sort_t *) malloc(...);

	// get the data
	for (int i = 0; i < win_size; i++) {
		(*rebalancedData)[i] = ...
	}
}
```

### Histogramming

In the second phase, histogramming, we find the correct set of splitters for each bar of our histogram. We perform a binary search for each bar and reduce the searching range by half during each iteration. `MPI_Allreduce` is used to gather the global count for each histogram bar through every iteration. A while loop is introduced for iterations and will be broken once the rank 0 gets the desired splitters set and broadcasts the stop sign. In the end, rank 0 will broadcast the count of each histogram bar and all splitters to all other ranks using `MPI_Bcast`.

```c
void findSplitters(const dist_sort_t *const data, const dist_sort_size_t data_size,
    dist_sort_t *splitters, dist_sort_size_t *counts, const int numSplitters) {
	
	while(1) {
		// get count for each histogram bar from each rank
		MPI_Allreduce(...);
		
		// check and move each bar
		for (int i = 0; i < numSplitters; i++) {
			if (counts[i] > upper_bound) {
				// move to left
			} else if (counts[i] < lower_bound) {
				// move to right
			}
		}

		// rank 0 broadcast stop signal
		MPI_Bcast(...);
		if (...) {
			break;
		}
	}

}

```

### Move Data

The final phase, move data, where all data is moved, is the major work we have to perform for this sorting algorithm. So far, we have found the correct splitters for each histogram bar, then we can use this information to identify whether the data entry in current rank should stay or leave. If it stays, we write it to the output array, and send it out to the correct rank otherwise. Here, we have three options to move the data, either point-to-point `MPI_Send/Recv`, or all-to-all collective, or one-side messaging `MPI_Put/Get`. I first used point-to-point `MPI_Send/Recv` since it is annoying to calculate the displacement for `MPI_Put/Get`. Then, I found out that since our data size is too big, sending everything needs to go into one rank will overload the `MPI_Send/Recv` process. There comes two ways to resolve this, either perform a segmentation or move to other methods. I decided to move to one-side messaging since segmentation relays too much on each different MPI setting. Here, each rank spans a window, and data is put into this window from other ranks if necessary similar to what happened in the rebalance phase. 

```c
void moveData(const dist_sort_t *const sendData, const dist_sort_size_t sDataCount,
    dist_sort_t **recvData, dist_sort_size_t *rDataCount,
    const dist_sort_t *const splitters, const dist_sort_t *const counts, const int numSplitters) {
	
	// span window
	MPI_Win_creat(...);

	// loop through all data entry
	for (int i = 0; i < sDataCount; i++) {
		if(sendData[i] within bar) {
			// write to result array
			(*recvData)[*rDataCount] = sendData[i];
		} else {
			// find the correct rank
			for (int j ... ) {
				if (j is the correct rank) {
					// send to rank j
					MPI_Put(...);
				}
			}
		}
	}

	// receive data from window
	for (int i ...) {
		(*recvData)[*rDataCount] = ...;
	}
}
```

The overall framework is set by MPI, and OpenMP comes into dividing same job into different cores to improve the performance. 

## The Future of High Performance Computing

Generally speaking, high performance programming and parallel programming is the way we will go in the future, but I am afraid there is still a lot of work to be done in this field to even full utilize the power we have right now. Considering the hardware development has not slower down dramatically, we probably will have to wait for several years until major research attention is drawn to this area of computing.  

[^1]: A correct result in parallel programming means it produces the identical result as if it runs under sequential programming.

[^2]: General idea in this [paper](https://charm.cs.illinois.edu/newPapers/93-01/paper.pdf) and optimization in this [paper](https://arxiv.org/pdf/1803.01237.pdf).

---






