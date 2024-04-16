# Переключение потоков  
Для переключения между потоками необходимо:  
* Сохранить контекст исполняемого потока
* Восстановить контекст потока, на который мы переключаемся  
## Функция переключения потоков на ассемблере  
```Assembler
 .text
	.code64
	.global __switch_threads

__switch_threads:
	/* Сохраняем контекст на стек */
	/* В 64-битной версии x86 сохраняем именно эти регистры, т.к. остальные сохраняет вызывающий код */
	pushq %rbx
	pushq %rbp
	pushq %r12
	pushq %r13
	pushq %r14
	pushq %r15
	pushfq /* Сохранение флагового регистра */

	/* (%rdi) - разименовали rdi. Получается значение rsp положили по месту, на которое указывает значение rdi. */
	/* rdi - указатель на указатель, первый аргумент switch_threads, см код на С ниже */
	movq %rsp, (%rdi)
	/* rdi - первый аргумент функции, а rsi - второй. Не мы их так задали, это в ассемблере всегда так жестко задано */
	/* В rdi по С коду находится указатель на контекст вызываемого потока */
	movq %rsi, %rsp

	/* Восстановление контекста на стеке */
	popfq
	popq %r15
	popq %r14
	popq %r13
	popq %r12
	popq %rbp
	popq %rbx

	/* Передаем управление назад. Как мы помним, retq достает адрес возврата со стека, но мы поменяли стек на строке movq %rsi, %rsp */
	/* То есть сейчас не в том стеке находимся, который вызывал функцию. retq достанет адрес возврата с нового стека. */
	/* Так она передает управление потоку, на который переключаемся. */
	/* Если захотим вернуться обратно в первый поток, надо будет сделать switch_threads во 2-ом */
	retq
```
## Ее интеграция в код на С   
```
#include <assert.h>
#include <stddef.h>
#include <stdint.h>
#include <stdlib.h>
#include <string.h>
#include <stdio.h>


struct switch_frame { // Структура контекста
// Расположили все флаги в обратном порядке, т.к. достаем их из стека (то, что мы сохранили первым, хранится по большим адресам, а то что последним - по меньшим) 
	uint64_t rflags;
	uint64_t r15;
	uint64_t r14;
	uint64_t r13;
	uint64_t r12;
	uint64_t rbp;
	uint64_t rbx;
	uint64_t rip; // Прямо перед контекстом сохраняется адрес возврата
} __attribute__((packed));


struct thread { // Будем использовать как дескриптор потока. Когда хотим переключиться с одного потока на другой, используем указатели.
	void *context;
};


void switch_threads(struct thread *from, struct thread *to) // Использует дескрипторы потока в качестве параметров для переключения
{
	void __switch_threads(void **prev, void *next); // *prev - куда нужно сохранить указатель (адрес) на контекст потока, с которого переключаемся
// next - указатель на сохраненный контекст потока, на который переключаемся

	__switch_threads(&from->context, to->context);
}

struct thread *__create_thread(size_t stack_size, void (*entry)(void))
{
	const size_t size = stack_size + sizeof(struct thread);
	struct switch_frame frame;
	struct thread *thread = malloc(size);

	if (!thread)
		return thread;

	// Инициализация начального контекста на стеке (все нули, кроме rip, куда сохраняем точку входа (адрес той функции, с которой начнется новый поток))
	memset(&frame, 0, sizeof(frame));
	frame.rip = (uint64_t)entry;

	thread->context = (char *)thread + size - sizeof(frame);
	memcpy(thread->context, &frame, sizeof(frame));
	return thread;
}

struct thread *create_thread(void (*entry)(void))
{
	const size_t default_stack_size = 4096;

	return __create_thread(default_stack_size, entry);
}

void destroy_thread(struct thread *thread)
{
	free(thread);
}


static struct thread _thread0;
static struct thread *thread[3];


static void thread_entry1(void)
{
	printf("In thread1, switching to thread2...\n");
	switch_threads(thread[1], thread[2]);
	printf("Back in thread1, switching to thread2...\n");
	switch_threads(thread[1], thread[2]);
	printf("Back in thread1, switching to thread0...\n");
	switch_threads(thread[1], thread[0]);
}

static void thread_entry2(void)
{
	printf("In thread2, switching to thread1...\n");
	switch_threads(thread[2], thread[1]);
	printf("Back in thread2, switching to thread1...\n");
	switch_threads(thread[2], thread[1]);
}

int main(void)
{
	thread[0] = &_thread0;
	thread[1] = create_thread(&thread_entry1);
	thread[2] = create_thread(&thread_entry2);

	printf("In thread0, switching to thread1...\n");
	switch_threads(thread[0], thread[1]);
	printf("Retunred to thread 0\n");

	destroy_thread(thread[2]);
	destroy_thread(thread[1]);

	return 0;
}
```
***Как создать новый поток и переключиться на него?***  
Проблема тут в том, что поток не запускался, соответственно нет информации о его контексте.  
Для создания потока достаточно выделить место под его стек. Также нужно сохранить на стеке начальный контекст и сохранить указатель на него. 
Так как ни разу не вызывали новый поток, должны `rip` (адрес возврата) задать сами осмысленно и записать туда адрес функции, с которой начнется выполнение потока 
(мы возвращаемся в новый поток, когда вызываем `switch_threads` из старого).  
То есть `rip` - точка входа в новый поток.
