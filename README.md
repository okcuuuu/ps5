# PS5 4.03 Kernel Exploit
---
## Summary
This repo contains an experimental WebKit ROP implementation of a PS5 kernel exploit based on **TheFlow's IPV6 Use-After-Free (UAF)**, which was [reported on HackerOne](https://hackerone.com/reports/1441103). The exploit strategy is for the most part based on TheFlow's BSD/PS4 PoC with some changes to accommodate the annoying PS5 memory layout (for more see *Research Notes* section). It establishes an arbitrary read / (semi-arbitrary) write primitive. This exploit and its capabilities have a lot of limitations, and as such, it's mostly intended for developers to play with to reverse engineer some parts of the system.

Also note; stability is fairly low, especially compared to PS4 exploits. This is due to the bug's nature of being tied to a race condition as well as the mitigations and memory layout of the PS5. This document will contain research info about the PS5, and this exploit will undergo continued development and improvements as time goes on.

This could possibly work on 4.50 as well via substituting valid 4.50 gadget offsets + kernel slides, but that will be for future work.



## Currently Included

- Obtains arbitrary read/write and can run a basic RPC server for reads/writes (or a dump server for large reads) (must edit your own address/port into the exploit file on lines 673-677)
- Enables debug settings menu
- Gets root privileges




## Limitations
- This exploit achieves read/write, **but not code execution**. This is because we cannot currently dump kernel code for gadgets, as kernel .text pages are marked as eXecute Only Memory (XOM). Attempting to read kernel .text pointers will panic!
- As per the above + the hypervisor (HV) enforcing kernel write protection, this exploit also **cannot install any patches or hooks into kernel space**, which means no homebrew-related code for the time being.
- Clang-based fine-grained Control Flow Integrity (CFI) is present and enforced.
- Supervisor Mode Access Prevention/Execution (SMAP/SMEP) cannot be disabled, due to the HV.
- The write primitive is somewhat constrained, as bytes 0x10-0x14 must be zero (or a valid network interface).
- The exploit's stability is currently poor. More on this below.



## Future work
- [x] ~~Fix-up sockets to exit browser cleanly (top prio)~~
- [ ] Write some data patches (second prio)
  - [x] Enable debug settings
  - [x] Patch creds for uid0
  - [ ] Jailbreak w/ cr_prison overwrite

- [ ] Improve UAF reliability
- [ ] Improve victim socket reliability (third prio)
- [ ] Use a better / more consistent leak target than kqueue



## Exploit Stages
This exploit works in 4 stages, and for the most part follows the same exploit strategy as theflow's poc.
1) Trigger the initial UAF on `ip6_pktopts` and get two sockets to point to the same `pktopts` / overlap (master socket <-> overlap spray socket)
2) Free the `pktopts` on the master socket and fake it with an `ip6_rthdr` spray containing a tagged `tclass` overlap.
3) Infoleak step. Use `pktopts`/`rthdr` overlap to leak a kqueue from the 0x200 slab and `pktopts` from the 0x100 slab.
4) Arbitrary read/write step. Fake `pktopts` again and find the overlap socket to use `IPV6_RTHDR` as a read/write primitive.



## Stability Notes
Stability for this exploit is at about 30%, and has multiple potential points of failure. In order of observed descending liklihood:
1) *Stage 1* causes more than one UAF due to failing to catch one or more in the reclaim, causing latent corruption that causes a panic some time later on.
2)  *Stage 4* finds the overlap/victim socket, but the pktopts is the same as the master socket's, causing the "read" primitive to just read back the pointer you attempt to read instead of that pointer's contents. This needs some improvement and to be fixed if possible because it's really annoying.
3) *Stage 1*'s attempt to reclaim the UAF fails and something else steals the pointer, causing immediate panic.
4) The kqueue leak fails and it fails to find a recognized kernel .data pointer.



## Research Notes
- It appears based on various testing and dumping with the read primitive, that the PS5 has reverted back to 0x1000 page size compared to the PS4's 0x4000.
- It also seems on PS5 that adjacent pages rarely belong to the same slab, as you'll get vastly different data in adjacent pages. Memory layout seems more scattered.
- Often when the PS5 panics (at least in webkit context), there will be awful audio output as the audio buffer gets corrupted in some way.
- Sometimes this audio corruption persists to the next boot, unsure why.
- Similar to PS4, the PS5 will require the power button to be manually pressed on the console twice to restart after a panic.



## Contributors / Special Thanks
- [Andy Nguyen / theflow0](https://twitter.com/theflow0)
- [ChendoChap](https://github.com/ChendoChap)
- [Znullptr](https://twitter.com/Znullptr)
- [sleirsgoevy](https://twitter.com/sleirsgoevy)
- [bigboss](https://twitter.com/psxdev)
- [flatz](https://twitter.com/flat_z)
- [zecoxao](https://twitter.com/notzecoxao)
- laureeeeeee