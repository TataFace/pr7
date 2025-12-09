#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>
#include <unistd.h>
#include <time.h>

#define MAX_TASKS 10
#define NUM_PRODUCERS 3
#define NUM_CONSUMERS 2

// Структура задачи
struct Task {
    int id;
    int data;
};

// Глобальные переменные
pthread_mutex_t task_mutex = PTHREAD_MUTEX_INITIALIZER;
pthread_cond_t task_cond = PTHREAD_COND_INITIALIZER;
pthread_cond_t space_cond = PTHREAD_COND_INITIALIZER;

struct Task task_queue[MAX_TASKS];
int task_count = 0;
int task_head = 0;
int task_tail = 0;
int task_id_counter = 0;

int stop_system = 0;
int produced_count = 0;
int consumed_count = 0;

// Функция случайной задержки
void random_delay(int max_ms) {
    int ms = (rand() % max_ms) + 1;
    usleep(ms * 1000);
}

// Функция добавления задачи
void add_task(struct Task task) {
    pthread_mutex_lock(&task_mutex);
    
    // Ждём, пока не появится место в очереди
    while (task_count >= MAX_TASKS && !stop_system) {
        pthread_cond_wait(&space_cond, &task_mutex);
    }
    
    if (!stop_system) {
        task_queue[task_tail] = task;
        task_tail = (task_tail + 1) % MAX_TASKS;
        task_count++;
        produced_count++;
        
        printf("Добавлена задача %d (данные: %d). Очередь: %d задач\n", 
               task.id, task.data, task_count);
        
        // Сигнализируем потребителям
        pthread_cond_signal(&task_cond);
    }
    
    pthread_mutex_unlock(&task_mutex);
}

// Функция получения задачи
struct Task get_task() {
    struct Task task = {0, 0};
    
    pthread_mutex_lock(&task_mutex);
    
    if (task_count > 0) {
        task = task_queue[task_head];
        task_head = (task_head + 1) % MAX_TASKS;
        task_count--;
        consumed_count++;
        
        // Сигнализируем производителям о свободном месте
        pthread_cond_signal(&space_cond);
    }
    
    pthread_mutex_unlock(&task_mutex);
    
    return task;
}

// Поток-производитель
void* producer_thread(void* arg) {
    int producer_id = *((int*)arg);
    
    while (!stop_system) {
        // Создание новой задачи
        struct Task new_task;
        
        pthread_mutex_lock(&task_mutex);
        new_task.id = ++task_id_counter;
        pthread_mutex_unlock(&task_mutex);
        
        new_task.data = rand() % 1000;
        
        add_task(new_task);
        
        // Случайная задержка
        random_delay(500);
    }
    
    printf("Производитель %d завершает работу\n", producer_id);
    return NULL;
}

// Поток-потребитель (обработчик)
void* consumer_thread(void* arg) {
    int consumer_id = *((int*)arg);
    
    while (!stop_system) {
        struct Task task;
        
        pthread_mutex_lock(&task_mutex);
        
        // Ждём, пока появятся задачи
        while (task_count == 0 && !stop_system) {
            pthread_cond_wait(&task_cond, &task_mutex);
        }
        
        if (stop_system) {
            pthread_mutex_unlock(&task_mutex);
            break;
        }
        
        pthread_mutex_unlock(&task_mutex);
        
        // Получаем задачу
        task = get_task();
        
        if (task.id != 0) {
            printf("Обработчик %d обрабатывает задачу %d (данные: %d)\n", 
                   consumer_id, task.id, task.data);
            
            // Имитация обработки
            random_delay(300);
        }
    }
    
    printf("Обработчик %d завершает работу\n", consumer_id);
    return NULL;
}

