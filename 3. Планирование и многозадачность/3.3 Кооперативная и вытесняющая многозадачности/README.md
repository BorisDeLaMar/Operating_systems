# Кооперативная и вытесняющая многозадачности  
Рассмотренный ранее подход предполагает добровольное переключение с потока на поток путем вызова функции перехода в исполняющемся потоке. Если он не вызовет функцию, 
то и переключения не произойдет. Такой подход называется **кооперативной (или невытесняющей) многозадачностью**.  
Если в изначальном потоке ошибка или он, например, вызвал стороннюю библиотеку для выполнения громоздкой операции, то переключиться на другой поток он сам не сможет.  

Альтернеативой такому подходу стала **вытесняющая (preemptive) многозадачность**. Идея в том, что ОС "силой" снимает поток с процессора и передает управление другому 
по истечении *кванта* времени. При этом усложняется синхронизация потоков.  
Вспомним об обработчике прерываний: он работает в контексте прерванного приложения. То есть функцию переключения контекста можно вызвать из него от имени потока.  
## Таймер  
Таймер может генерировать прерывания с заданной периодичностью. Для x86 есть следующие варианты:  
* Programmable Internal Timer (PIT или intel 8253) - использовался в IBM PC, древний, но даже сейчас можно использовать благодаря обратной совместимости
* High Precision Event Timer (HPET или медиа-таймер)
* Local APIC Timer - обычно таймер есть на современных контроллерах прерываний  
## Реализация вытесняющей многозадачности  
### Функция switch threads на ассемблере  
```Assembler
	.text
	.code64
	.global __switch_threads

__switch_threads:
	pushq %rbx
	pushq %rbp
	pushq %r12
	pushq %r13
	pushq %r14
	pushq %r15
	pushfq

	movq %rsp, (%rdi)
	movq %rsi, %rsp

	popfq
	popq %r15
	popq %r14
	popq %r13
	popq %r12
	popq %rbp
	popq %rbx

	retq
```  
### Код реализации на С  
Поскольку пользовательским приложениям пользоваться прерываниями нельзя, будем задавать сигналы.
```C
#include <assert.h>
#include <stddef.h>
#include <stdint.h>
#include <stdlib.h>
#include <string.h>
#include <stdio.h>
#include <time.h>

#include <signal.h>
#include <unistd.h>


struct switch_frame {
	uint64_t rflags;
	uint64_t r15;
	uint64_t r14;
	uint64_t r13;
	uint64_t r12;
	uint64_t rbp;
	uint64_t rbx;
	uint64_t rip;
} __attribute__((packed));


struct thread {
	void *context;
};



void switch_threads(struct thread *from, struct thread *to)
{
	void __switch_threads(void **prev, void *next);

	__switch_threads(&from->context, to->context);
}

struct thread *__create_thread(size_t stack_size, void (*entry)(void))
{
	const size_t size = stack_size + sizeof(struct thread);
	struct switch_frame frame;
	struct thread *thread = malloc(size);

	if (!thread)
		return thread;

	memset(&frame, 0, sizeof(frame));
	frame.rip = (uint64_t)entry;

	thread->context = (char *)thread + size - sizeof(frame);
	memcpy(thread->context, &frame, sizeof(frame));
	return thread;
}

struct thread *create_thread(void (*entry)(void))
{
	const size_t default_stack_size = 16384;

	return __create_thread(default_stack_size, entry);
}

void destroy_thread(struct thread *thread)
{
	free(thread);
}


static struct thread *thread[2];
static int current_thread;


static void interrupt(int unused) // Обработчик прерывания
{
	(void) unused;

	struct thread *prev = thread[current_thread++ % 2];
	struct thread *next = thread[current_thread % 2];

	alarm(1); // Вызываем обработчик раз в секунду
	printf("interrupt\n");
	switch_threads(prev, next); // Переключение потоков
}

static unsigned long long timespec2ms(const struct timespec *spec)
{
	return (unsigned long long)spec->tv_sec * 1000 + spec->tv_sec / 1000000;
}

static unsigned long long now(void)
{
	struct timespec spec;

	clock_gettime(CLOCK_MONOTONIC, &spec);
	return timespec2ms(&spec);
}

static void delay(int ms)
{
	const unsigned long long start = now();

	while (now() - start < ms); 
}

static void thread_entry1(void)
{
	while (1) {
		printf("In thread 1\n");
		delay(200);
	}
}

static void thread_entry2(void)
{
	while (1) {
		printf("In thread 2\n");
		delay(200);
	}
}

int main(void)
{
	struct sigaction action;
	struct thread th;

	/* Setup an "interrupt" handler */
	action.sa_handler = &interrupt;
	action.sa_flags = SA_NODEFER;
	sigaction(SIGALRM, &action, NULL); // Сигнал, позволяющий асинхронно исполняющемуся коду вызывать обработчик через некоторые промежутки времени.

	thread[0] = create_thread(&thread_entry1);
	thread[1] = create_thread(&thread_entry2);

	/* Start timer and kick off threading */
	alarm(1);
	switch_threads(&th, thread[0]);

	while (1);

	destroy_thread(thread[2]);
	destroy_thread(thread[1]);

	return 0;
}
```
