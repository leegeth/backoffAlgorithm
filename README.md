## backOffAlgorithm Library

This repository contains the backoffAlgorithm library, a utility library to calculate backoff period for network operation retries (like failed network connection with server) using an exponential backoff with jitter algorithm. The backoffAlgorithm library is distributed under the [MIT Open Source License](LICENSE).

This library uses the "Full Jitter" strategy for the exponential backoff with jitter algorithm.
More information about the algorithm can be seen in the [Exponential Backoff and Jitter](https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/) AWS blog. 

Exponential backoff with jitter is typically used when retrying a failed
connection or network operation with the server. An exponential backoff with jitter helps to
mitigate the server request failure caused from network congestion or high request load on
the server by spreading out retry requests across multiple devices.
Besides, in an environment with poor connectivity, a client can get disconnected at any time. 
A backoff strategy helps the client to conserve battery by not repeatedly attempting reconnections when they are
unlikely to succeed.

## Reference example

The example below shows how to use the backoffAlgorithm library to retry a DNS resolution query for `amazon.com`.

```c
#include "backoff_algorithm.h"
#include <stdlib.h>
#include <string.h>
#include <netdb.h>

/* The maximum number of retries for the example code. */
#define RETRY_MAX_ATTEMPTS            ( 5U )

/* The maximum back-off delay (in milliseconds) for between retries in the example. */
#define RETRY_MAX_BACKOFF_DELAY_MS    ( 5000U )

/* The base back-off delay (in milliseconds) for retry configuration in the example. */
#define RETRY_BACKOFF_BASE_MS         ( 500U )

/**
 * A random number generator to provide to the backoffAlgorithm
 * library.
 *
 * This function is used in the exponential backoff with jitter algorithm
 * to calculate the backoff value for the next retry attempt.
 *
 * It is recommended to either use a True Random Number Generator (TRNG) for
 * calculation of unique back-off values in devices so that collision between
 * devices attempting retries at the same intervals can be avoided.
 * 
 * For the simplicity of the code example, this function is a pseudo 
 * random number generator.
 *
 * @return The generated random number. This example function ALWAYS succeeds
 * in generating a random number.
 */
static int32_t pseudoRng()
{
    return( rand() % ( INT32_MAX ) );
}

int main()
{
    /* Variables used in this example. */
    BackoffAlgorithmStatus_t retryStatus = BackoffAlgorithmSuccess;
    BackoffAlgorithmContext_t retryParams;
    char serverAddress[] = "amazon.com";
    uint16_t nextRetryBackOff = 0;

    int32_t dnsStatus = -1;
    struct addrinfo hints;
    struct addrinfo ** pListHead;

    /* Add hints to retrieve only TCP sockets in getaddrinfo. */
    ( void ) memset( &hints, 0, sizeof( hints ) );

    /* Address family of either IPv4 or IPv6. */
    hints.ai_family = AF_UNSPEC;
    /* TCP Socket. */
    hints.ai_socktype = ( int32_t ) SOCK_STREAM;
    hints.ai_protocol = IPPROTO_TCP;

    /* Initialize reconnect attempts and interval. */
    BackoffAlgorithm_InitializeParams( &retryParams,
                                 RETRY_BACKOFF_BASE_MS,
                                 RETRY_MAX_BACKOFF_DELAY_MS,
                                 RETRY_MAX_ATTEMPTS,
                                 pseudoRng );

    do
    {
        /* Perform a DNS lookup on the given host name. */
        dnsStatus = getaddrinfo( serverAddress, NULL, &hints, pListHead );

        /* Retry if DNS resolution query failed. */
        if( dnsStatus != 0 )
        {
            /* Get back-off value (in milliseconds) for the next retry. */
            retryStatus = BackoffAlgorithm_GetNextBackoff( &retryParams, &nextRetryBackOff );
        }
    } while( ( dnsStatus != 0 ) && ( retryStatus != BackoffAlgorithmRetriesExhausted ) );

    return dnsStatus;
}
```

## Building the library

A compiler that supports **C89 or later** such as *gcc* is required to build the library.

For instance, if the example above is copied to a file named `example.c`, *gcc* can be used like so:
```bash
gcc -I source/include example.c source/backoff_algorithm.c -o example
./example
```

*gcc* can also produce an output file to be linked:
```bash
gcc -I source/include -c source/backoff_algorithm.c
```

## Building unit tests

### Checkout Unity Submodule
By default, the submodules in this repository are configured with `update=none` in [.gitmodules](.gitmodules), to avoid increasing clone time and disk space usage of other repositories (like [amazon-freertos](https://github.com/aws/amazon-freertos) that submodules this repository).

To build unit tests, the submodule dependency of Unity is required. Use the following command to clone the submodule:
```
git submodule update --checkout --init --recursive --test/unit-test/Unity
```

### Platform Prerequisites

- For running unit tests
    - C89 or later compiler like gcc
    - CMake 3.13.0 or later
- For running the coverage target, gcov is additionally required.

### Steps to build Unit Tests

1. Go to the root directory of this repository. (Make sure that the **Unity** submodule is cloned as described [above](#checkout-unity-submodule).)

1. Create build directory: `mkdir build && cd build`

1. Run *cmake* while inside build directory: `cmake -S ../test`

1. Run this command to build the library and unit tests: `make all`

1. The generated test executables will be present in `build/bin/tests` folder.

1. Run `ctest` to execute all tests and view the test run summary.
