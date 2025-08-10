##include <stddef.h>  // For size_t
#include <stdint.h>  // For uintptr_t

#define MEMORY_SIZE 102400  // 100KB
#define ALIGN4(x) (((((x) - 1) >> 2) << 2) + 4)  // Align to 4 bytes

typedef struct Block {
    size_t size;
    int free;
    struct Block *next;
} Block;

#define BLOCK_SIZE sizeof(Block)

// Memory pool
static uint8_t memory_pool[MEMORY_SIZE];
static Block *free_list = NULL;  // Start of the memory block list

void init_memory() {
    free_list = (Block *)memory_pool;
    free_list->size = MEMORY_SIZE - BLOCK_SIZE;
    free_list->free = 1;
    free_list->next = NULL;
}

// Allocate memory
void *allocate(size_t size) {
    size = ALIGN4(size);
    Block *curr = free_list;

    while (curr) {
        if (curr->free && curr->size >= size) {
            // If big enough to split
            if (curr->size >= size + BLOCK_SIZE + 4) {
                Block *new_block = (Block *)((uint8_t *)curr + BLOCK_SIZE + size);
                new_block->size = curr->size - size - BLOCK_SIZE;
                new_block->free = 1;
                new_block->next = curr->next;

                curr->size = size;
                curr->next = new_block;
            }
            curr->free = 0;
            return (uint8_t *)curr + BLOCK_SIZE;
        }
        curr = curr->next;
    }
    return NULL;  // No suitable block found
}

// Deallocate memory
void deallocate(void *ptr) {
    if (!ptr) return;

    Block *block_ptr = (Block *)((uint8_t *)ptr - BLOCK_SIZE);
    block_ptr->free = 1;

    // Merge adjacent free blocks
    Block *curr = free_list;
    while (curr && curr->next) {
        if (curr->free && curr->next->free) {
            curr->size += BLOCK_SIZE + curr->next->size;
            curr->next = curr->next->next;
        } else {
            curr = curr->next;
        }
    }
}
 
