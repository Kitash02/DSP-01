DSP course

Assignment 1


Instance type:
ami-054217b0faf130d36
Worker instance Type = T2.nano
Manager instance Type = T2_MEDIUM

Implementation:
First run the command: mvn clean package
Then usage:
>java -jar yourjar.jar inputFileName outputFileName n [terminate]

where:
        yourjar.jar: LocalApplication jar file full path (located in target folder).
        inputFileName: input file full path.
        outputFileName: output file name.
        n: is number of tasks per worker for the job.
        terminate: put 'terminate' string to terminate the program (optional).


Explanation:

LOCAL APP:

Local App starts a manager instance if one is not found.
Local App creates a unique bucket and uploads the input files to this bucket.
The Local App waiting for a response from the Manager indicating the process is done and then downloads the output files from s3.
If terminate flag in on, then the Local App sends a termination message to the Manager.
Finally, the Local App delete the unique SQS queue and the unique bucket.

MANAGER:

The manager's work is preformed using 2 main threads, with 2 different tasks that runs in parallel:
    1. apps listener
    2. workers listener

The first one listens on incoming messages from different local applications from APP_TO_MANAGER queue.
Once it receives a new message it:

send the massage to client thread.
It extracts the file location and downloads it.
split the message to small tasks and increment the number of tasks for this client massage.
Then send it to the workers throgh MANAGER_TO_WORKER queue.
If a terminate message is received, the terminateFlag will turn on in order to announce termination.

The second thread listens on the completed job from workers through WORKER_TO_MANAGER queue.
Once it receives a new message it:

submits the message as a task to a threadPool.
each thread in the threadPool sums up the jobs to match files and check whether the whole task is complete.
If the task is done, the thread is uploads the file to the match location, and sends to the local application a message indicating the process is done.



WORKER:

The workers waiting for a job from the manager through MANAGER_TO_WORKER queue.
Once a worker receives a message it extracts all the information.
Then, the worker goes through the task and preforms the tasks for each URL if it's possible.
Then he sends the parsed result to the manager through WORKER_TO_MANAGER queue.
Finally, remove the processed message from the MANAGER_TO_WORKER queue.



Runtime:
1. 1 app with terminate, 2500 files, n = 10 : 8 min
2. 1 app with terminate, 100 files, n = 10 : 5 min
3. 3 apps, 1 with terminate , 100 files for each(1557 KB), n = 10 : 5 min

Security:

We securely stored the credentials in a designated folder on our local computer, ensuring that they remain inaccessible to unauthorized parties.
By keeping the sensitive information offline, within our premises, we maintain strict control over who can access and retrieve these details.
Our system is designed to ensure that this critical data remains isolated and protected, as it is not included or accessible within the codebase.


Scalability:

We've designed the project for scalability, meaning it dynamically adjusts the number of worker computers based on workload.
We assume the users select a reasonable n according to workload, and the system opens an appropriate number of workers to handle tasks efficiently without overloading.
However, due to AWS limitations for students, we're constrained to running a maximum of 9 instances simultaneously (workers and manager instances).
While theoretically, multiple managers could handle a large client base, our project using a single manager according to the requirements.
To enhance manager efficiency within its limits, we employ a ThreadPool where each task runs concurrently in separate threads.
This strategy ensures flexibility without burdening the manager with intensive tasks.


Persistence:

If a worker node dies it's job will eventually timeout.The next time the manager polls the incoming task queue, it will re-evaluate the necessary-workers amount and create new worker instances.
When a termination message arrives in the incoming tasks queue, the manager updates it state. When all former tasks are done it terminates the workers, purges the queues and closes itself.
A maximum-workers amount was hard-coded to ensure the system doesn't exceed the maximum instances limit.
The manager doesn't do any of the parsing work. Each worker node's work is completely agnostic to the any other worker node's work.

When a worker node fails, messages are not deleted immediately.
Deletion occurs only after a worker completes its task.
We set a visibility timeout of 5n minutes to avoid interrupting ongoing work due to large inputs.
If a worker fails to complete within this time, the message becomes visible again for another worker to handle.


Threads:

As part of our scalability strategy, we integrated a custom ThreadPool-based implementation into the manager code.
This optimizes the manager's performance in handling message exchanges with a large number of users.
Threads can be problematic if they complicate the code unnecessarily.


Termination:
As soon as the manager receives a terminate message, he closes the APPTOMANAGER queue,
Then he waits for the two main threads (a thread listening to LOCALAPPS and a thread listening to WORKERS) to finish working.
Both threads are programmed to check if a TERMINATE message has been received, and if Yes, they first finish all their tasks and only then finish.
After the two threads are finished, the manager eliminates all the WORKERS, closes the two remaining queues (MANAGERTOWORKER and WORKERTOMANAGER)
and at the end of the process the manager turns itself off.