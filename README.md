# Паттерны параллельного программирования(concurrency) Go: Конвееры и отмена
Перевод https://blog.golang.org/pipelines

13 Mar 2014

Tags: concurrency, pipelines, cancellation

Sameer Ajmani

## Введение

Примитивы параллельного программирования(concurrency) Go упрощают создание потоковых конвейеров данных, которые эффективно используют I/O и несколько процессоров. В этой статье представлены примеры таких конвейеров, освещаются тонкости, возникающие при сбоях в работе, и внедряются методы для устранения сбоев.

## Что такое конвейер(pipeline)?

There's no formal definition of a pipeline in Go; it's just one of many kinds of
concurrent programs.  Informally, a pipeline is a series of _stages_ connected
by channels, where each stage is a group of goroutines running the same
function.  In each stage, the goroutines

В Go нет формального определения конвейера; это всего лишь один из многих видов параллельных программ. Неформально конвейер представляет собой серию `этапов`(*stages*), связанных каналами, где каждый этап представляет собой группу горутин, выполняющих ту же функцию. На каждом этапе горутины:

- получать значения от *upstream* через *inbound* каналы
- выполняют некоторые функции по этим данным, обычно производя новые значения
- отправляют значения *downstream* через *outbound* каналы

Каждый этап имеет любое количество входящих и исходящих каналов, за исключением первого и последнего этапов, которые имеют только исходящие или входящие каналы соответственно. Первый этап иногда называют *source*(*источник*) или *producer*; последний этап, *sink* или *consumer*(*потребитель*).

Мы начнем с простого примера конвейера, чтобы объяснить идеи и методы.
Позже мы представим более реалистичный пример.

## Возведение чисел в степень

Consider a pipeline with three stages.

The first stage, `gen`, is a function that converts a list of integers to a
channel that emits the integers in the list.  The `gen` function starts a
goroutine that sends the integers on the channel and closes the channel when all
the values have been sent:

