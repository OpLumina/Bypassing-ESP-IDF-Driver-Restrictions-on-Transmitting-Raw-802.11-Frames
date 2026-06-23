# Bypassing ESP‑IDF-v5.3 Software Restrictions on Transmitting Raw 802.11 Management Frames (C++)
## Sources and Creds:
**The Genius Redditor u/willstunforfood** was the one who originally found this bypass, and how I was able to find out the root cause of the issue, here's the post:
https://www.reddit.com/r/WillStunForFood/comments/ot8vzl/finally_got_the_esp32_to_send_deauthentication/

---

## 1. Overview
The ESP-IDF framework contains a software-based restriction on transmitting spoofed and other custom 802.11 Management Frames. When using the `esp_wifi_80211_tx` function to transmit custom 802.11 frames, it sends the request to a function named `ieee80211_raw_frame_sanity_check`, which is a check that filters and drops most critical management frames, including Deauthentication (Deauth) Frames, Disassociation Frames, and Authentication / Association Requests. When using an ESP32 for security research, router operations, and other basic tasks, this can cause quite a few problems and must be bypassed prior to sending the frames.

**ESP‑IDF v5.3 libraries involved (found via grep):**

```
components/esp_rom/esp32c2/ld/esp32c2.rom.ld
components/esp_wifi/lib/{esp32,esp32c2,esp32c3,esp32c5,esp32c6,esp32s2,esp32s3}/libnet80211.a
components/esp_wifi/lib/esp32_host/libnet80211.a
```

---

## 2. Overriding the sanity‑check  

To bypass these restrictions, you can wrap the ieee80211_raw_frame_sanity_check to make the restriction pass to your own function, like just returning 0.

### 2.1 Stub implementation  

Add the following code **before** any call to `esp_wifi_80211_tx()` (e.g., in `main.cpp`):

```cpp
// Dummy Function
extern "C" int ieee80211_raw_frame_sanity_check(int32_t a,int32_t b,int32_t c)
{
    return 0;
}
```

### 2.2 Linker redirection  

In order to include this dummy function without build errors, you must also instruct the linker to replace the library’s definition with the stub. Two common options are to edit CMakeLists.txt to either wrap the function, or to pass the `-zmuldefs` flag:

* **Wrap mode** – `-Wl,--wrap=ieee80211_raw_frame_sanity_check`  
  The linker redirects calls to `ieee80211_raw_frame_sanity_check` to `__wrap_ieee80211_raw_frame_sanity_check`, which you provide (the stub above).

* **Multiple definitions** – `-Wl,-z,muldefs`  
  Allows several definitions of the same symbol; the linker will favor the one you supplied.

```
# Option 1: allow multiple definitions
idf_build_set_property(LINK_OPTIONS "-zmuldefs" APPEND)

# Option 2: explicit wrap (preferred for production)
idf_build_set_property(LINK_OPTIONS "--wrap=ieee80211_raw_frame_sanity_check" APPEND)
```

With this configuration, every call from `esp_wifi_80211_tx()` is resolved to the dummy function, effectively disabling the built‑in sanity check. Consequently, the ESP32 can transmit arbitrary management frames without being dropped by the SDK.

---

## 3. Production considerations  

* **Prefer `--wrap` over `-z muldefs`** – the wrap option is explicit and avoids the broader permissiveness of allowing multiple symbol definitions.  
* **Document the override** – guard it behind a CMake option (e.g., `ENABLE_RAW_MGMT_TX`) so it can be disabled for production builds.  
* **Security impact** – removing this safeguard exposes the device to malformed frame injection; restrict its use to controlled research environments.  

---  

**Summary** – By supplying a minimal stub for `ieee80211_raw_frame_sanity_check()` and linking with either `--wrap` or `-z muldefs`, researchers can bypass ESP‑IDF’s default frame‑sanity enforcement and transmit raw 802.11 management frames from an ESP32. Use responsibly and limit the override to experimental builds.
