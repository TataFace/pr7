Вот готовые коды для вставки в отчёт и компиляции в Linux.

---

## **1. `employee_no_mutex.c` (программа БЕЗ мьютекса)**

```c
#include <stdio.h>     /* стандартный ввод/вывод */
#include <pthread.h>   /* функции и структуры pthread */
#include <string.h>    /* для strcpy */
#include <stdlib.h>    /* для exit() */

#define NUM_EMPLOYEES 2

struct employee {
    int number;
    int id;
    char first_name[20];
    char last_name[30];
    char department[30];
    int room_number;
};

/* глобальный массив сотрудников */
struct employee employees[] = {
    { 1, 12345678, "Danny", "Coresh", "Accounting", 101 },
    { 2, 87654321, "Misha", "Levyn", "Programmers", 202 }
};

/* глобальная переменная - "Сотрудник дня" */
struct employee employee_of_the_day;

/* функция копирования структуры сотрудника */
void copy_employee(struct employee* from, struct employee* to) {
    to->number = from->number;
    to->id = from->id;
    strcpy(to->first_name, from->first_name);
    strcpy(to->last_name, from->last_name);
    strcpy(to->department, from->department);
    to->room_number = from->room_number;
}

/* функция, выполняемая в потоке */
void* do_loop(void* data) {
    int my_num = *((int*)data); /* номер сотрудника для этого потока */
    while (1) {
        /* устанавливаем "Сотрудника дня" */
        copy_employee(&employees[my_num-1], &employee_of_the_day);
    }
    return NULL;
}

int main() {
    pthread_t thread1, thread2;
    int num1 = 1, num2 = 2;
    struct employee eotd;
    struct employee* worker;
    int i;

    /* инициализация первого сотрудника как "Сотрудника дня" */
    copy_employee(&employees[0], &employee_of_the_day);

    /* создание потоков */
    pthread_create(&thread1, NULL, do_loop, (void*)&num1);
    pthread_create(&thread2, NULL, do_loop, (void*)&num2);

    /* основной поток проверяет целостность данных */
    for (i = 0; i < 60000; i++) {
        copy_employee(&employee_of_the_day, &eotd);
        worker = &employees[eotd.number-1];

        if (eotd.id != worker->id) {
            printf("Ошибка 'id': %d != %d (итерация %d)\n",
                   eotd.id, worker->id, i);
            exit(1);
        }
        if (strcmp(eotd.first_name, worker->first_name) != 0) {
            printf("Ошибка 'first_name': %s != %s (итерация %d)\n",
                   eotd.first_name, worker->first_name, i);
            exit(1);
        }
        if (strcmp(eotd.last_name, worker->last_name) != 0) {
            printf("Ошибка 'last_name': %s != %s (итерация %d)\n",
                   eotd.last_name, worker->last_name, i);
            exit(1);
        }
        if (strcmp(eotd.department, worker->department) != 0) {
            printf("Ошибка 'department': %s != %s (итерация %d)\n",
                   eotd.department, worker->department, i);
            exit(1);
        }
        if (eotd.room_number != worker->room_number) {
            printf("Ошибка 'room_number': %d != %d (итерация %d)\n",
                   eotd.room_number, worker->room_number, i);
            exit(1);
        }
    }

    printf("Успех! Данные всегда оставались согласованными.\n");
    return 0;
}
```

---

## **2. `employee_with_mutex.c` (программа С мьютексом)**

