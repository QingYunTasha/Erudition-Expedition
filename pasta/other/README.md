# Memory Management / Garbage Collection Methods
| GC Method                | Description                                                                                                                                                                         | Pros                                                                                                                                                                              | Cons                                                                                                                                                                        |
|--------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Manual Memory Management | Memory is allocated and deallocated manually by the programmer using functions like `malloc` and `free`.                                                                           | - Full control over memory management.                                                                                                                                             | - Prone to memory leaks and dangling pointers if not handled properly.<br>- Difficult to manage complex memory structures and deal with concurrency issues.                        |
| Reference Counting       | Each object tracks the number of references pointing to it, and memory is deallocated when the reference count reaches zero.                                                       | - Immediate reclamation of memory when an object is no longer referenced.<br>- Can handle circular references.                                                                   | - Overhead of maintaining reference counts for each object.<br>- Inefficient for handling objects with a large number of references or in the presence of cycles.       |
| Mark and Sweep           | A two-phase process where reachable objects are marked and then memory occupied by unmarked (unreachable) objects is freed.                                                        | - Able to handle cycles and objects with complex interdependencies.<br>- Supports dynamic memory allocation.<br>- Suitable for long-lived objects and large heaps.                | - Requires a "stop-the-world" pause during the sweep phase, leading to temporary application unresponsiveness.<br>- Fragmentation may occur over time, affecting performance. |
| Generational GC          | Divides objects into different generations based on their age, and performs GC more frequently on younger generations. Older generations are collected less frequently.           | - Reduces the frequency of GC cycles by focusing on younger generations.<br>- Improves GC efficiency and reduces pause times.<br>- Suitable for workloads with generational object lifetimes. | - Requires additional bookkeeping to track object generations.<br>- Complexity increases with the number of generations.<br>- Can result in longer pauses for older generations. |
| Concurrent GC            | GC process runs concurrently with the application, reducing or eliminating "stop-the-world" pauses.                                                                                | - Minimizes application unresponsiveness.<br>- Better suited for latency-sensitive applications.                                                                                 | - Increased complexity in managing concurrent execution and synchronization.<br>- May introduce additional overhead and impact application performance.                   |

## Automatic Garbage Collection (GC)
This is the most common method where the language runtime automatically manages memory by periodically identifying and reclaiming unused memory (garbage) through various algorithms like mark-and-sweep, generational, or concurrent garbage collection.
## Manual Memory Management
In languages like C and C++, memory management is done explicitly by the programmer using functions like malloc() and free(). The programmer is responsible for allocating and deallocating memory manually.
## Reference Counting
This approach tracks the number of references to an object. When the count reaches zero, meaning no references exist, the memory associated with the object is automatically freed.

# Error Handling Methods
## Exceptions
Many languages, including Java, C#, Python, and Ruby, use exceptions for error handling. Exceptions allow for structured and controlled handling of exceptional conditions. Errors can be raised (thrown) at one point and caught at a higher level in the program's execution stack.

## Return Values
In languages like C and Go, error handling is often done by returning error codes or error values from functions. The calling code checks the return value and handles the error condition accordingly.

## Error Flags/Status Codes
Some languages, such as C, use error flags or status codes to indicate errors. Functions return an error code or set a global error flag that the calling code can check to determine if an error occurred.

## Optional Types
Certain languages, like Swift and Kotlin, offer optional types. These types allow variables to hold either a value or a special "null" or "nil" value to indicate the absence of a value. Developers can use conditional checks to handle null values.

## Try/Catch Blocks
These blocks are used in languages with exception handling mechanisms. Code that might raise an exception is enclosed in a "try" block, and specific error-handling code is written in one or more "catch" blocks to handle specific types of exceptions.

## Panic/Recover
In Go, panic and recover mechanisms are used for error handling. A panic occurs when the program encounters an unexpected condition, and recover is used to capture and handle the panic by deferring it or logging the error.