// Основная функция
int main() {
    pthread_t producers[NUM_PRODUCERS];
    pthread_t consumers[NUM_CONSUMERS];
    int producer_ids[NUM_PRODUCERS];
    int consumer_ids[NUM_CONSUMERS];
    
    srand(time(NULL));
    
    printf("=== Система обработки задач с условными переменными ===\n");
    printf("Производителей: %d, Обработчиков: %d\n", NUM_PRODUCERS, NUM_CONSUMERS);
    printf("Максимальный размер очереди: %d\n\n", MAX_TASKS);
    
    // Создание потоков-производителей
    for (int i = 0; i < NUM_PRODUCERS; i++) {
        producer_ids[i] = i + 1;
        pthread_create(&producers[i], NULL, producer_thread, &producer_ids[i]);
    }
    
    // Создание потоков-потребителей
    for (int i = 0; i < NUM_CONSUMERS; i++) {
        consumer_ids[i] = i + 1;
        pthread_create(&consumers[i], NULL, consumer_thread, &consumer_ids[i]);
    }
    
    // Даём системе поработать 10 секунд
    sleep(10);
    
    // Остановка системы
    printf("\nОстановка системы...\n");
    pthread_mutex_lock(&task_mutex);
    stop_system = 1;
    pthread_cond_broadcast(&task_cond);
    pthread_cond_broadcast(&space_cond);
    pthread_mutex_unlock(&task_mutex);
    
    // Ожидание завершения потоков
    for (int i = 0; i < NUM_PRODUCERS; i++) {
        pthread_join(producers[i], NULL);
    }
    
    for (int i = 0; i < NUM_CONSUMERS; i++) {
        pthread_join(consumers[i], NULL);
    }
    
    // Вывод статистики
    printf("\n=== Статистика ===\n");
    printf("Создано задач: %d\n", produced_count);
    printf("Обработано задач: %d\n", consumed_count);
    printf("Задач в очереди: %d\n", task_count);
    
    // Очистка ресурсов
    pthread_mutex_destroy(&task_mutex);
    pthread_cond_destroy(&task_cond);
    pthread_cond_destroy(&space_cond);
    
    printf("\nПрограмма завершена\n");
    return 0;
}



























#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>
#include <unistd.h>
#include <time.h>

/* Количество потоков-обработчиков */
#define NUM_HANDLER_THREADS 3

/* Глобальные мьютексы - РАЗДЕЛЕННЫЕ */
pthread_mutex_t request_list_mutex = PTHREAD_MUTEX_INITIALIZER;  // для списка запросов
pthread_mutex_t cond_mutex = PTHREAD_MUTEX_INITIALIZER;         // для условной переменной

/* Глобальная условная переменная */
pthread_cond_t got_request = PTHREAD_COND_INITIALIZER;

int num_requests = 0;  /* количество ожидающих запросов */

/* Структура запроса */
struct request {
    int number;                 /* номер запроса */
    struct request* next;       /* указатель на следующий запрос */
};

struct request* requests = NULL;      /* начало списка запросов */
struct request* last_request = NULL;  /* последний запрос в списке */

/* 
 * Функция add_request: добавление запроса в список
 */
void add_request(int request_num) {
    struct request* a_request;
    
    /* Создаем структуру нового запроса */
    a_request = (struct request*)malloc(sizeof(struct request));
    if (!a_request) {
        fprintf(stderr, "add_request: ошибка выделения памяти\n");
        exit(1);
    }
    a_request->number = request_num;
    a_request->next = NULL;
    
    /* Блокируем мьютекс списка запросов */
    pthread_mutex_lock(&request_list_mutex);
    
    /* Добавляем запрос в конец списка */
    if (num_requests == 0) {  /* список пуст */
        requests = a_request;
        last_request = a_request;
    } else {
        last_request->next = a_request;
        last_request = a_request;
    }
    
    /* Увеличиваем счетчик запросов */
    num_requests++;
    
    printf("Добавлен запрос #%d (всего в очереди: %d)\n", 
           request_num, num_requests);
    fflush(stdout);
    
    /* Разблокируем мьютекс списка запросов */
    pthread_mutex_unlock(&request_list_mutex);
    
    /* Блокируем мьютекс условной переменной */
    pthread_mutex_lock(&cond_mutex);
    
    /* Сигнализируем о новом запросе */
    pthread_cond_signal(&got_request);
    
    /* Разблокируем мьютекс условной переменной */
    pthread_mutex_unlock(&cond_mutex);
}

/* 
 * Функция get_request: получение запроса из списка
 * Возвращает: указатель на запрос или NULL если список пуст
 */
struct request* get_request() {
    struct request* a_request = NULL;
    
    /* Блокируем мьютекс списка запросов */
    pthread_mutex_lock(&request_list_mutex);
    
    if (num_requests > 0) {
        a_request = requests;
        requests = a_request->next;
        
        if (requests == NULL) {  /* это был последний запрос */
            last_request = NULL;
        }
        
        /* Уменьшаем счетчик запросов */
        num_requests--;
    }
    
