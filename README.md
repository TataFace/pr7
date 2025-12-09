#include <stdio.h>
#include <pthread.h>
#include <stdlib.h>
#include <unistd.h>

#define NUM_HANDLER_THREADS 3

pthread_mutex_t request_mutex = PTHREAD_RECURSIVE_MUTEX_INITIALIZER_NP;
pthread_cond_t got_request = PTHREAD_COND_INITIALIZER;
int num_requests = 0;

struct request {
    int number;
    struct request* next;
};

struct request* requests = NULL;
struct request* last_request = NULL;

void add_request(int request_num) {
    struct request* a_request = (struct request*)malloc(sizeof(struct request));
    if (!a_request) {
        fprintf(stderr, "add_request: out of memory\n");
        exit(1);
    }
    a_request->number = request_num;
    a_request->next = NULL;

    pthread_mutex_lock(&request_mutex);

    if (num_requests == 0) {
        requests = a_request;
        last_request = a_request;
    } else {
        last_request->next = a_request;
        last_request = a_request;
    }
    num_requests++;

    pthread_mutex_unlock(&request_mutex);
    pthread_cond_signal(&got_request);
}

struct request* get_request() {
    struct request* a_request;
    
    pthread_mutex_lock(&request_mutex);

    if (num_requests > 0) {
        a_request = requests;
        requests = a_request->next;
        if (requests == NULL) {
            last_request = NULL;
        }
        num_requests--;
    } else {
        a_request = NULL;
    }

    pthread_mutex_unlock(&request_mutex);
    return a_request;
}

void handle_request(struct request* a_request, int thread_id) {
    if (a_request) {
        printf("Thread '%d' handled request '%d'\n", thread_id, a_request->number);
        free(a_request);
    }
}

void* handle_requests_loop(void* data) {
    struct request* a_request;
    int thread_id = *((int*)data);

    pthread_mutex_lock(&request_mutex);

    while (1) {
        if (num_requests > 0) {
            a_request = get_request();
            if (a_request) {
                pthread_mutex_unlock(&request_mutex);
                handle_request(a_request, thread_id);
                pthread_mutex_lock(&request_mutex);
            }
        } else {
            pthread_cond_wait(&got_request, &request_mutex);
        }
    }
    return NULL;
}

int main() {
    int i;
    int thr_id[NUM_HANDLER_THREADS];
    pthread_t p_threads[NUM_HANDLER_THREADS];
    struct timespec delay;

    srand(time(NULL));

    for (i = 0; i < NUM_HANDLER_THREADS; i++) {
        thr_id[i] = i;
        pthread_create(&p_threads[i], NULL, handle_requests_loop, (void*)&thr_id[i]);
    }

    sleep(3);

    for (i = 0; i < 600; i++) {
        add_request(i);
        if (rand() > 3 * (RAND_MAX / 4)) {
            delay.tv_sec = 0;
            delay.tv_nsec = 10;
            nanosleep(&delay, NULL);
        }
    }

    sleep(5);
    printf("Glory, we are done.\n");
    return 0;
}
