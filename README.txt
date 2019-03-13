===========================================================================================

CREATED BY: 
	Vladimir Shargorodsky
	Dor Shtarker

===========================================================================================

INSTRUCTIONS:

	1. choose a unique bucket name and queue name and add them to the userinfo.txt file
	 located inside the LocalApplication folder.

	2. in your aws console, create a role with the premissions:
		a. EC2FullAccess
		b. S3FullAccess
		c. SQSFullAccess
		
	(if you run into problems, try to add AministratorAccess as well).
	copy the Instance Profile ARNs of the role, and add it to the userinfo.txt file.

	3. provide a key pair name and security group id to the userinfo.txt file as well.

	4. make sure your images url and userinfo files are located in the directory you will
	   run the jar from

	5. inside the LocalApplication folder, compile the LocalApplication using the command:
	   $ mvn package

	6. run using "java -jar target/LocalApplication-1.0.jar inputFile n" wherest inputFile
	   is your image urls file
	   and n is the number of images each worker should run ocr on.

	7. wait for completion.

===========================================================================================

HOW THE PROGRAM WORKS:

	The local application uploads the image url's file to the S3 bucket and bootstraps
	an isntance with the Manager jar, then sends a "new task" message to the SQS queue
	and waits for the "done task" message, once it recieves the message, the HTML file
	is downloaded from S3.

	The manager recieves the creates an Executor Service with a 10 threads thread pool,
	it then listens on the queue for "new task" messages, when one is recieved it creates
	a Runnable "TaskExecutor" to take care of the task.

	The TaskExecutor downloads the current image url's file from S3, and creates 2 unique
	SQS Queues, one for sending tasks, one for recieving answers.
	it then creates the currect ammount of Worker instances (up to 20, limited by aws),
	and waits for the tasks queue to empty out, it then recieves the answers from the answer
	queue and creats the HTML file, uploads it to S3 and sends a "done task" message,
	shuts down all the workers and deletes the queues.

	If the manager recieves a request while all it's threads are busy working, it will luanch
	a new manager isntance to offset more tasks. (altough its importent to note there is a
	limit on how many isntances we can luanch (20) without issuing a request to amazon and
	paying larger sums of money)

	The worker recieves messages from the tasks queue forever (until shut down), once it
	recieves one, it downloads the image file via a system call to run wget, it then runs
	Tesseract-ocr using a systam call as well
	its importent to note the ocr doesn't work well and most of the time the output is garbage.
	it then creates the answer message and sends it to the answers queue.

===========================================================================================

SECURITY:
	We did not send our credentials to any where, theres no need for it if you use roles.

===========================================================================================

SCALABILITY:
	As explained above, more and more Manager isntances are created to accomedate larger
	ammounts of clients.

===========================================================================================

PERSISTANCE:
	The workers never stop running, unless there is a internal problem with the AMI,
	any Exception thrown during runtime is cought and sent to an error queue, and then
	starts the loop from the start.
	the task message is only deleted if the task succedes, so tasks are never lost.
	by using SQSAsync we make sure no two workers are working on the same task.

===========================================================================================

THREADS:
	We use a thread pool inside the Manager to support multiple clients at once
	(currently using a t2.micro we use 10 threads, but we can increase that number using
	a stronger instance).

===========================================================================================

MORE THEN ONE CLIENT:
	We tested 4 clients and it worked

===========================================================================================

DID WE UNDERSTAND HOW THE SYSTEMS WORK?:
	Yes, as explained above.

===========================================================================================

TERMINATION:
	The Thread that runs the TaskExecutor task inside the Manager, Creates and deletes
	the instances and queues used by it during runtime.
	No element in the operation is responsible of terminating Manager instances, we must
	do it by hand, or write a dedicated application outside the scope of this assignment.

===========================================================================================

LIMITATION:
	we took this in mind. for instance we can only run 20 instances of the type t2.medium

===========================================================================================

WORKERS:
	all the workers keep looking for task to do inside the queues, and continue to do so
	unitl terminated by the manager.

===========================================================================================

MANAGER:
	the Manager only does what it's supposed to do, serve clients and wait for answers,
	then create and upload the HTML file.

===========================================================================================

DISTRIBUTED:
	The only waiting accures while the client (local application) awaits for the task to
	be done, and while the manager awaits for the workers to go over all the messages.

	besides that all the workers are independent of one another, and the ammount of clients
	connected simultaniously are limited only by amazon's restrictions.

===========================================================================================
FEAR AND LOATHING:
	Fear and Loating in Las Vegas is a movie about a country of bats, but we cant stop
	there.