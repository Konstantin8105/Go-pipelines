# Паттерны параллельного программирования(concurrency) Go: Конвееры и отмена
Перевод https://blog.golang.org/pipelines

13 Mar 2014

Tags: concurrency, pipelines, cancellation

Sameer Ajmani

## Введение

Примитивы параллельного программирования(concurrency) Go упрощают создание потоковых конвейеров данных, которые эффективно используют I/O и несколько процессоров. В этой статье представлены примеры таких конвейеров, освещаются тонкости, возникающие при сбоях в работе, и внедряются методы для устранения сбоев.

## Что такое конвейер(pipeline)?

В Go нет формального определения конвейера; это всего лишь один из многих видов параллельных программ. Неформально конвейер представляет собой серию `этапов`(*stages*), связанных каналами, где каждый этап представляет собой группу горутин, выполняющих ту же функцию. На каждом этапе горутины:

- получать значения от *upstream* через *inbound* каналы
- выполняют некоторые функции по этим данным, обычно производя новые значения
- отправляют значения *downstream* через *outbound* каналы

Каждый этап имеет любое количество входящих и исходящих каналов, за исключением первого и последнего этапов, которые имеют только исходящие или входящие каналы соответственно. Первый этап иногда называют *source*(*источник*) или *producer*; последний этап, *sink* или *consumer*(*потребитель*).

Мы начнем с простого примера конвейера, чтобы объяснить идеи и методы.
Позже мы представим более реалистичный пример.

## Возведение чисел в степень

Рассмотрим конвеер с тремя этапами.

Первый этап, `gen`, является функцией, которая преобразует список целых чисел в канал. Функция `gen` запускает горутину, которая отправляет целые числа в канал и закрывает канал, когда все значения были отправлены:

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

Второй этап, `sq`, получает целые числа из канала и возвращает канал, который испускает квадрат каждого принятого целого. После того, как входящий канал закрыт, и этот этап отправил все значения ниже по течению, он закрывает исходящий канал:

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

Функция `main` создает конвейер и запускает заключительный этап: он получает значения со второго этапа и печатает их, пока канал не будет закрыт:

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

Поскольку `sq` имеет один и тот же тип для входящих и исходящих каналов, мы можем создавать его сколько угодно раз. Мы также можем переписать `main` как цикл диапазона, как и другие этапы:

