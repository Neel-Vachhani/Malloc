# Custom Memory Allocator (malloc/free Implementation)

A thread-safe, segregated free list memory allocator implemented in C that mimics the behavior of standard library `malloc()` and `free()` functions with efficient memory management and coalescing.

## Overview

This project implements a custom dynamic memory allocator that manages heap memory through a segregated free list approach. The allocator requests memory from the OS in chunks and subdivides it for user allocations, implementing best-fit allocation, block splitting, and coalescing to minimize fragmentation and improve memory utilization.

## Tech Stack

- **Language**: C (GNU C11 standard)
- **Threading**: POSIX Threads (pthread)
- **System Calls**: `sbrk()` for heap expansion
- **Build System**: Make
- **Debugging**: Valgrind-compatible implementation

## Features

### Core Functionality
- **Dynamic Memory Allocation**: Custom implementations of `malloc`, `calloc`, `realloc`, and `free`
- **Segregated Free Lists**: 59 separate free lists for different size classes to optimize allocation speed
- **Memory Coalescing**: Automatic merging of adjacent free blocks to reduce fragmentation
- **Block Splitting**: Divides larger blocks when possible to satisfy smaller requests
- **Thread Safety**: Mutex-protected operations for concurrent access
- **Boundary Tags**: Bidirectional coalescing using size information in both directions

### Memory Management Strategies
- **Best-Fit Allocation**: Searches appropriate size classes for optimal block selection
- **Chunk Management**: Requests memory from OS in configurable arena sizes (default 4KB)
- **Fenceposts**: Special boundary markers prevent coalescing across chunk boundaries
- **Efficient Metadata**: Utilizes bit manipulation to store allocation state in size field

## Architecture

### Data Structures

#### Block Header
Each memory block contains metadata for tracking size, allocation state, and free list pointers:

```c
typedef struct header {
  size_t size_state;      // Size + allocation state (last 3 bits)
  size_t left_size;       // Size of left neighbor block
  union {
    struct {
      struct header * next;  // Next in free list (when unallocated)
      struct header * prev;  // Previous in free list (when unallocated)
    };
    char data[0];           // User data starts here (when allocated)
  };
} header;
```

#### Allocation States
- **UNALLOCATED**: Block is free and in a free list
- **ALLOCATED**: Block is in use by the application
- **FENCEPOST**: Boundary marker at chunk edges

### Segregated Free Lists

The allocator maintains 59 free lists:
- **Lists 0-57**: Constant-sized blocks (8, 16, 24, ..., 464 bytes)
- **List 58**: Variable-sized blocks (472+ bytes) sorted by address

This design enables O(1) lookup for small allocations and efficient best-fit for larger requests.

### Memory Layout

```
[FENCEPOST] [HEADER|DATA...] [HEADER|DATA...] ... [FENCEPOST]
     ^              ^                                    ^
   Left          Allocated                           Right
 Boundary         Block                           Boundary
```

## Implementation Details

### Allocation Algorithm

1. **Size Calculation**: Round requested size to 8-byte alignment (minimum 16 bytes)
2. **List Selection**: Calculate appropriate free list index
3. **Block Search**:
   - Search constant-sized lists for exact fit
   - Search variable-sized list for best fit
   - Request new chunk from OS if no suitable block found
4. **Block Splitting**: If block is sufficiently larger than request, split and return remainder to free list
5. **State Update**: Mark block as allocated and update boundary tags

### Deallocation Algorithm

1. **Validation**: Check for double-free attempts
2. **State Update**: Mark block as unallocated
3. **Coalescing**:
   - Check right neighbor; merge if free
   - Check left neighbor; merge if free
   - Update free list pointers accordingly
4. **List Insertion**: Place coalesced block in appropriate free list

### Coalescing Strategy

The allocator implements **immediate coalescing** on free operations:
- **Right Coalescing**: Merge with right neighbor if unallocated
- **Left Coalescing**: Merge with left neighbor if unallocated
- **Chunk Coalescing**: When new OS chunks are adjacent to existing chunks, merge them

