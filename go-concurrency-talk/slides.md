<!-- $theme: default -->

Go and concurrent programs
==

André Luiz Alves Moraes

```bash
➜  go-concurrency-talk git:(master) ✗ whoami
programmer since 2001 (+/-)
go programmer since Nov/2009
```

---

Introdução
==

Go foi projetada para simplificar a tarefa de escrever programas concorrents eviando a criação dos problemas normalmente associados com programas multi-thread.

Go é fundamentada no modelo CSP (communicatin sequential process) proposto por [Tony Hoare](http://www.cs.ox.ac.uk/people/tony.hoare/). No modelo CSP os dados são compartilhados enviando mensagens através de canais.

Go utiliza `goroutines` que são executadas em `threads`. Cada `goroutine` roda de forma independente das outras e utilizam um ou mais objetos `chan` para passar mensagens. Uma mensagem pode ter qualquer tipo incluindo outro `chan` ou uma `closure`.

---

Hello world concorrente - um relógio
==

```go
package main

import "fmt"
import "time"

func printNow(t time.Time) {
	fmt.Printf("time now: %v\n", t)
}

func main() {
	go func() {
		ticker := time.NewTicker(time.Second * 1)
		for {
			currentTime := <-ticker.C
			printNow(currentTime)
		}
	}()

	time.Sleep(time.Second * 3)
}
```

---

Channels
==

Um `canal` é um ponto de sincronização entre `goroutines`. Uma `goroutine` vai ficar bloqueada escrevendo em um canal até que outra `goroutine` leia do mesmo canal.

Ler de um canal segue a mesma lógica, uma `goroutine` irá bloquear até que outra escreva ou o canal seja fechado (retornando o `zero-value` do tipo).

---

Channels (cont.)
==

Um `canal` pode ser fechado. Isso é útil para indicar que nenhum outro valor seré escrito no canal.

Ler um `canal` fechado retorna o valor zero para o tipo do canal.

Escrever em um `canal` fechado causa um `panic` na aplicação.

---

Geradores
==

Geradores são funções que iniciam uma `goroutine` para escrever uma lista de valores em um canal que é retornado para quem acionou a função

```go
func numbers(start, finish int) <-chan int {
	out := make(chan int)
	go func() {
		for i := start; i <= finish; i++ {
			out <- i
		}
		close(out)
	}()
	return out
}

func main() {
	values := numbers(1, 1000)
	for value := range values {
		fmt.Printf("value: %v\n", value)
	}
}
```

---


Pipeline
==

Um pipeline trabalho recebendo valores em um `canal` and escrevendo em outro canal, normalmente após efetuar uma transformação no dado.

```go
func doubleFloat(out chan<- float64, input <-chan int) {
	for value := range input {
		out <- float64(value) * 2
	}
	close(out)
}
```

---


Fan-In
==

Um fan-in copia dados de N canais de entrada e escreve em um canal de saída. Normalmente um fan-in só termina quando todos os canais de entrada são fechados.

Código no próximo slide

---


Fan-In (code)

```go
type signal struct{}

func copy(done chan signal, out chan<- int, input <-chan int) {
	for value := range input {
		out <- value
	}
	close(done)
}

func fanin(out chan<- int, inputs ...<-chan int) {
	registry := make(map[chan signal]bool)
	for _, input := range inputs {
		done := make(chan signal)
		registry[done] = true
		go copy(done, out, input)
	}
	for k := range registry {
		<-k // wait for k to be closed
		delete(registry, k)
	}
	close(out)
}
```

---


Fan-Out
=======

Um fan-ou copia dados de um canal de entrada para uma lista de saídas. Um cuidado especial é necessário para evitar que leitores lentos travem todo o fluxo. Nós próximos slides veremos as alternativas.

---

Fan-Out (blocking)

```go
func fanout(in <-chan int, outputs ...chan<- int) {
	for v := range in {
    	for _, output := range outputs {
			// block while waiting for consumer
        	output <- v
        }
    }
    closeAll(outputs)
}
```

---

Fan-Out (drop without timeout)


```go
func fanout(in <-chan int, outputs ...chan<- int) {
	for v := range in {
		for _, output := range outputs {
			select {
			case output <- v:
				// consumer received the data
			default:
				// could track here how many
				// drops per output channel
			}
		}
	}
    closeAll(outputs)
}
```

---

Fan-Out (drop with timeout)


```go
func publish(output chan<- int, v int, timeout time.Duration, wg *sync.WaitGroup) {
	timer := time.NewTimer(timeout)
	select {
	case <-timer.C:
		// timeout before consumer reading data
	case output <- v:
		// consumer read data before timeout
	}
	wg.Done()
	t.Stop()
}

func fanout(in <-chan int, timeout time.Duration,
	outputs ...chan<- int) {
	var wg sync.WaitGroup
	for v := range in {
		wg.Add(len(output))
		for _, output := range outputs {
			go publish(output, v, timeout, &wg)
		}
		wg.Wait()
	}
    closeAll(outputs)
}
```

---

Sliding window
===

Uma slinding window é utilizada para prevenir que um leitor lento trave um escritor rápido. Ela funciona deslizando sobre os dados. A ordem de entregas é garantida porém dados antigos podem ser descartados se o consumidor for muito lento.

---

Sliding window (code)

```go
func slidingWindow(out chan<- interface{}, in <-chan interface{}, sz int) {
	buf := list.New()
	defer close(out)
	for in != nil || buf.Len() > 0 {
		if buf.Len() == 0 {
			// we have an empty buffer
			// and a valid input channel
			val := <-in
			if val == nil { // assume a nil means close
				in = nil // won't read data anymore
				continue
			}
			buf.PushBack(val)
			continue
		}
		// see next slide....
	}
}
```

---
Sliding window (code cont..)
```go
	for in != nil || buf.Len() > 0 {
		// ... code from previous slide
		select {
		case out <- buf.Front():
			// consumer read data
			buf.Remove(buf.Front()) // remove first item
		case val := <-in:
			// received new input
			if val == nil {
				// invalidate input
				in = nil
				// continue since we might have data
				// on the buffer
				continue
			}
			if buf.Len() == sz {
				// buffer full, discard old data
				buf.Remove(buf.Front())
			}
			// add new data to the end
			buf.PushBack(val)
		}
	}
```

---

Bulk/Batch
===

Um `batch` é usado quando uma `goroutine` gera itens um-por-um mas o consumidor deseja processar os items em `blocos`. Normalmente um `canal de conclusão` é usado para notificar o escritor que o item foi processado. Um `canal de flush` pode user usado para forçar que o buffer seja enviado antes que ele esteja cheio.

Exemplo: Ao invés de salvar cada item no banco de dados assim que ele é recebido, é possível utilizar um buffer de 100 itens ou 100ms e salvar os itens em uma única requisição.

Código no próximo slide

---

Bulk/Batch (cont...)

```go
func bulk(output chan<- []req, input <-chan req, flush <-chan struct{}, bulk int) {
	defer close(output)
	buf := make([]req, 0, bulk)
	var closed bool
	for !closed {
		var shouldFlush bool
		select {
		case r := <-input:
			if r == req{} {
				// close on zero value
				closed = true
				continue
			}
			buf = append(buf, r)
			shouldFlush = len(buf) == bulk
		case <-flush:
			shouldFlush=true
		}
		if shouldFlush {
			output <- buf
			buf = buf[:0]
		}
	}
	if len(buf) > 0 {
		output <- buf
	}
}
```

---

Bulk/Batch (cont...)

```go
type req struct {
	value int
	done chan struct{}
}

func bulkProcess(input <-chan []req) {
	for batch := range input {
		doSomething(batch)
		for _, r := range batch {
			close(r.done)
		}
	}
}

func writeItem(output chan req) {
	r := req{value: 1, done: make(chan struct{})}
	// send the item to be processed
	output <- r
	// block until the item is processed
	<-r.done
}
```

---

Ticketing system
===

Um sistema de ticket é usado para controlar quando um determinado trabalho pode ser executado, normalmente é utilizado para limitar o uso de um recurso sobre um período de tempo.

Exemplo: Uma API pode ser acionada apenas 15 vezes em um perído de 15 minutos. A utilização é medida em blocos de 15 minutos.

---

Ticketing system

```go
type (
	Work   func()
	ticket int
)

func worker(tickets <-chan ticket, work <-chan Work) {
	for w := range work {
		<-tickets   // wait for a ticket
		w.Perform() // do some work
	}
}

func ticket(tickets chan<- ticket, timeout time.Duration, nTickets int) {
	for {
		for i := 0; i < nTickets; i++ {
			tickets <- i
		}
		// wait until more tickets can be issued
		<-time.After(timeout)
	}
}
```

---

Run On My goroutine
===

Normalmente utilizado quando deseja que um trecho de código qualquer seja executado em uma `goroutine` específica. A maioria das rotinas de GUI só podem ser acionadas pelo `thread` principal.

Acionar `runtime.LockOSThread` força uma `goroutine` a rodar  **sempre no mesmo thread**

---
Run on my goroutine (code)

```go
func init() { runtime.LockOSThread() }

type action func()

func main() {
	actions := make(chan action)
	gui.Init()
	window := gui.MainWindow()
	go otherGoroutine(actions, window)
	for a := range action {
		if a == nil {
			break
		}
		a()
	}
	window.Close()
	gui.Close()
}
```

---

Run on my goroutine (code cont)
```go
func setTitle(w *gui.Window, t string) func() {
	return func() { w.SetTitle(t) }
}
func resize(w *gui.Window, w, h int) func() {
	return func() { w.Resize(w, h) }
}

func otherGoroutine(act chan<- action, w *gui.Window) {
	tick := time.NewTicker(time.Second * 1)
	for t := range tick.C {
		act <- setTitle(w, t.Format(time.RFC3339))
		act <- resize(w, 200, 30+t.Second())
	}
}
```

---

Go Actor
===

Similar ao `actor model` de Erlang mas com algumas diferenças:

* As mensagens são síncronas
* A fila têm um tamanho finito

---

Go Actor (code)

```go
type Actor struct {
	sayHello  chan struct{}
	sayMyName chan string
}

func (a *Actor) Run(ctx context.Context) error {
	var stop bool
	for !stop {
		select {
		case <-a.sayHello:
			println("hello world")
		case msg := <-a.sayMyName:
			println(msg)
		case <-ctx.Done():
			stop = true
		}
	}
	return ctx.Err()
}
```

---

Go Actor (code cont...)

```go
func (a *Actor) SayHello() {
	a.sayHello <- struct{}{}
}
func (a *Actor) SayMyName() {
	a.sayMyName <- "Heisenberg"
}
```

---

# Obrigado!

---

### Perguntas?
