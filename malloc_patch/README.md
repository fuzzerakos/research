# Mitigating modern day heap attacks with a simple check.


`_int_malloc` already has a check for the size of a chunk that is about to be removed from fastbin (`malloc(): memory corruption (fast)`).

It is checking whether the size of the chunk falls in fast chunk size range.

This check though can be easily bypassed. Attackers are using the most significant byte of a libc address for a fake chunk size(`0x7f`).

Thus being able to overwrite hook functions or file structures from just a simple fastbin attack.

There is a simple check that can be added in `_int_malloc` that will mitigate these types of attacks.

Real chunk sizes never have their 4th bit set.
But sizes that are fake always have it set.
Thus by adding this line of code in malloc source code we can mitigate these attacks.
I am using glibc-2.27 (Ubuntu 18.04) with tcache disabled. This check can be implemented in tcache too the same way!
The check was:
```C
if (__builtin_expect (victim_idx != idx, 0))
                malloc_printerr ("malloc(): memory corruption (fast)");
```

And after this simple patch a lot of bugs will become unexploitable.

```C
if (__builtin_expect (victim_idx != idx, 0)||(chunksize (victim)&8)!=0)
                malloc_printerr ("malloc(): memory corruption (fast)");

```
Note that even though this attack is only used in 0x70 sized fastbins we need to check every fastbin because if we didn't the patch could be easily bypassed by attacking the `global_max_fast` and setting it to a larger value. This way we could use the 2 most significant bytes of a libc address as a fake chunk size.

I compiled the edited libc.
We will use this binary to compare the results.
```C
#include <stdio.h>

int main(){
        char *a = malloc(0x68);
        printf("[!] Allocated chunk\n");
        free(a);
        printf("[!] Freed chunk\n");
        *(a-8)=0x7f;
        printf("[!] Changed chunk's size to %p\n", *(a-8));

        printf("[!] Will now try calling malloc\n");
        malloc(0x68);
        printf("[+] Bypassed\n");

}
```
Without the patch:
```python
[mike@220]~/Desktop/research/malloc$ ./a.out 
[!] Allocated chunk
[!] Freed chunk
[!] Changed chunk's size to 0x7f
[!] Will now try calling malloc
[+] Bypassed
```
With the patch:
```python
[mike@220]~/Desktop/research/malloc$ patchelf --add-needed ./libc.so ./a.out
[mike@220]~/Desktop/research/malloc$ ./a.out 
[!] Allocated chunk
[!] Freed chunk
[!] Changed chunk's size to 0x7f
[!] Will now try calling malloc
malloc(): memory corruption (fast)
Aborted (core dumped)
```
Not bypassed!

@fuzzerakos