```golang
func main() {
	// Set up the pipeline and consume the output.
	for n := range sq(sq(gen(2, 3))) {
		fmt.Println(n) // 16 then 81
	}
}
```
[`Смотри исходный код`](https://github.com/Konstantin8105/Go-pipelines/blob/master/pipelines/square2.go)

## Расщепление (fan-out), слияние (fan-in) каналов

Несколько функций могут считывать значения с одного канала до тех пор, пока этот канал не будет закрыт; это называется *расщепление*(*fan-out*) канала. Это дает возможность распределить работу среди группы горутин для параллелизации использования ЦП и операций ввода-вывода.

Функция может считываться с нескольких входов и действовать до тех пор, пока все не будут закрыты путем мультиплексирования входных каналов на один канал, который закрыт, когда все входы будут закрыты. Это называется *слияние*(*fan-in*) каналов.

Мы можем изменить наш конвейер для запуска двух экземпляров `sq`, каждый из которых считывается из одного входного канала. Введем новую функцию, `merge`, чтобы объеденить результаты:

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

Функция `merge` преобразует список каналов в один канал, запустив горутину для каждого входящего канала, который копирует значения в единственный исходящий канал. Как только все «выходные» горутины были запущены, `merge` запускает еще одну горутину, чтобы закрыть исходящий канал после того, как все отправленные на этом канале будут выполнены.

Отправка в закрытый канал приводит к панике, поэтому важно обеспечить, чтобы все посылки были выполнены до вызова. Тип [`sync.WaitGroup`](http://golang.org/pkg/sync/#WaitGroup) обеспечивает простой способ организовать эту синхронизацию:

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

## Внезапная остановка

Для наших функций конвеера существует следующий паттерн:

- этапы закрытия исходящих каналов, когда выполняются все операции отправки.
- этапы продолжают получать значения от входящих каналов до тех пор, пока эти каналы не будут закрыты.

Этот шаблон позволяет каждому участку приема записываться как цикл «range» и гарантирует, что все горутины будут завершены после того, как все значения будут успешно отправлены вниз по течению.

Но в реальных конвеерах этапы не всегда получают все входящие значения. Иногда получателю может понадобиться только подмножество значений для достижения прогресса. Чаще всего этап выходит раньше, потому что входящее значение представляет ошибку на ранней стадии. В любом случае получателю не нужно ждать появления остальных значений, и мы хотим, чтобы более ранние этапы перестали выдавать значения, которые более поздние этапы не нужны.

В нашем примере конвейера, если на этапе не хватает входящих значений, горутины, пытающиеся отправить эти значения, будут бесконечно блокироваться:

```golang
    // Consume the first value from output.
    out := merge(c1, c2)
    fmt.Println(<-out) // 4 or 9
    return
    // Since we didn't receive the second value from out,
    // one of the output goroutines is hung attempting to send it.
}
```
[`Смотри исходный код`](https://github.com/Konstantin8105/Go-pipelines/blob/master/pipelines/sqleak.go)

Это утечка ресурсов: горутины потребляют память и ресурсы времени выполнения, а кучи ссылок в стэке горутины не позволяют собирать мусор.
Горутины не собираются сборщиком мусора; они должны выйти сами по себе.

Нам нужно организовать выходные этапы нашего конвейера, чтобы выйти, даже если нисходящие этапы не смогут получить все входящие значения. Один из способов сделать это - изменить исходящие каналы на наличие буфера. Буфер может содержать фиксированное количество значений; операции отправки выполняются немедленно, если в буфере есть место

```golang
        c := make(chan int, 2) // buffer size 2
        c <- 1  // succeeds immediately
        c <- 2  // succeeds immediately
        c <- 3  // blocks until another goroutine does <-c and receives 1
```

Когда количество отправляемых значений известно во время создания канала, буфер может упростить код. Например, мы можем переписать `gen`, чтобы скопировать список целых чисел в буферный канал и избежать создания новой горутины:

```golang
func gen(nums ...int) <-chan int {
	out := make(chan int, len(nums))
	for _, n := range nums {
		out <- n
	}
	close(out)
	return out
}
```
[`Смотри исходный код`](https://github.com/Konstantin8105/Go-pipelines/blob/master/pipelines/sqbuffer.go)

Возвращаясь к заблокированным горутинам в нашем конвейере, мы можем рассмотреть возможность добавления буфера в исходящий канал, возвращаемый `merge`:

```golang
func merge(cs ...<-chan int) <-chan int {
	var wg sync.WaitGroup
	out := make(chan int, 1) // enough space for the unread inputs
	// ... the rest is unchanged ...
```
[`Смотри исходный код`](https://github.com/Konstantin8105/Go-pipelines/blob/master/pipelines/sqbuffer.go)

Хотя это исправляет заблокированный горутины в этой программе, это плохой код. Выбор размера буфера 1 здесь зависит от того, будет ли полученное количество значений `merge` будет получено, и количество уровней, которые будут использоваться ниже по течению. Это хрупко: если мы передаем дополнительное значение в `gen`, или если нижестоящий этап считывает меньшее количество значений, мы снова заблокируем горутину.

Вместо этого нам необходимо предоставить путь для последующих этапов, чтобы указать отправителям, что они перестанут принимать входные данные.

## Явная отмена

Когда `main` решает выйти без получения всех значений из` out`, он должен сообщить горутинам на этапах выше по течению отказаться от значений, которые они пытаются отправить. Он делает это, отправляя значения в канал, называемый `done`. Он отправляет два значения, поскольку есть потенциально два заблокированных отправителя:

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

Отправляющие горутины заменяют свою операцию отправки оператором `select`, который выполняется либо при отправке на` out`, либо при получении значения из `done`. Тип значения `done` - это пустая структура, потому что значение не имеет значения: это событие получения, указывающее на отправку на `out`, должно быть отменено. Горутины `output` продолжают цикл на своем входящем канале `c`, поэтому этапы восходящего потока не блокируются. (Мы обсудим в какой-то момент, как разрешить этот цикл вернуться раньше.)

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

Этот подход имеет проблему: *каждый* нисходящий приемник должен знать количество потенциально заблокированных отправителей восходящего потока и организовать сигнализацию этих отправителей при раннем возврате. Отслеживание этих показателей является утомительным и подвержен ошибкам.

Нам нужен способ сообщить неизвестному и неограниченному количеству горутин прекратить отправку своих значений вниз по течению. В Go мы можем сделать это, закрыв канал, потому что [операция приема на закрытом канале всегда может действовать немедленно, что дает нулевое значение типа элемента.](http://golang.org/ref/spec#Receive_operator)

Это означает, что `main` может разблокировать всех отправителей, просто закрыв канал `done`. Это близко к  широковещательным сигналом отправителям. Мы расширяем *каждого* наших функций конвейера, чтобы принять `done` в качестве параметра и организовать близость с помощью инструкции `defer`, так что все пути возврата от `main` будут сигнализировать о завершении этапа конвейера

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

Каждый из наших этапов конвейера теперь может вернуться, как только `done` будет закрыт.
Процедура `output` в `merge` может возвращаться без дренирования входящего канала, так как она знает, что отправитель вверх, `sq`, перестанет пытаться отправить, когда `done` будет закрыт. `output` гарантирует, что `wg.Done` вызывается для всех путей возврата через оператор `defer`:

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

Аналогично, `sq` может вернуться, как только `done` будет закрыт. `sq` гарантирует, что канал `out` закрыт на всех обратных путях с помощью инструкции `defer`:

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

Вот рекомендации по разработке конвеера:

- этапы закрывают свои исходящие каналы, когда выполняются все операции отправки.
- этапы продолжают получать значения от входящих каналов до тех пор, пока эти каналы не будут закрыты или отправители не будут разблокированы.

Конвейеры разблокируют отправители либо путем обеспечения достаточного количества буфера для всех отправленных значений, либо путем явного оповещения отправителей, когда приемник может отказаться от канала.

## Прохождение по дереву

Рассмотрим более реалистичный конвейер.

MD5 - это алгоритм сокращения сообщений до короткого значения, который полезен в качестве контрольной суммы файла. Утилита командной строки `md5sum` печатает значения для списка файлов.

```
	% md5sum *.go
	d47c2bbc28298ca9befdfbc5d3aa4e65  bounded.go
	ee869afd31f83cbb2d10ee81b2b831dc  parallel.go
	b88175e65fdcbc01ac08aaf1fd9b5e96  serial.go
```

Наша программа для примера похожа на `md5sum`, но вместо этого принимает один каталог в качестве аргумента и печатает значения дайджеста для каждого файла в этом каталоге, отсортированного по имени пути.

```
	% go run serial.go .
	d47c2bbc28298ca9befdfbc5d3aa4e65  bounded.go
	ee869afd31f83cbb2d10ee81b2b831dc  parallel.go
	b88175e65fdcbc01ac08aaf1fd9b5e96  serial.go
```

Основная функция нашей программы вызывает вспомогательную функцию `MD5All`, которая возвращает карту из имени пути в значение, затем сортирует и печатает результаты:

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

Функция `MD5All` находится в центре нашего обсуждения. В реализации [serial.go](https://github.com/Konstantin8105/Go-pipelines/blob/master/pipelines/serial.go) не используется параллелизм и просто считывает и суммирует каждый файл при прохождении дерева.

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

## Параллельная реализация

В реализации [parallel.go](https://github.com/Konstantin8105/Go-pipelines/blob/master/pipelines/parallel.go), мы разделим `MD5All` на двухступенчатый конвейер. Первый этап, `sumFiles`, ходит по дереву, переваривает каждый файл в своей горутине и отправляет результаты по каналу со значением типа `result`:

```golang
// A result is the product of reading and summing a file using MD5.
type result struct {
	path string
	sum  [md5.Size]byte
	err  error
}
```
[`Смотри исходный код`](https://github.com/Konstantin8105/Go-pipelines/blob/master/pipelines/parallel.go)

`sumFiles` возвращает два канала: один для `результатов` и другой для ошибки, возвращаемой `filepath.Walk`. Функция `walk` запускает новую горутину для обработки каждого файла, а затем проверяет `done`. Если `done` будет закрыто, проход по файлам немедленно прекратится:

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

`MD5All` получает значение из `c`. `MD5All` при ошибки, закрывая `done` через `defer`:

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

## Ограниченный параллелизм

Реализация `MD5All` в [parallel.go](https://github.com/Konstantin8105/Go-pipelines/blob/master/pipelines/parallel.go) запускает новую горутину для каждого файла. В каталоге со большими количеством файлами это может привести к выделению большого объема памяти, чем доступно на машине.

Мы можем ограничить эти распределение, ограничивая количество файлов, прочитаемых параллельно. В [bounded.go](https://github.com/Konstantin8105/Go-pipelines/blob/master/pipelines/bounded.go) мы делаем это, создавая фиксированное количество горутин для чтения файлов. Наш конвеер теперь имеет три этапа: ходить по дереву, читать и просчитывать файлы, а также собирать окончательные значения.

Первый этап, `walkFiles`, результатом которого является пути файлов в дереве:

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

Средний этап запускает фиксированное количество горутин(считоводов, `digester`), которые получают имена файлов из «путей» и отправляют «результаты» в канал `c`:

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

В отличие от предыдущих примеров, `digester` не закрывает свой выходной канал, поскольку несколько горутин отправляют по общему каналу. Вместо этого код в `MD5All` упорядочивает канал, который будет закрыт, когда все `digesters` будут выполнены:

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

We could instead have each digester create and return its own output channel, but then we would need additional goroutines to fan-in the results.

The final stage receives all the `results` from `c` then checks the error from `errc`.  This check cannot happen any earlier, since before this point, `walkFiles` may block sending values downstream:

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

This article has presented techniques for constructing streaming data pipelines in Go.  Dealing with failures in such pipelines is tricky, since each stage in the pipeline may block attempting to send values downstream, and the downstream stages may no longer care about the incoming data.  We showed how closing a channel can broadcast a "done" signal to all the goroutines started by a pipeline and defined guidelines for constructing pipelines correctly.

Further reading:

- [Go Concurrency Patterns](http://talks.golang.org/2012/concurrency.slide#1) ([video](https://www.youtube.com/watch?v=f6kdp27TYZs)) presents the basics of Go's concurrency primitives and several ways to apply them.
- [Advanced Go Concurrency Patterns](http://blog.golang.org/advanced-go-concurrency-patterns) ([video](http://www.youtube.com/watch?v=QDDwwePbDtw)) covers more complex uses of Go's primitives, especially `select`.
- Douglas McIlroy's paper [Squinting at Power Series](http://swtch.com/~rsc/thread/squint.pdf) shows how Go-like concurrency provides elegant support for complex calculations.
