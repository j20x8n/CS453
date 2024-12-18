java cProject description (CS-453)
September 24, 2024
Contents
1 Software transactional memory 2
1.1 Gentle introduction . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . 2
1.2 Speciffcation . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . 4
1.3 Possible implementations (in English) . . . . . . . . . . . . . . . . . . . . . . . . . . . . . 5
1.3.1 Using a coarse-lock (i.e. the “reference” implementation) . . . . . . . . . . . . . . . 5
1.3.2 Dual-versioned transactional memory . . . . . . . . . . . . . . . . . . . . . . . . . 5
2 The project 9
2.1 Practical information . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . 9
2.1.1 Testing and submission . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . 10
2.1.2 Grading . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . 10
2.1.3 Other rules . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . 11
2.2 Interface of the software transactional memory . . . . . . . . . . . . . . . . . . . . . . . . 11
2.2.1 Data structures . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . 12
2.2.2 Functions to implement . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . 12
3 Concurrent programming with shared memory 17
3.1 Atomic operations since C11/C++11 . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . 17
3.2 Memory ordering in C11/C++11: a quick overview . . . . . . . . . . . . . . . . . . . . . . 17
3.3 From C11/C++11 to the model used in the course . . . . . . . . . . . . . . . . . . . . . . 19
11 Software transactional memory
Your goal is to implement a software transactional memory library.
The project repository (see Section 2.1) contains a very simple reference implementation of such a
library, along with the skeleton for another implementation: the one you’ll have to (design and) code.
The project repository includes the source code for, and a Makeffle to run the program that will evaluate
 your implementation on the evaluation server (provided by the Distributed Computing Laboratory).