### Thread Safety

All public-facing functions (`my_malloc`, `my_free`, `my_calloc`, `my_realloc`) are protected by a pthread mutex to ensure thread-safe operation in multi-threaded environments.


## Building and Usage

### Compilation

```bash
# Build all tests and examples
make all

# Build only tests
make tests

# Build only examples
make examples

# Clean build artifacts
make clean
```

### Running Tests

```bash
# Run all test suites
make test

# Run a specific test
python3 runtest.py -t test_name
```

### Test Suites

The project includes comprehensive test coverage:

- **Simple Tests**: Basic allocation and freeing
- **Malloc Tests**: Block splitting, multi-allocation, chunk insertion, coalescing
- **Free Tests**: Various coalescing scenarios (left, right, both sides)
- **Other Tests**: Thread safety, edge cases (zero-size, null pointer, double-free)
- **Robustness Tests**: Large allocations, stress testing

### Integration

To use the custom allocator in your programs:

```c
#include "myMalloc.h"

int main() {
    // Use my_malloc instead of malloc
    int *array = (int *)my_malloc(10 * sizeof(int));
    
    // Use my_free instead of free
    my_free(array);
    
    return 0;
}
```

## Configuration

The allocator supports compile-time configuration:

```bash
# Custom arena size (default: 4096 bytes)
gcc -DARENA_SIZE=8192 myMalloc.c -o myMalloc

# Custom number of free lists (default: 59)
gcc -DN_LISTS=100 myMalloc.c -o myMalloc
```

## Debugging Features

### Visualization

The implementation includes multiple print formatters for debugging:

- **`print_object()`**: Detailed metadata for each block
- **`print_status()`**: Allocation state visualization
- **`print_list()`**: Simple size listing
- **`freelist_print()`**: Complete free list structure
- **`tags_print()`**: Boundary tag verification

### Color Output

Enable color-coded output for easier debugging:

```bash
export MALLOC_DEBUG_COLOR=1337_CoLoRs
./your_test
```

Colors indicate:
- **Green**: Unallocated blocks
- **Blue**: Allocated blocks
- **Yellow**: Fenceposts

### Verification

Built-in structural integrity checks:
- Cycle detection in free lists
- Pointer consistency validation
- Boundary tag verification
- Double-free detection

## Performance Characteristics

- **Time Complexity**:
  - Small allocations (≤464 bytes): O(1) average case
  - Large allocations (>464 bytes): O(n) where n is number of free blocks in last list
  - Free operations: O(1) with immediate coalescing
  
- **Space Overhead**:
  - Allocated blocks: 8 bytes per block
  - Free blocks: 24 bytes per block (includes free list pointers)

## Examples

### Architecture Exploration
```c
// examples/arch_ex.c - Platform-dependent size verification
// Demonstrates importance of using sizeof() instead of hardcoded sizes
```

### Composite Data Types
```c
// examples/composite_ex.c - Struct vs Union memory layout
// Illustrates memory usage differences between composite types
```

### Constructor Attributes
```c
// examples/constructor_ex.c - Pre-main initialization
// Shows GCC constructor attribute for code execution before main()
```

## Error Handling

The allocator detects and reports:
- **Double Free**: Attempting to free already-freed memory
- **Invalid Pointers**: Freeing non-allocated addresses
- **Memory Corruption**: Detecting corrupted metadata
- **Out of Memory**: When OS cannot provide more heap space

## Limitations

- Maximum OS chunks: 1024
- Minimum allocation: 8 bytes (internal), 16 bytes after header
- Arena size: Configurable, default 4096 bytes
- Not POSIX `malloc` replacement without LD_PRELOAD wrapper


**Note**: This allocator is designed for educational purposes to demonstrate memory management concepts including dynamic allocation, fragmentation, coalescing, and data structure design for systems programming.
