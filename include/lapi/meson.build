# SPDX-License-Identifier: GPL-2.0-or-later
# Copyright (C) 2023 SUSE LLC Andrea Cervesato <andrea.cervesato@suse.com>

prog_python = find_program('python3')

syscalls_h = custom_target(
    'syscalls.h',
    input : 'syscalls/gen-syscalls.py',
    output : 'syscalls.h',
    command : [prog_python, '@INPUT@', '@OUTPUT@'],
    install : true,
    install_dir : 'include/lapi',
)

ltp_sources += syscalls_h

#############################
# Install headers
#############################
install_headers(
    'abisize.h',
    'acct.h',
    'bpf.h',
    'capability.h',
    'close_range.h',
    'common_timers.h',
    'cpuid.h',
    'cpuset.h',
    'cryptouser.h',
    'dccp.h',
    'epoll.h',
    'execveat.h',
    'faccessat.h',
    'fallocate.h',
    'fanotify.h',
    'fcntl.h',
    'fnmatch.h',
    'fs.h',
    'fsmount.h',
    'fsverity.h',
    'futex.h',
    'getrandom.h',
    'if_addr.h',
    'if_alg.h',
    'if_ether.h',
    'if_packet.h',
    'init_module.h',
    'io_pgetevents.h',
    'io_uring.h',
    'ioctl.h',
    'ioctl_ns.h',
    'ioprio.h',
    'iovec.h',
    'ip_tables.h',
    'ipc.h',
    'ipcbuf.h',
    'kcmp.h',
    'keyctl.h',
    'loop.h',
    'membarrier.h',
    'memfd.h',
    'mlock2.h',
    'mmap.h',
    'mount.h',
    'msg.h',
    'msgbuf.h',
    'name_to_handle_at.h',
    'netinet_in.h',
    'numaif.h',
    'openat2.h',
    'personality.h',
    'pidfd.h',
    'posix_clocks.h',
    'posix_types.h',
    'prctl.h',
    'preadv2.h',
    'pwritev2.h',
    'quotactl.h',
    'readdir.h',
    'rt_sigaction.h',
    'rtnetlink.h',
    'safe_rt_signal.h',
    'sched.h',
    'sctp.h',
    'seccomp.h',
    'securebits.h',
    'seek.h',
    'sem.h',
    'sembuf.h',
    'setns.h',
    'shm.h',
    'shmbuf.h',
    'signal.h',
    'socket.h',
    'splice.h',
    'stat.h',
    'sync_file_range.h',
    'syncfs.h',
    'tcp.h',
    'tee.h',
    'termbits.h',
    'timerfd.h',
    'timex.h',
    'tty.h',
    'udp.h',
    'uinput.h',
    'userfaultfd.h',
    'ustat.h',
    'utime.h',
    'utsname.h',
    'vm_sockets.h',
    'vmsplice.h',
    'watch_queue.h',
    'xfrm.h',
    install_dir : 'include/lapi',
)