But ffrst, let’s informally describe what a (software) transactional memory is, and why it can be useful.
1.1 Gentle introduction
Let me introduce you to Alice. Alice runs a bank. As for any bank today, her system must be able to
process electronic orders. And it should be fast (Alice’s bank is very successful and has many customers).
At least two decades ago, to increase the system throughput, Alice could have only needed to buy
newer hardware, and her program would run faster. But 15–20 years ago, this free lunch ended [1]: Alice
now has to use the computational power of several hardware threads to get the most of modern processors.
This is where a transactional memory can come in handy, and where you step in to help Alice.
Alice’s program is currently written to run on a single thread. When Alice runs her code on her
single-core processor (no concurrency), it runs as expected.
For instance, let’s say that two customers, Bob and Charlie, want to transfer money: both Bob and
Charlie wants to wire each other (and they did not agree beforehand on “who owes who” to make only
one transfer. . .). The pseudo-code for the “money transfer” function could be the following:
shared accounts as integer[];
fn transfer(src, dst, amount) {
if (accounts[src]  no transfer
accounts[dst] = accounts[dst] + amount;
accounts[src] = accounts[src] - amount;
}
Bob wants to send 10 CHF to Charlie, and Charlie 15 CHF to Bob. Since there is only one hardware
thread, each transfer will naturally be processed one after the other. That is, if Bob’s request was
received just before Charlie’s request:
Bob 12 CHF
Charlie 10 CHF
−−−−−−−−→
Bob’s request
Bob 2 CHF
Charlie 20 CHF
−−−−−−−−−−→
Charlie’s request
Bob 17 CHF
Charlie 5 CHF
Another possible execution, if instead Charlie’s request was processed ffrst by Alice’s banking system:
Bob 12 CHF
Charlie 10 CHF
−−−−−−−−−−→
Charlie’s request
Bob 12 CHF
Charlie 10 CHF
−−−−−−−−→
Bob’s request
Bob 2 CHF
Charlie 20 CHF
Even if Bob may not be very happy with this second possible outcome, it is still a consistent execution
as far as the speciffcation of the bank is concerned: when the funds are insufffcient, no transfer happens.
Now, Alice wants to try her program (by assumption correct on a single-core processor) on brandnew,
 multi-core hardware. This way her system will be able to process transfers concurrently, (hopefully)
answering her customers faster. So Alice, for testing purposes, runs the same program on multiple
threads, each taking customer requests as they come. To analyze what outcomes are possible when two
threads process the two respective transfers from Bob and Charlie in concurrence, we will rewrite the
pseudo code to highlight each shared memory access, and in which order they are written in the program:
fn transfer(src, dst, amount) {
let v0 = accounts[src]; // 0th read
if (v0  no transfer
let v1 = accounts[dst]; // 1st read
accounts[dst] = v1 + amount; // 1st write
let v2 = accounts[src]; // 2nd read
accounts[src] = v2 - amount; // 2nd write
}
2We assume that this pseudo-code runs on a pseudo-machine with atomic register
1
, and that this
pseudo-machine does not reorder memory accesses
2
: memory is always accessed in the same order as
written in the program, and there exist a total order following which each atomic memory access happens.
Supposing the account of Bob is at index 0 in accounts, and the account of Charlie at index 1, a possible,
concurrent ex代 写CS-453、C++
代做程序编程语言ecution is then (for clarity, v0, v1, v2 have been replaced by accounts[. . . ]):
Bob request’s thread Charlie request’s thread
0th read 12 == accounts[0]
1st read 10 == accounts[1]
1st write accounts[1] = 20
0th read 20 == accounts[1]
1st read 12 == accounts[0]
2nd read 12 == accounts[0]
2nd write accounts[0] = 2
1st write accounts[0] = 27
2nd read 20 == accounts[1]
2nd write accounts[1] = 5
Figure 1: A concurrent execution that leads to an inconsistent state.
Under this concurrent execution, starting from Bob and Charlie having respectively 12 + 10 = 22
CHF in total on their bank accounts, they end up with a total of 27 + 5 = 32 CHF. Oups! Alice’s system
involutarily “offered” 10 CHF to Bob, by letting the thread from Charlie’s request overwrites (in this
speciffc execution) the deduction of 10 CHF made by the thread concurrently handling Bob’s request.
From the standpoint of the speciffcation of the bank, this is an inconsistency.
A possible immediate solution could be to use a mutual exclusion mechanism, i.e. a lock:
fn transfer(src, dst, amount) {
lock accounts { /* Only one thread at a time can enter this block:
...concurrent threads wait until no thread is in this block */
let v0 = accounts[src]; // 0th read
if (v0  no transfer
let v1 = accounts[dst]; // 1st read
accounts[dst] = v1 + amount; // 1st write
let v2 = accounts[src]; // 2nd read
accounts[src] = v2 - amount; // 2nd write
}
}
Such a mechanism would prevent the execution from Figure 1, as only one thread accessing the shared
variable accounts can run at any given time. But this mechanism also prevents multiple threads from
processing transfers concurrently. Alice wanted to use multiple cores speciffcally to process multiple
requests at the same time, answering faster her (many) customers. Always taking a single, coarse lock
when accessing the shared memory is thus not a satisfying solution.
Actually, Alice noticed that some transfers can always run concurrently, without the need for any
synchronization. For instance, if Bob wires Charlie and Dan wires Erin at the same time, no ordering
 of memory accesses can lead to an inconsistency. This is simply because the two sets of shared
memory segments respectively accessed by these two transfers, i.e. {accounts[0], accounts[1]} and
{accounts[2], accounts[3]}, do not overlap
3
: they do not conffict.
This is where the transactional memory is useful: it runs in parallel transactions that do not conffict,
 and synchronizes these which do conffict, (hopefully) allowing to make the most out of multi-core
processors with minimal coding efforts. To get a glimpse of the general ideas behind the interface of the
software transactional memory proposed in this project
4
, let’s rewrite (again) Alice’s pseudo-code:
1The notion of atomic register is taught at the beginning of the course.
2The memory consistency models of modern hardware is outside the scope of this course.
3Also, memory segments that are only read by each thread can overlap without synchronization.
4Please refer to Section 2.2 for a precise deffnition of the interface of the library you have to implement.
3// Initialization of the transactional memory, which holds the accounts
shared tm = new TransactionalMemory(/* ... */);
/* ...Initialize the array of accounts,
at the "start address" of the shared memory... */
fn transfer(src, dst, amount) {
let accounts = tm.get start address() as integer[];
while (true) {
try {
let tx = tm.begin();
if (tx.read(accounts[src])  123456.zip
The UUID (Unique User IDentifier) to use is the one you should have received by mail.
Your request will be queued on the evaluation server, and you will receive the results of both the compilation
and the execution of your implementation (including its speedup vs the reference one) as soon as
it finishes running. You can submit code as often as you want until the deadline.
Only the (correct) submission with the highest speedup will be stored persistently and considered for
grading, so no need to resubmit your best implementation just before the deadline. The TAs may still
ask you to provide a copy of your best implementation though (possibly after the deadline), so do not
delete anything before the end of the semester (2025-01-30).
As an option from the submit.py script, you can force-replace your submission with another one. In
this case no compilation nor performance check is performed. The submission script also allows you to
download the submission considered for your grade, so you can make sure the system has the version you
want to be considered for grading. You can display the command-line help for this tool with:
you@your-pc:/path/to/repository$ python3 submit.py -h
Also note that the automated submission system has no false negative, i.e. a correct implementation
will always be deemed correct; unless it is more than 16 times slower than the reference (see Section
2.1.2).
On the other hand, the system has false positive (i.e. incorrect implementations deemed correct).
Unless your submission includes portions of code that appear to be specifically designed to trick the
submission system into considering an incorrect implementation correct, if the submission system qualifies
your implementation as correct, it will be graded as a correct one.
2.1.2 Grading
Correctness is a must
The executed transactions must satisfy the specification (sections 1.2 and 2.2).
No correctness ⇒ no “passing” grade for the project (i.e. grade  2.5 g = 6
For reference, the maximum speedup obtained on this project is ×15.04 with an optimized (algorithmwise)
version of TL2. The assoc         
加QQ：99515681  WX：codinghelp  Email: 99515681@qq.com
