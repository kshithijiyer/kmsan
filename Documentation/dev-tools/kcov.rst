kcov: code coverage for fuzzing
===============================

kcov exposes kernel code coverage information in a form suitable for coverage-
guided fuzzing (randomized testing). Coverage data of a running kernel is
exported via the "kcov" debugfs file. Coverage collection is enabled on a task
basis, and thus it can capture precise coverage of a single system call.

Note that kcov does not aim to collect as much coverage as possible. It aims
to collect more or less stable coverage that is function of syscall inputs.
To achieve this goal it does not collect coverage in soft/hard interrupts
and instrumentation of some inherently non-deterministic parts of kernel is
disabled (e.g. scheduler, locking).

kcov is also able to collect comparison operands from the instrumented code
(this feature currently requires that the kernel is compiled with clang).

Prerequisites
-------------

Configure the kernel with::

        CONFIG_KCOV=y

CONFIG_KCOV requires gcc 6.1.0 or later.

If the comparison operands need to be collected, set::

	CONFIG_KCOV_ENABLE_COMPARISONS=y

Profiling data will only become accessible once debugfs has been mounted::

        mount -t debugfs none /sys/kernel/debug

Coverage collection
-------------------

The following program demonstrates coverage collection from within a test
program using kcov:

.. code-block:: c

    #include <stdio.h>
    #include <stddef.h>
    #include <stdint.h>
    #include <stdlib.h>
    #include <sys/types.h>
    #include <sys/stat.h>
    #include <sys/ioctl.h>
    #include <sys/mman.h>
    #include <unistd.h>
    #include <fcntl.h>

    #define KCOV_INIT_TRACE			_IOR('c', 1, unsigned long)
    #define KCOV_ENABLE			_IO('c', 100)
    #define KCOV_DISABLE			_IO('c', 101)
    #define COVER_SIZE			(64<<10)

    #define KCOV_TRACE_PC  0
    #define KCOV_TRACE_CMP 1

    int main(int argc, char **argv)
    {
	int fd;
	unsigned long *cover, n, i;

	/* A single fd descriptor allows coverage collection on a single
	 * thread.
	 */
	fd = open("/sys/kernel/debug/kcov", O_RDWR);
	if (fd == -1)
		perror("open"), exit(1);
	/* Setup trace mode and trace size. */
	if (ioctl(fd, KCOV_INIT_TRACE, COVER_SIZE))
		perror("ioctl"), exit(1);
	/* Mmap buffer shared between kernel- and user-space. */
	cover = (unsigned long*)mmap(NULL, COVER_SIZE * sizeof(unsigned long),
				     PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
	if ((void*)cover == MAP_FAILED)
		perror("mmap"), exit(1);
	/* Enable coverage collection on the current thread. */
	if (ioctl(fd, KCOV_ENABLE, KCOV_TRACE_PC))
		perror("ioctl"), exit(1);
	/* Reset coverage from the tail of the ioctl() call. */
	__atomic_store_n(&cover[0], 0, __ATOMIC_RELAXED);
	/* That's the target syscal call. */
	read(-1, NULL, 0);
	/* Read number of PCs collected. */
	n = __atomic_load_n(&cover[0], __ATOMIC_RELAXED);
	for (i = 0; i < n; i++)
		printf("0x%lx\n", cover[i + 1]);
	/* Disable coverage collection for the current thread. After this call
	 * coverage can be enabled for a different thread.
	 */
	if (ioctl(fd, KCOV_DISABLE, 0))
		perror("ioctl"), exit(1);
	/* Free resources. */
	if (munmap(cover, COVER_SIZE * sizeof(unsigned long)))
		perror("munmap"), exit(1);
	if (close(fd))
		perror("close"), exit(1);
	return 0;
    }

After piping through addr2line output of the program looks as follows::

    SyS_read
    fs/read_write.c:562
    __fdget_pos
    fs/file.c:774
    __fget_light
    fs/file.c:746
    __fget_light
    fs/file.c:750
    __fget_light
    fs/file.c:760
    __fdget_pos
    fs/file.c:784
    SyS_read
    fs/read_write.c:562

If a program needs to collect coverage from several threads (independently),
it needs to open /sys/kernel/debug/kcov in each thread separately.

The interface is fine-grained to allow efficient forking of test processes.
That is, a parent process opens /sys/kernel/debug/kcov, enables trace mode,
mmaps coverage buffer and then forks child processes in a loop. Child processes
only need to enable coverage (disable happens automatically on thread end).

Comparison operands collection
------------------------------

Comparison operands collection is similar to coverage collection:

.. code-block:: c

    /* Same includes and defines as above. */

    /* Number of 64-bit words per record. */
    #define KCOV_WORDS_PER_CMP 4

    /*
     * The format for the types of collected comparisons.
     *
     * Bit 0 shows whether one of the arguments is a compile-time constant.
     * Bits 1 & 2 contain log2 of the argument size, up to 8 bytes.
     */

    #define KCOV_CMP_CONST          (1 << 0)
    #define KCOV_CMP_SIZE(n)        ((n) << 1)
    #define KCOV_CMP_MASK           KCOV_CMP_SIZE(3)

    int main(int argc, char **argv)
    {
	int fd;
	uint64_t *cover, type, arg1, arg2, is_const, size;
	unsigned long n, i;

	fd = open("/sys/kernel/debug/kcov", O_RDWR);
	if (fd == -1)
		perror("open"), exit(1);
	if (ioctl(fd, KCOV_INIT_TRACE, COVER_SIZE))
		perror("ioctl"), exit(1);
	/*
	* Note that the buffer pointer is of type uint64_t*, because all
	* the comparison operands are promoted to uint64_t.
	*/
	cover = (uint64_t *)mmap(NULL, COVER_SIZE * sizeof(unsigned long),
				     PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
	if ((void*)cover == MAP_FAILED)
		perror("mmap"), exit(1);
	/* Note KCOV_TRACE_CMP instead of KCOV_TRACE_PC. */
	if (ioctl(fd, KCOV_ENABLE, KCOV_TRACE_CMP))
		perror("ioctl"), exit(1);
	__atomic_store_n(&cover[0], 0, __ATOMIC_RELAXED);
	read(-1, NULL, 0);
	/* Read number of comparisons collected. */
	n = __atomic_load_n(&cover[0], __ATOMIC_RELAXED);
	for (i = 0; i < n; i++) {
		type = cover[i * KCOV_WORDS_PER_CMP + 1];
		/* arg1 and arg2 - operands of the comparison. */
		arg1 = cover[i * KCOV_WORDS_PER_CMP + 2];
		arg2 = cover[i * KCOV_WORDS_PER_CMP + 3];
		/* ip - caller address. */
		ip = cover[i * KCOV_WORDS_PER_CMP + 4];
		/* size of the operands. */
		size = 1 << ((type & KCOV_CMP_MASK) >> 1);
		/* is_const - true if either operand is a compile-time constant.*/
		is_const = type & KCOV_CMP_CONST;
		printf("ip: 0x%lx type: 0x%lx, arg1: 0x%lx, arg2: 0x%lx, "
			"size: %lu, %s\n",
			ip, type, arg1, arg2, size,
		is_const ? "const" : "non-const");
	}
	if (ioctl(fd, KCOV_DISABLE, 0))
		perror("ioctl"), exit(1);
	/* Free resources. */
	if (munmap(cover, COVER_SIZE * sizeof(unsigned long)))
		perror("munmap"), exit(1);
	if (close(fd))
		perror("close"), exit(1);
	return 0;
    }

Note that the kcov modes (coverage collection or comparison operands) are
mutually exclusive.

Remote coverage collection
--------------------------

With KCOV_REMOTE_ENABLE it's possible to collect coverage from arbitrary
kernel threads. For that the targeted section of kernel code needs to be
annotated with kcov_remote_start(unique_id) and kcov_remote_stop(). Then
ioctl(kcov, KCOV_REMOTE_ENABLE) can be used to make kcov start collecting
coverage from that code section. Multiple ids can be targeted with the same
kcov device simultaneously.

This allows to collect coverage from both types of kernel background threads:
the global ones, that are spawned during kernel boot and are always running
(e.g. USB hub_event); and the local ones, that are spawned when a user
interacts with some kernel interfaces (e.g. vhost). Collecting coverage from
both types of kernel threads requires custom annotations.

To collect coverage from a global background thread add kcov_remote_start/
kcov_remote_stop annotation with a unique id to that thread's code and then
pass this id in the handles array field of the kcov_remote_arg struct.

A tracking id for local background threads is passed through the common_handle
field of the kcov_remote_arg struct. This id gets saved to the kcov_handle
field in the thread that created kcov and needs to be passed to the newly
spawned threads via custom annotations. Those threads should be in turn
annotated with kcov_remote_start/kcov_remote_stop.

.. code-block:: c

    struct kcov_remote_arg {
	unsigned	trace_mode;
	unsigned	area_size;
	unsigned	num_handles;
	uint64_t	common_handle;
	uint64_t	handles[0];
    };

    #define KCOV_REMOTE_MAX_HANDLES		0x10000

    #define KCOV_INIT_TRACE			_IOR('c', 1, unsigned long)
    #define KCOV_ENABLE			_IO('c', 100)
    #define KCOV_DISABLE			_IO('c', 101)
    #define KCOV_REMOTE_ENABLE		_IOW('c', 102, struct kcov_remote_arg)


    #define KCOV_REMOTE_HANDLE_USB  0x4242000000000000ull

    static inline __u64 kcov_remote_handle_usb(int bus)
    {
	return KCOV_REMOTE_HANDLE_USB + (__u64)bus;
    }

    #define COVER_SIZE	(64 << 10)

    #define KCOV_TRACE_PC	0
    #define KCOV_TRACE_CMP	1

    int main(int argc, char **argv)
    {
	int fd;
	unsigned long *cover, n, i;
	uint64_t handle;

	fd = open("/sys/kernel/debug/kcov", O_RDWR);
	if (fd == -1)
		perror("open"), exit(1);
	if (ioctl(fd, KCOV_INIT_TRACE, COVER_SIZE))
		perror("ioctl"), exit(1);
	cover = (unsigned long*)mmap(NULL, COVER_SIZE * sizeof(unsigned long),
				     PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
	if ((void*)cover == MAP_FAILED)
		perror("mmap"), exit(1);
	/* Enable coverage collection from the USB bus #1. */
	arg = calloc(1, sizeof(*arg) + sizeof(uint64_t));
	if (!arg)
		perror("calloc"), exit(1);
	arg->trace_mode = KCOV_TRACE_PC;
	arg->area_size = COVER_SIZE;
	arg->num_handles = 1;
	arg->handles[0] = kcov_remote_handle_usb(1);
	if (ioctl(fd, KCOV_REMOTE_ENABLE, arg))
		perror("ioctl"), free(arg), exit(1);
	free(arg);

	/* Sleep. The user needs to trigger some activity on the USB bus #1. */
	sleep(2);

	n = __atomic_load_n(&cover[0], __ATOMIC_RELAXED);
	for (i = 0; i < n; i++)
		printf("0x%lx\n", cover[i + 1]);
	if (ioctl(fd, KCOV_DISABLE, 0))
		perror("ioctl"), exit(1);
	if (munmap(cover, COVER_SIZE * sizeof(unsigned long)))
		perror("munmap"), exit(1);
	if (close(fd))
		perror("close"), exit(1);
	return 0;
    }
