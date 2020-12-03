## Reader Writer Problem

A typical Reader Writer Problem:
- No write and read at the same time
- Multiple reads are allowed
- Multiple writes are not allowed


```C
write() {
    // wait for no one is reading or writing
    while (reading != 0 || writing != 0) {
        /*spin*/
    }

    // increase the write_counter and write
    writing = 1
    // do write stuff
    writing = 0
}

read() {
    lock(&m);
    while (writing != 0)
        pthread_cond_wait(&cv, &m);
    
    reading++;

    /* Read here! */

    reading--;
    pthread_cond_signal(&cv);
    unlock(&m);
}
```