    /* Разблокируем мьютекс списка запросов */
    pthread_mutex_unlock(&request_list_mutex);
    
    return a_request;
}

/* 
 * Функция handle_request: обработка запроса
 */
void handle_request(struct request* a_request, int thread_id) {
    if (a_request) {
        printf("Поток %d обрабатывает запрос #%d\n", 
               thread_id, a_request->number);
        fflush(stdout);
        
        /* Имитация обработки запроса */
        usleep(100000 + (rand() % 200000));  /* 100-300 мс */
    }
}

/* 
 * Функция handle_requests_loop: основной цикл обработки запросов
 */
void* handle_requests_loop(void* data) {
    int thread_id = *((int*)data);
    struct request* a_request;
    
    printf("Запущен поток-обработчик #%d\n", thread_id);
    fflush(stdout);
    
    while (1) {
        /* Проверяем, есть ли запросы в очереди */
        int has_requests = 0;
        
        pthread_mutex_lock(&request_list_mutex);
        has_requests = (num_requests > 0);
        pthread_mutex_unlock(&request_list_mutex);
        
        if (has_requests) {
            /* Получаем и обрабатываем запрос */
            a_request = get_request();
            if (a_request) {
                handle_request(a_request, thread_id);
                free(a_request);
            }
        } else {
            /* Если запросов нет - ждём */
            pthread_mutex_lock(&cond_mutex);
            
            /* ДВОЙНАЯ ПРОВЕРКА для избежания состояния гонки */
            pthread_mutex_lock(&request_list_mutex);
            if (num_requests == 0) {
                /* Разблокируем мьютекс списка перед ожиданием */
                pthread_mutex_unlock(&request_list_mutex);
                
                /* Ожидаем сигнала о новом запросе */
                pthread_cond_wait(&got_request, &cond_mutex);
            } else {
                /* Разблокируем мьютекс списка */
                pthread_mutex_unlock(&request_list_mutex);
            }
            
            /* Разблокируем мьютекс условной переменной */
            pthread_mutex_unlock(&cond_mutex);
        }
    }
    
    return NULL;
}

/* Основная функция */
int main(int argc, char* argv[]) {
    int i;
    int thr_id[NUM_HANDLER_THREADS];
    pthread_t p_threads[NUM_HANDLER_THREADS];
    
    srand(time(NULL));
    
    printf("=== Сервер с пулом потоков (разделенные мьютексы) ===\n");
    printf("Количество потоков-обработчиков: %d\n\n", NUM_HANDLER_THREADS);
    
    /* Создаем потоки-обработчики */
    for (i = 0; i < NUM_HANDLER_THREADS; i++) {
        thr_id[i] = i + 1;
        pthread_create(&p_threads[i], NULL, handle_requests_loop, (void*)&thr_id[i]);
    }
    
    /* Даем потокам время на запуск */
    sleep(1);
    
    /* Генерируем запросы */
    printf("\nГенерация запросов...\n");
    for (i = 1; i <= 20; i++) {
        add_request(i);
        
        /* Случайная пауза между запросами */
        if (rand() % 3 == 0) {
            usleep(500000);  /* 500 мс */
        }
    }
    
    /* Ждем завершения обработки всех запросов */
    printf("\nОжидание завершения обработки всех запросов...\n");
    while (1) {
        pthread_mutex_lock(&request_list_mutex);
        int remaining = num_requests;
        pthread_mutex_unlock(&request_list_mutex);
        
        if (remaining == 0) {
            break;
        }
        
        usleep(100000);  /* 100 мс */
    }
    
    /* Даем дополнительное время для вывода */
    sleep(2);
    
    printf("\n=== Все запросы обработаны ===\n");
    printf("Программа продолжит работу до принудительной остановки (Ctrl+C)\n");
    
    /* Ожидаем завершения потоков (в реальной программе нужен механизм остановки) */
    for (i = 0; i < NUM_HANDLER_THREADS; i++) {
        pthread_join(p_threads[i], NULL);
    }
    
    /* Очистка ресурсов */
    pthread_mutex_destroy(&request_list_mutex);
    pthread_mutex_destroy(&cond_mutex);
    pthread_cond_destroy(&got_request);
    
    return 0;
}