```golang
func gen(nums ...int) <-chan int {
	out := make(chan int)
	go func() {
		for _, n := range nums {
			out <- n
		}
		close(out)
	}()
	return out
}
```
[`Смотри исходный код`](https://github.com/Konstantin8105/Go-pipelines/blob/master/pipelines/square.go)

The second stage, `sq`, receives integers from a channel and returns a
channel that emits the square of each received integer.  After the
inbound channel is closed and this stage has sent all the values
downstream, it closes the outbound channel:

```golang
func sq(in <-chan int) <-chan int {
	out := make(chan int)
	go func() {
		for n := range in {
			out <- n * n
		}
		close(out)
	}()
	return out
}
```
[`Смотри исходный код`](https://github.com/Konstantin8105/Go-pipelines/blob/master/pipelines/square.go)

The `main` function sets up the pipeline and runs the final stage: it receives
values from the second stage and prints each one, until the channel is closed:

```golang
func main() {
	// Set up the pipeline.
	c := gen(2, 3)
	out := sq(c)

	// Consume the output.
	fmt.Println(<-out) // 4
	fmt.Println(<-out) // 9
}
```
[`Смотри исходный код`](https://github.com/Konstantin8105/Go-pipelines/blob/master/pipelines/square.go)

Since `sq` has the same type for its inbound and outbound channels, we
can compose it any number of times.  We can also rewrite `main` as a
range loop, like the other stages:

```golang
func main() {
	// Set up the pipeline and consume the output.
	for n := range sq(sq(gen(2, 3))) {
		fmt.Println(n) // 16 then 81
	}
}
```
[`Смотри исходный код`](https://github.com/Konstantin8105/Go-pipelines/blob/master/pipelines/square2.go)

## Fan-out, fan-in

Multiple functions can read from the same channel until that channel is closed;
this is called _fan-out_. This provides a way to distribute work amongst a group
of workers to parallelize CPU use and I/O.

A function can read from multiple inputs and proceed until all are closed by
multiplexing the input channels onto a single channel that's closed when all the
inputs are closed.  This is called _fan-in_.

We can change our pipeline to run two instances of `sq`, each reading from the
same input channel.  We introduce a new function, _merge_, to fan in the
results:

```golang
func main() {
	in := gen(2, 3)

	// Distribute the sq work across two goroutines that both read from in.
	c1 := sq(in)
	c2 := sq(in)

	// Consume the merged output from c1 and c2.
	for n := range merge(c1, c2) {
		fmt.Println(n) // 4 then 9, or 9 then 4
	}
}
```
[`Смотри исходный код`](https://github.com/Konstantin8105/Go-pipelines/blob/master/pipelines/sqfan.go)

The `merge` function converts a list of channels to a single channel by starting
a goroutine for each inbound channel that copies the values to the sole outbound
channel.  Once all the `output` goroutines have been started, `merge` starts one
more goroutine to close the outbound channel after all sends on that channel are
done.

Sends on a closed channel panic, so it's important to ensure all sends
are done before calling close.  The
[`sync.WaitGroup`](http://golang.org/pkg/sync/#WaitGroup) type
provides a simple way to arrange this synchronization:

```golang
// merge receives values from each input channel and sends them on the returned
// channel.  merge closes the returned channel after all the input values have
// been sent.
func merge(cs ...<-chan int) <-chan int {
	var wg sync.WaitGroup // HL
	out := make(chan int)

	// Start an output goroutine for each input channel in cs.  output
	// copies values from c to out until c is closed, then calls wg.Done.
	output := func(c <-chan int) {
		for n := range c {
			out <- n
		}
		wg.Done() // HL
	}
	wg.Add(len(cs)) // HL
	for _, c := range cs {
		go output(c)
	}

	// Start a goroutine to close out once all the output goroutines are
	// done.  This must start after the wg.Add call.
	go func() {
		wg.Wait() // HL
		close(out)
	}()
	return out
}
```
[`Смотри исходный код`](https://github.com/Konstantin8105/Go-pipelines/blob/master/pipelines/sqfan.go)

## Stopping short

There is a pattern to our pipeline functions:

- stages close their outbound channels when all the send operations are done.
- stages keep receiving values from inbound channels until those channels are closed.

This pattern allows each receiving stage to be written as a `range` loop and
ensures that all goroutines exit once all values have been successfully sent
downstream.

But in real pipelines, stages don't always receive all the inbound
values.  Sometimes this is by design: the receiver may only need a
subset of values to make progress.  More often, a stage exits early
because an inbound value represents an error in an earlier stage. In
either case the receiver should not have to wait for the remaining
values to arrive, and we want earlier stages to stop producing values
that later stages don't need.

In our example pipeline, if a stage fails to consume all the inbound values, the
goroutines attempting to send those values will block indefinitely:

.code pipelines/sqleak.go /first value/,/^}/

This is a resource leak: goroutines consume memory and runtime resources, and
heap references in goroutine stacks keep data from being garbage collected.
Goroutines are not garbage collected; they must exit on their own.

We need to arrange for the upstream stages of our pipeline to exit even when the
downstream stages fail to receive all the inbound values.  One way to do this is
to change the outbound channels to have a buffer.  A buffer can hold a fixed
number of values; send operations complete immediately if there's room in the
buffer:
```
        c := make(chan int, 2) // buffer size 2
        c <- 1  // succeeds immediately
        c <- 2  // succeeds immediately
        c <- 3  // blocks until another goroutine does <-c and receives 1
```
When the number of values to be sent is known at channel creation time, a buffer
can simplify the code.  For example, we can rewrite `gen` to copy the list of
integers into a buffered channel and avoid creating a new goroutine:

.code pipelines/sqbuffer.go /func gen/,/^}/

Returning to the blocked goroutines in our pipeline, we might consider adding a
buffer to the outbound channel returned by `merge`:

.code pipelines/sqbuffer.go /func merge/,/unchanged/

While this fixes the blocked goroutine in this program, this is bad code.  The
choice of buffer size of 1 here depends on knowing the number of values `merge`
will receive and the number of values downstream stages will consume.  This is
fragile: if we pass an additional value to `gen`, or if the downstream stage
reads any fewer values, we will again have blocked goroutines.

Instead, we need to provide a way for downstream stages to indicate to the
senders that they will stop accepting input.

## Explicit cancellation

When `main` decides to exit without receiving all the values from
`out`, it must tell the goroutines in the upstream stages to abandon
the values they're trying it send.  It does so by sending values on a
channel called `done`.  It sends two values since there are
potentially two blocked senders:

```golang
func main() {
	in := gen(2, 3)

	// Distribute the sq work across two goroutines that both read from in.
	c1 := sq(in)
	c2 := sq(in)

	// Consume the first value from output.
	done := make(chan struct{}, 2) // HL
	out := merge(done, c1, c2)
	fmt.Println(<-out) // 4 or 9

	// Tell the remaining senders we're leaving.
	done <- struct{}{} // HL
	done <- struct{}{} // HL
}
```
[`Смотри исходный код`](https://github.com/Konstantin8105/Go-pipelines/blob/master/pipelines/sqdone1.go)

The sending goroutines replace their send operation with a `select` statement
that proceeds either when the send on `out` happens or when they receive a value
from `done`.  The value type of `done` is the empty struct because the value
doesn't matter: it is the receive event that indicates the send on `out` should
be abandoned.  The `output` goroutines continue looping on their inbound
channel, `c`, so the upstream stages are not blocked. (We'll discuss in a moment
how to allow this loop to return early.)

```golang
func merge(done <-chan struct{}, cs ...<-chan int) <-chan int {
	var wg sync.WaitGroup
	out := make(chan int)

	// Start an output goroutine for each input channel in cs.  output
	// copies values from c to out until c is closed or it receives a value
	// from done, then output calls wg.Done.
	output := func(c <-chan int) {
		for n := range c {
			select {
			case out <- n:
			case <-done: // HL
			}
		}
		wg.Done()
	}
	// ... the rest is unchanged ...

	wg.Add(len(cs))
	for _, c := range cs {
		go output(c)
	}

	// Start a goroutine to close out once all the output goroutines are
	// done.  This must start after the wg.Add call.
	go func() {
		wg.Wait()
		close(out)
	}()
	return out
}
```
[`Смотри исходный код`](https://github.com/Konstantin8105/Go-pipelines/blob/master/pipelines/sqdone1.go)

This approach has a problem: _each_ downstream receiver needs to know the number
of potentially blocked upstream senders and arrange to signal those senders on
early return.  Keeping track of these counts is tedious and error-prone.

We need a way to tell an unknown and unbounded number of goroutines to
stop sending their values downstream.  In Go, we can do this by
closing a channel, because
[a receive operation on a closed channel can always proceed immediately, yielding the element type's zero value.](http://golang.org/ref/spec#Receive_operator)

This means that `main` can unblock all the senders simply by closing
the `done` channel.  This close is effectively a broadcast signal to
the senders.  We extend _each_ of our pipeline functions to accept
`done` as a parameter and arrange for the close to happen via a
`defer` statement, so that all return paths from `main` will signal
the pipeline stages to exit.

```golang
func main() {
	// Set up a done channel that's shared by the whole pipeline,
	// and close that channel when this pipeline exits, as a signal
	// for all the goroutines we started to exit.
	done := make(chan struct{}) // HL
	defer close(done)           // HL

	in := gen(done, 2, 3)

	// Distribute the sq work across two goroutines that both read from in.
	c1 := sq(done, in)
	c2 := sq(done, in)

	// Consume the first value from output.
	out := merge(done, c1, c2)
	fmt.Println(<-out) // 4 or 9

	// done will be closed by the deferred call. // HL
}
```
[`Смотри исходный код`](https://github.com/Konstantin8105/Go-pipelines/blob/master/pipelines/sqdone3.go)

Each of our pipeline stages is now free to return as soon as `done` is closed.
The `output` routine in `merge` can return without draining its inbound channel,
since it knows the upstream sender, `sq`, will stop attempting to send when
`done` is closed.  `output` ensures `wg.Done` is called on all return paths via
a `defer` statement:

```golang
func merge(done <-chan struct{}, cs ...<-chan int) <-chan int {
	var wg sync.WaitGroup
	out := make(chan int)

	// Start an output goroutine for each input channel in cs.  output
	// copies values from c to out until c or done is closed, then calls
	// wg.Done.
	output := func(c <-chan int) {
		defer wg.Done() // HL
		for n := range c {
			select {
			case out <- n:
			case <-done:
				return // HL
			}
		}
	}
	// ... the rest is unchanged ...

	wg.Add(len(cs))
	for _, c := range cs {
		go output(c)
	}

	// Start a goroutine to close out once all the output goroutines are
	// done.  This must start after the wg.Add call.
	go func() {
		wg.Wait()
		close(out)
	}()
	return out
}
```
[`Смотри исходный код`](https://github.com/Konstantin8105/Go-pipelines/blob/master/pipelines/sqdone3.go)

Similarly, `sq` can return as soon as `done` is closed.  `sq` ensures its `out`
channel is closed on all return paths via a `defer` statement:

```golang
func sq(done <-chan struct{}, in <-chan int) <-chan int {
	out := make(chan int)
	go func() {
		defer close(out) // HL
		for n := range in {
			select {
			case out <- n * n:
			case <-done:
				return // HL
			}
		}
	}()
	return out
}
```
[`Смотри исходный код`](https://github.com/Konstantin8105/Go-pipelines/blob/master/pipelines/sqdone3.go)

Here are the guidelines for pipeline construction:

- stages close their outbound channels when all the send operations are done.
- stages keep receiving values from inbound channels until those channels are closed or the senders are unblocked.

Pipelines unblock senders either by ensuring there's enough buffer for all the
values that are sent or by explicitly signalling senders when the receiver may
abandon the channel.

## Digesting a tree

Let's consider a more realistic pipeline.

MD5 is a message-digest algorithm that's useful as a file checksum.  The command
line utility `md5sum` prints digest values for a list of files.
```
	% md5sum *.go
	d47c2bbc28298ca9befdfbc5d3aa4e65  bounded.go
	ee869afd31f83cbb2d10ee81b2b831dc  parallel.go
	b88175e65fdcbc01ac08aaf1fd9b5e96  serial.go
```
Our example program is like `md5sum` but instead takes a single directory as an
argument and prints the digest values for each regular file under that
directory, sorted by path name.
```
	% go run serial.go .
	d47c2bbc28298ca9befdfbc5d3aa4e65  bounded.go
	ee869afd31f83cbb2d10ee81b2b831dc  parallel.go
	b88175e65fdcbc01ac08aaf1fd9b5e96  serial.go
```
The main function of our program invokes a helper function `MD5All`, which
returns a map from path name to digest value, then sorts and prints the results:

```golang
func main() {
	// Calculate the MD5 sum of all files under the specified directory,
	// then print the results sorted by path name.
	m, err := MD5All(os.Args[1]) // HL
	if err != nil {
		fmt.Println(err)
		return
	}
	var paths []string
	for path := range m {
		paths = append(paths, path)
	}
	sort.Strings(paths) // HL
	for _, path := range paths {
		fmt.Printf("%x  %s\n", m[path], path)
	}
}
```
[`Смотри исходный код`](https://github.com/Konstantin8105/Go-pipelines/blob/master/pipelines/serial.go)

The `MD5All` function is the focus of our discussion.  In
[[pipelines/serial.go][serial.go]], the implementation uses no concurrency and
simply reads and sums each file as it walks the tree.

```golang
func MD5All(root string) (map[string][md5.Size]byte, error) {
	m := make(map[string][md5.Size]byte)
	err := filepath.Walk(root, func(path string, info os.FileInfo, err error) error { // HL
		if err != nil {
			return err
		}
		if !info.Mode().IsRegular() {
			return nil
		}
		data, err := ioutil.ReadFile(path) // HL
		if err != nil {
			return err
		}
		m[path] = md5.Sum(data) // HL
		return nil
	})
	if err != nil {
		return nil, err
	}
	return m, nil
}
```
[`Смотри исходный код`](https://github.com/Konstantin8105/Go-pipelines/blob/master/pipelines/serial.go)

## Parallel digestion

In [[pipelines/parallel.go][parallel.go]], we split `MD5All` into a two-stage
pipeline.  The first stage, `sumFiles`, walks the tree, digests each file in
a new goroutine, and sends the results on a channel with value type `result`:

```golang
// A result is the product of reading and summing a file using MD5.
type result struct {
	path string
	sum  [md5.Size]byte
	err  error
}
```
[`Смотри исходный код`](https://github.com/Konstantin8105/Go-pipelines/blob/master/pipelines/parallel.go)

`sumFiles` returns two channels: one for the `results` and another for the error
returned by `filepath.Walk`.  The walk function starts a new goroutine to
process each regular file, then checks `done`.  If `done` is closed, the walk
stops immediately:

```golang
func sumFiles(done <-chan struct{}, root string) (<-chan result, <-chan error) {
	// For each regular file, start a goroutine that sums the file and sends
	// the result on c.  Send the result of the walk on errc.
	c := make(chan result)
	errc := make(chan error, 1)
	go func() { // HL
		var wg sync.WaitGroup
		err := filepath.Walk(root, func(path string, info os.FileInfo, err error) error {
			if err != nil {
				return err
			}
			if !info.Mode().IsRegular() {
				return nil
			}
			wg.Add(1)
			go func() { // HL
				data, err := ioutil.ReadFile(path)
				select {
				case c <- result{path, md5.Sum(data), err}: // HL
				case <-done: // HL
				}
				wg.Done()
			}()
			// Abort the walk if done is closed.
			select {
			case <-done: // HL
				return errors.New("walk canceled")
			default:
				return nil
			}
		})
		// Walk has returned, so all calls to wg.Add are done.  Start a
		// goroutine to close c once all the sends are done.
		go func() { // HL
			wg.Wait()
			close(c) // HL
		}()
		// No select needed here, since errc is buffered.
		errc <- err // HL
	}()
	return c, errc
}
```
[`Смотри исходный код`](https://github.com/Konstantin8105/Go-pipelines/blob/master/pipelines/parallel.go)

`MD5All` receives the digest values from `c`.  `MD5All` returns early on error,
closing `done` via a `defer`:

```golang
func MD5All(root string) (map[string][md5.Size]byte, error) {
	// MD5All closes the done channel when it returns; it may do so before
	// receiving all the values from c and errc.
	done := make(chan struct{}) // HLdone
	defer close(done)           // HLdone

	c, errc := sumFiles(done, root) // HLdone

	m := make(map[string][md5.Size]byte)
	for r := range c { // HLrange
		if r.err != nil {
			return nil, r.err
		}
		m[r.path] = r.sum
	}
	if err := <-errc; err != nil {
		return nil, err
	}
	return m, nil
}
```
[`Смотри исходный код`](https://github.com/Konstantin8105/Go-pipelines/blob/master/pipelines/parallel.go)

## Bounded parallelism

The `MD5All` implementation in [[pipelines/parallel.go][parallel.go]]
starts a new goroutine for each file. In a directory with many large
files, this may allocate more memory than is available on the machine.

We can limit these allocations by bounding the number of files read in
parallel.  In [[pipelines/bounded.go][bounded.go]], we do this by
creating a fixed number of goroutines for reading files.  Our pipeline
now has three stages: walk the tree, read and digest the files, and
collect the digests.

The first stage, `walkFiles`, emits the paths of regular files in the tree:

```golang
func walkFiles(done <-chan struct{}, root string) (<-chan string, <-chan error) {
	paths := make(chan string)
	errc := make(chan error, 1)
	go func() { // HL
		// Close the paths channel after Walk returns.
		defer close(paths) // HL
		// No select needed for this send, since errc is buffered.
		errc <- filepath.Walk(root, func(path string, info os.FileInfo, err error) error { // HL
			if err != nil {
				return err
			}
			if !info.Mode().IsRegular() {
				return nil
			}
			select {
			case paths <- path: // HL
			case <-done: // HL
				return errors.New("walk canceled")
			}
			return nil
		})
	}()
	return paths, errc
}
```
[`Смотри исходный код`](https://github.com/Konstantin8105/Go-pipelines/blob/master/pipelines/bounded.go)

The middle stage starts a fixed number of `digester` goroutines that receive
file names from `paths` and send `results` on channel `c`:

```golang
func digester(done <-chan struct{}, paths <-chan string, c chan<- result) {
	for path := range paths { // HLpaths
		data, err := ioutil.ReadFile(path)
		select {
		case c <- result{path, md5.Sum(data), err}:
		case <-done:
			return
		}
	}
}
```
[`Смотри исходный код`](https://github.com/Konstantin8105/Go-pipelines/blob/master/pipelines/bounded.go)

Unlike our previous examples, `digester` does not close its output channel, as
multiple goroutines are sending on a shared channel.  Instead, code in `MD5All`
arranges for the channel to be closed when all the `digesters` are done:

```golang
c := make(chan result) // HLc
var wg sync.WaitGroup
const numDigesters = 20
wg.Add(numDigesters)
for i := 0; i < numDigesters; i++ {
    go func() {
        digester(done, paths, c) // HLc
        wg.Done()
    }()
}
go func() {
    wg.Wait()
    close(c) // HLc
}()
```
[`Смотри исходный код`](https://github.com/Konstantin8105/Go-pipelines/blob/master/pipelines/bounded.go)

We could instead have each digester create and return its own output
channel, but then we would need additional goroutines to fan-in the
results.

The final stage receives all the `results` from `c` then checks the
error from `errc`.  This check cannot happen any earlier, since before
this point, `walkFiles` may block sending values downstream:

```golang
m := make(map[string][md5.Size]byte)
for r := range c {
    if r.err != nil {
        return nil, r.err
    }
    m[r.path] = r.sum
}
// Check whether the Walk failed.
if err := <-errc; err != nil { // HLerrc
    return nil, err
}
return m, nil
```
[`Смотри исходный код`](https://github.com/Konstantin8105/Go-pipelines/blob/master/pipelines/bounded.go)

## Conclusion

This article has presented techniques for constructing streaming data pipelines
in Go.  Dealing with failures in such pipelines is tricky, since each stage in
the pipeline may block attempting to send values downstream, and the downstream
stages may no longer care about the incoming data.  We showed how closing a
channel can broadcast a "done" signal to all the goroutines started by a
pipeline and defined guidelines for constructing pipelines correctly.

Further reading:

- [Go Concurrency Patterns](http://talks.golang.org/2012/concurrency.slide#1) ([video](https://www.youtube.com/watch?v=f6kdp27TYZs)) presents the basics of Go's concurrency primitives and several ways to apply them.
- [Advanced Go Concurrency Patterns](http://blog.golang.org/advanced-go-concurrency-patterns) ([video](http://www.youtube.com/watch?v=QDDwwePbDtw)) covers more complex uses of Go's primitives, especially `select`.
- Douglas McIlroy's paper [Squinting at Power Series](http://swtch.com/~rsc/thread/squint.pdf) shows how Go-like concurrency provides elegant support for complex calculations.
