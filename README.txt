# SimpleProcess #

This small library makes it simple to use the `pcntl` extension;
Also, I strongly encourage you to take a look at the PThreads library (http://pthreads.org) which is more powerful than this tiny wrapper; I wrote it just because using pthreads was not an option for me.

# Basic usage #

Basic usage looks like this:

    declare(ticks=1); // This part is critical, be sure to include it
    $manager = new SimpleProcess\ProcessManager();
    $manager->fork(new SimpleProcess\Process(function() { sleep(5); }, "My super cool process"));
    
    do
    {
	foreach($manager->getChildren() as $process)
	{
	    $iid = $process->getInternalId();
	    if($process->isAlive())
	    {
		echo sprintf('Process %s is running', $iid);
	    } else if($process->isFinished()) {
		echo sprintf('Process %s is finished', $iid);
	    }
	    echo "\n";
	}
	sleep(1);
    } while($this->manager->countAliveChildren());
    
And that's it! Child processes will execute only the provided callable, so there is no need to worry about "If I am in the right process in this line?"; Parent process is executed normally after the `->fork()` was called; ProcessManager class takes care of reaping children processes, so you may focus on your application's logic instead of dark corners of pcntl_* functions.

# Communication between child processes and parent process #

ProcessManager may allocate some shared memory for each child process - then you may access it from the parent process:

    declare(ticks=1); // This part is critical, be sure to include it
    $manager = new SimpleProcess\ProcessManager();
    $manager->allocateSHMPerChildren(1000); // allocate 1000 bytes for each forked process
    for($i=0;$i<4;$i++)
    {
        $manager->fork(new SimpleProcess\Process(function(SimpleProcess\Process $currentProcess) {
            $currentProcess->getShmSegment()->save('status', 'Processing data...');
            sleep(5);
            $currentProcess->getShmSegment()->save('status', 'Connecting to the satellite...');
            sleep(5);
        }, $i));
    }
    $manager->cleanupOnShutdown(); // Register shutdown function that will release allocated shared memory;
                                   // It is important to call this after all fork() calls, as we don't want
                                   // to release it when child process exits

    do
    {
	foreach($manager->getChildren() as $process)
	{
	    $iid = $process->getInternalId();
	    if($process->isAlive())
	    {
		echo sprintf('Process %s is running with status "%s"', $iid, $process->getShmSegment()->fetch('status'));
	    } else if($process->isFinished()) {
		echo sprintf('Process %s finished execution', $iid);
	    }
	    echo "\n";
	}
	sleep(1);
    } while($this->manager->countAliveChildren());
    
# Other things #

This library comes also with the `Semaphore` class in case you'd need to use semaphores somewhere in your code; Use it like this:

    $s = Semaphore::create('critical_section');
    $s->acquire();
    $s->release();
    
    