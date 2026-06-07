# Sync Breeze Enterprise 10.0.28 —  Windows Version-Independent ROP / DEP Bypass

  A remote stack buffer-overflow exploit for **Sync Breeze Enterprise 10.0.28**
  that bypasses DEP with a ROP chain built **entirely from `libspp.dll`** and resolves
  `LoadLibraryA` / `GetProcAddress` / `VirtualAlloc` dynamically at runtime — so it does
  not depend on the victim's Windows / `kernel32.dll` version as it does not rely on OS DLL offsets.

  Written while preparing for the OSED certification. The repo also includes a small
  bad-bytes scanner that can easily be adjusted to work with other binaries as well.

  > 📝 Full write-up: https://book.blindsecurity.gr/posts/sync-breeze-enterprise-10.0.28-windows-version-independent-rop-chain

  ---

  ## Legal

  This code is provided **for education and authorized security testing only**
  (CTFs, OSED/exam prep, your own lab, or systems you have **explicit written
  permission** to test). Running it against systems you do not own or are not
  authorized to test is illegal. The author accepts no liability for misuse.

  ---

  ## Target

  | | |
  |---|---|
  | **Software** | Sync Breeze Enterprise 10.0.28 |
  | **Service** | `syncbrs.exe` ("Sync Breeze Enterprise") |
  | **Port** | 80/tcp (HTTP) |
  | **Vuln** | Stack buffer overflow in the `username` field of `POST /login` |
  | **Protections** | DEP enabled on all modules; ASLR enabled on all modules **except** `libspp.dll` |
  | **Tested on** | Windows 10 x86 |
  
  ---

  ## How it works

  Assuming DEP is enabled for all modules and ASLR is off only on `libspp.dll`, which itself imports none
  of the usual `VirtualAlloc` / `VirtualProtect` / `WriteProcessMemory` functions. The
  chain works around that:

  1. Anchor on non-ASLR `libspp.dll` for all gadgets.
  2. Dereference a function `libspp.dll` imports from `libpal.dll` (`WriteStringEx`) to defeat
     `libpal.dll`'s ASLR via its fixed internal offsets.
  3. From there, resolve `LoadLibraryA` and `GetProcAddress` out of `libpal.dll`'s IAT.
  4. `LoadLibraryA("kernel32.dll")` → base of `kernel32`.
  5. `GetProcAddress(kernel32, "VirtualAlloc")` → address of `VirtualAlloc`.
  6. Call `VirtualAlloc` to flip the stack region to `PAGE_EXECUTE_READWRITE`, then
     return into the shellcode.

  Because the API addresses are resolved at runtime, the exploit is independent of the
  OS / `kernel32` version — it only depends on this specific application version's
  `libspp.dll` / `libpal.dll`.
  
  ---

  ## Files

  | File | Purpose |
  |------|---------|
  | `Sync_Breeze_Enterprise_10.0.28.py` | The exploit. Builds the payload, (optionally) attaches WinDbg, and sends it. |
  | `bad_bytes.py` | Used to automatically detect bad bytes in input buffers by parsing WinDbg output to detect EIP overwrite. |
  | `README.md` | This file. |

  ---

  ## Credits / References

  - https://www.exploit-db.com/exploits/42928
