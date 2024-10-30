# need-for-speed

Here is a summary of the discussion we had (with [Hanno](https://github.com/HHildenbrandt) and others) after my talk on how to use the local storage on a high performance computer cluster (here, the [Habrok](https://www.rug.nl/society-business/center-for-information-technology/research/services/hpc/facilities/habrok-hpc-cluster?lang=en) cluster) handle when handling many simulations and many files (30th of October 2024, [TRÊS](https://www.rug.nl/research/gelifes/tres/?lang=en) lab meeting, University of Groningen).

Basically, what I have now is a pipeline where I launch several batches of simulations that run sequentially within a batch, as part of a single job (see [this](https://github.com/rscherrer/reschoice-data) --- private --- repository as an example). I make sure that the output of those simulations is saved in the local storage, to avoid having to write (at every saved time step) back into the shared storage, which is expensive. Only when the batch of sequential simulations is done do I compress the various outputs into an archive and copy that single file back to the shared storage. This allowed me to (1) no longer reach the maximum number of jobs I can submit per day (which I did when running all of my simulations as separated jobs instead of batches), (2) no longer reach the disk quota in terms of number of files handled (by only copying one file between disks), and (3) to save a lot of time (not writing so regularly to the shared storage but into the local storage instead allowed me to reduce my total used time by ten fold). This pipeline came about by following the instructions on [this]([https](https://wiki.hpc.rug.nl/habrok/advanced_job_management/many_file_jobs)) page of the Habrok documentation, kindly shared by [Pedro Neves](https://github.com/Neves-P).

That said, we can still do better. Here are a few ways how.

## 0. Binary output

The use of binary data is good. I am writing this as step 0 because I am already doing it (see e.g. [here](https://github.com/rscherrer/speciome) or [here](https://github.com/rscherrer/brachypode)), I just wanted here to re-iterate why it is a good idea. Saving data as text is very intensive, especially when many decimals have to be written. Instead, writing values as they are (i.e. binary numbers) is much more compact, takes a lot less space and is very fast to write and read back. The downside is that the data saved cannot be read by a human, and so we need to provide information for how to read that data: type of encoding (how many bytes make up one value), and byte order (in which direction should the string of bytes be read --- either little endian or big endian). This means that we often need to provide tools to allow the user to read back the simulation data (e.g. some R or python script or, in my case, an R package that comes with the simulation program, such as [this one](https://github.com/rscherrer/speciomer) or [that one](https://github.com/rscherrer/brachypoder)). 

**Note:** Hanno says that if files are small, there is no need to go for binary. However, I still prefer to go for binary because my simulations save data on a user request-basis, that is, the user decides what is saved. So, my simulations can go from saving almost nothing to saving a lot of data in certain cases. Plus, the reason why we allow the user to save many different variables is because we may want to look at the data in many different ways, such that which format the data should be saved (table or not, how many rows per time step?) may change from choice to choice. Saving the variables as separate strings in separate files allows the user to basically assemble back the saved data into tabular objects of the dimensions they want. The alternative would be to save tabular files, but then we would need to code options for various formats of tabular files depending on which variable(s) the user has decided to save and their resolution (e.g. does the variable have one value per time step, or one value per individual per time step?), and that would require a lot of (in my opinion, unnecessarily) complicated code. Also, if the user wants multiple tables out of the same simulation, that would probably mean a lot of redundant information (e.g. the time step column will likely be present in most of the saved tables). Saving each variable as a one-dimensional array of values allows us to combine the various outputs in whichever way we want after the simulation is done (of course, that requires providing functions to do that). But then, that means we are saving possibly a bunch of separate files from one simulation, and then, if since there may be a lot of writing going on, binary seems to make more sense (to me) than text. As a general rule of thumb, Hanno suggested to stick to binary once one gets the hang of it, but not to forget to communicate it properly to the audience.  

## 1. Use the memory

I save data by writing to file at every time step that I want to record, within on simulation. Writing to file is expensive, even if the file is in the local storage. There is also RAM available to the program, however, and so we may store the output data in memory before saving it to file. Of course, we cannot do this if the program saves a very large amount of data, just because we may not have the space in memory to store all that. That's an issue with my simulations: like I said above, my simulations are built to be flexible in which output they save, so the same program may save very little, or a lot of data, depending on the user's choice, making storing in memory a bit challenging in those latter cases. Fortunately, we can use a buffer to make sure that not too much data is stored in memory. For example, we could set the maximum buffer size to 2Gb, and when the amount of data in memory reaches that number, we flush the buffer by writing everything to a file. This way, we avoid writing too often to file, but we also make sure not to use too much memory. This should speed up my simulations a lot already.

## 2. Multiple simulations per process

Right now a simulation is a single process. However, a single process could be running multiple simulations, for example, by encapsulating the main simulation function in a for-loop, in which case the simulations would run sequentially, or by using thread pools, in which case the simulations could make use of multi-threading and run in parallel (as part of the same process, not multiple processes running in parallel). This is easy to implement if the simulation code is fully self-contained (e.g. no globals), which is my case, as I had to have very well-defined, modular parts of the program in order to run my tests. Plus, contrary to what I thought, it does not presuppose a particular way to explore parameter space. I thought that the structure of the for-loop, for example, would depend on whether we are exploring combinations of two parameters, or one single parameter along a transect of values, or several specific combinations out of all the possible ones. But actually, all a given simulation needs is some input parameter file, and as long as there are many parameter files, all we need to do is go through those in a single loop --- the code does not need to know how it is exploring parameter space.

Of course, using thread pools is more efficient than a for-loop, but it requires some knowledge of multi-threading. The big advantage is that it reduces the number of processes and the time spent on each process. In short, jobs are run in parallel, but within each job several simulations are also run in parallel. This may speed up by a lot the computations, but is quite a steep learning curve, according to Hanno. Running multiple simulations within a for-loop is no more efficient than running multiple simulations are separate processes. But see below.

## 3. Saving to a common file

Right now every simulation saves to its own files, and those are bundled together into a compressed archive that is then sent back to the shared storage. One suggestion was to have multiple simulations writing to the same file instead of multiple files, reducing the amount of files written to. Of course, there is still a lot of writing going on if we write at every time step, but see above about using memory before writing. If simulations run sequentially within the same process, it is easy to have them append their stuff to the expanding file (with or without buffer). But we need to make sure that we can tell the difference between the different simulations in the output. That is easy when saving, say, a table, where we can have columns with parameter values and simulation identifiers such as the seed. But when saving variables as separate binary files, it gets more complicated. Do we have multiple variable-files, each containing the saved value of that variable for multiple simulations? Then, we would need to make sure that we can differentiate the simulations upon reading. Which I guess is an option, but not one that would speed up things so much compared to compressing separate files once all the simulations are done running (especially if we use memory as buffer).

Alternatively, we could also have a multi-threaded process write the output of each of its simulations to a common file, but that gets very complicated to do (we do not want the threads to step on each other's toes). Not impossible, but would also not speed things up much compared to simply compressing the separate output once everything is done running. 

## Conclusion

Basically, writing less is the main goal post here --- easy to do and high reward. Multi-threading would be next in line, making optimal use of the parallel computing power that the cluster offers, but is quit the learning curve. Having multiple simulations writing to the same file is not much of an improvement over writing many files and compressing them.