```c
#include <stdio.h>
#include <pthread.h>
#include <string.h>
#include <stdlib.h>

#define NUM_EMPLOYEES 2

/* Глобальный мьютекс */
pthread_mutex_t a_mutex = PTHREAD_MUTEX_INITIALIZER;

struct employee {
    int number;
    int id;
    char first_name[20];
    char last_name[30];
    char department[30];
    int room_number;
};

struct employee employees[] = {
    { 1, 12345678, "Danny", "Coresh", "Accounting", 101 },
    { 2, 87654321, "Misha", "Levyn", "Programmers", 202 }
};

struct employee employee_of_the_day;

/* Копирование с защитой мьютексом */
void copy_employee(struct employee* from, struct employee* to) {
    pthread_mutex_lock(&a_mutex); /* ЗАХВАТ мьютекса */
    
    to->number = from->number;
    to->id = from->id;
    strcpy(to->first_name, from->first_name);
    strcpy(to->last_name, from->last_name);
    strcpy(to->department, from->department);
    to->room_number = from->room_number;
    
    pthread_mutex_unlock(&a_mutex); /* ОСВОБОЖДЕНИЕ мьютекса */
}

void* do_loop(void* data) {
    int my_num = *((int*)data);
    while (1) {
        copy_employee(&employees[my_num-1], &employee_of_the_day);
    }
    return NULL;
}

int main() {
    pthread_t thread1, thread2;
    int num1 = 1, num2 = 2;
    struct employee eotd;
    struct employee* worker;
    int i;

    copy_employee(&employees[0], &employee_of_the_day);

    pthread_create(&thread1, NULL, do_loop, (void*)&num1);
    pthread_create(&thread2, NULL, do_loop, (void*)&num2);

    for (i = 0; i < 60000; i++) {
        copy_employee(&employee_of_the_day, &eotd);
        worker = &employees[eotd.number-1];

        if (eotd.id != worker->id) {
            printf("Ошибка 'id': %d != %d (итерация %d)\n",
                   eotd.id, worker->id, i);
            exit(1);
        }
        if (strcmp(eotd.first_name, worker->first_name) != 0) {
            printf("Ошибка 'first_name': %s != %s (итерация %d)\n",
                   eotd.first_name, worker->first_name, i);
            exit(1);
        }
        if (strcmp(eotd.last_name, worker->last_name) != 0) {
            printf("Ошибка 'last_name': %s != %s (итерация %d)\n",
                   eotd.last_name, worker->last_name, i);
            exit(1);
        }
        if (strcmp(eotd.department, worker->department) != 0) {
            printf("Ошибка 'department': %s != %s (итерация %d)\n",
                   eotd.department, worker->department, i);
            exit(1);
        }
        if (eotd.room_number != worker->room_number) {
            printf("Ошибка 'room_number': %d != %d (итерация %d)\n",
                   eotd.room_number, worker->room_number, i);
            exit(1);
        }
    }

    printf("Успех! Данные всегда оставались согласованными.\n");
    return 0;
}
```

---

## **3. `mutex1.c` (простой пример с мьютексом)**

```c
#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>

void *functionC();
pthread_mutex_t mutex1 = PTHREAD_MUTEX_INITIALIZER;
int counter = 0;

int main() {
    int rc1, rc2;
    pthread_t thread1, thread2;

    /* Создаем два независимых потока */
    if ((rc1 = pthread_create(&thread1, NULL, &functionC, NULL))) {
        printf("Ошибка создания потока: %d\n", rc1);
    }
    if ((rc2 = pthread_create(&thread2, NULL, &functionC, NULL))) {
        printf("Ошибка создания потока: %d\n", rc2);
    }

    /* Ждём завершения потоков */
    pthread_join(thread1, NULL);
    pthread_join(thread2, NULL);

    exit(EXIT_SUCCESS);
}

void *functionC() {
    pthread_mutex_lock(&mutex1);   /* ЗАХВАТ */
    counter++;
    printf("Значение счётчика: %d\n", counter);
    pthread_mutex_unlock(&mutex1); /* ОСВОБОЖДЕНИЕ */
    return NULL;
}
```

---

## **4. `join1.c` (пример с функцией join)**

```c
#include <stdio.h>
#include <pthread.h>

#define NTHREADS 10

void *thread_function(void *);
pthread_mutex_t mutex1 = PTHREAD_MUTEX_INITIALIZER;
int counter = 0;

int main() {
    pthread_t thread_id[NTHREADS];
    int i, j;

    /* Создаем 10 потоков */
    for (i = 0; i < NTHREADS; i++) {
        pthread_create(&thread_id[i], NULL, thread_function, NULL);
    }

    /* Ждём завершения ВСЕХ потоков */
    for (j = 0; j < NTHREADS; j++) {
        pthread_join(thread_id[j], NULL);
    }

    /* Теперь можно безопасно вывести итоговое значение */
    printf("Итоговое значение счётчика: %d\n", counter);
    return 0;
}

void *thread_function(void *dummyPtr) {
    printf("Поток №%ld\n", pthread_self());
    pthread_mutex_lock(&mutex1);
    counter++;
    pthread_mutex_unlock(&mutex1);
    return NULL;
}
```

---

## **Инструкция для выполнения в Linux:**

1. **Создайте файлы:**
   ```bash
   nano employee_no_mutex.c
   ```
   Вставьте код №1, сохраните (Ctrl+O, Enter, Ctrl+X).  
   Аналогично создайте остальные три файла.

2. **Компиляция:**
   ```bash
   gcc -pthread employee_no_mutex.c -o no_mutex
   gcc -pthread employee_with_mutex.c -o with_mutex
   gcc -pthread mutex1.c -o mutex1
   gcc -pthread join1.c -o join1
   ```

3. **Запуск:**
   ```bash
   ./no_mutex        # скорее всего завершится с ошибкой
   ./with_mutex      # должен пройти все проверки
   ./mutex1          # вывод: 1, 2
   ./join1           # вывод ID потоков и итоговое значение 10
   ```

Эти коды готовы для вставки в отчёт и выполнения в Linux.
