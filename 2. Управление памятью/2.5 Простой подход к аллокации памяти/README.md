# Простой подход к аллокации памяти  
***Как распределять между пользователями (аллоцировать) память?***  
Представим, что у нас есть непрерывный участок логической памяти, мы знаем его начало и размер. Хотим реализвать функции:  
+ `void* alloc(int size)` - возвращает указатель на место размером, как минимум, size байт в этом непрерывном участке
+ `void free(void* free)` - принимает указатель и возвращает память аллокатору  
## Ограничения аллокатора памяти  
Некоторые процессоры требуют **выравнивания** указателей, поскольку не могут обращаться по невыровненным адресам. В таких случаях часто прибегают к 
*натуральному выровниванию*.  
*NB Про адрес говорят, что он **выровнен** на x байт, если он кратен x*  
*NBB **Натуральное выравнивание** - выравниваем адрес указателя на столько, на сколько хотим прочитать/записать данные в память (хотим прочитать 8 байт, значит на 8 и
выравниваем)*  
## Описание подхода к аллокации  
1) Свяжем все свободные участки памяти в список, каждый узел которого будет описывать непрерывный участок свободной памяти. Данные, необходимые для связи элементов 
в список (указатели на предыдущий и следующий элементы, размер участка), будем располагать в начале каждого элемента.
2) Указатель на список может храниться, например, в виде глобальной переменной.
3) Таким образом, достаточно будет аллокатором пройтись по списку и найти элемент достаточного размера
   * Если блок слишком большой, отрезаем от него лишь часть, равную объему необходимой памяти, + небольшой участок спереди, в котором храним метаданные.  
Это нужно, т.к. `free` не принимает размера в качестве параметра. Указатель из `alloc` при этом, понятное дело, возвращается на участок после данных о размере.
   * Если подходящего блока не нашлось, возвращаем ошибку
4) Чтобы освободитьсвободный блок, его нужно вернуть в список. Для этого используются **граничные маркеры** (boundary tags, O(1)): добавляем в начало и конец 
каждого блока информацию о том, свободен он или нет + сами метаданные пишем как в начало каждого блока, так и в конец. Тогда, перед освобождением памяти смотрим,
свободны соседи или нет, если да, то объединяем в один свободный блок. Если нет - добавляем наш блок как новый элемент. 
