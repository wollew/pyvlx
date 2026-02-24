# pyvlx Bug & Code Quality Analysis

## CRITICAL Bugs

### 1. `NodeUpdater.process_frame` — opening/closing state never mutually reset

**File:** [pyvlx/node_updater.py](pyvlx/node_updater.py#L99-L126)

```python
if (position.position > target.position <= Parameter.MAX) and (
    (frame.state == OperatingState.EXECUTING)
    or frame.remaining_time > 0
):
    node.is_opening = True
    ...
elif (position.position < target.position <= Parameter.MAX) and (
    (frame.state == OperatingState.EXECUTING)
    or frame.remaining_time > 0
):
    node.is_closing = True
    ...
else:
    if node.is_opening:
        node.is_opening = False
        node.state_received_at = None
        node.estimated_completion = None
    if node.is_closing:
        node.is_closing = False
        # BUG: state_received_at and estimated_completion not reset here
```

**Problem:** Two issues:
1. When a device starts opening, `is_opening = True` is set — but `is_closing` is never reset to `False` (and vice versa). A device that was closing and then starts opening will report as *both* opening and closing simultaneously.
2. When closing stops (the `if node.is_closing` branch in `else`), `state_received_at` and `estimated_completion` are not reset, leaving stale timestamps that corrupt `movement_percent()` calculations.

**Criticality: CRITICAL** — Directly affects correctness of device state tracking in Home Assistant integrations. A device can report as simultaneously opening *and* closing, and `movement_percent()` will calculate from stale timestamps.

---

### 2. `SetUTC.__init__` — `time.time()` evaluated at import time, not call time

**File:** [pyvlx/api/set_utc.py](pyvlx/api/set_utc.py#L15)

```python
def __init__(self, pyvlx: "PyVLX", timestamp: float = time.time()):
```

**Problem:** `time.time()` is evaluated **once** at class definition / module import time, not at each instantiation. Every subsequent `SetUTC(pyvlx)` call will send the **import-time** timestamp, not the current time. For a long-running process (like Home Assistant), this means the gateway clock will be set to the time the library was first loaded — potentially hours or days in the past.

**Fix:** Use `None` as default and resolve inside the method:
```python
def __init__(self, pyvlx: "PyVLX", timestamp: float | None = None):
    super().__init__(pyvlx=pyvlx)
    self.timestamp = timestamp if timestamp is not None else time.time()
```

**Criticality: CRITICAL** — Gateway time will be silently wrong after the first call, affecting all time-dependent operations (timers, schedules, sunrise/sunset triggers).

---

### 3. `DtoLocalTime.to_payload` — `weekday` compared as property instead of called as method

**File:** [pyvlx/dataobjects.py](pyvlx/dataobjects.py#L58)

```python
if self.localtime.weekday == 6:     # BUG: comparing method object to int — always False
    payload += (0).to_bytes(1, "big")
else:
    payload += (self.localtime.weekday() + 1).to_bytes(1, "big")  # correct call with ()
```

**Problem:** `self.localtime.weekday` is a *bound method*, never equal to `6`. The `if` branch is dead code. On Sundays (weekday `6` in Python's convention), the code falls through to the `else` branch and encodes weekday as `7` instead of `0`.

**Fix:** Add parentheses: `self.localtime.weekday() == 6`

**Criticality: CRITICAL** — Serializes wrong weekday for Sundays; can corrupt time synchronization with the gateway.

---

## HIGH Bugs

### 4. `FrameGetLimitationStatusNotification.from_payload` — wrong byte slicing (1 byte instead of 2)

**File:** [pyvlx/api/frames/frame_get_limitation.py](pyvlx/api/frames/frame_get_limitation.py#L112-L113)

```python
self.min_value = payload[4:5]   # only 1 byte — should be payload[4:6]
self.max_value = payload[6:7]   # only 1 byte — should be payload[6:8]
```

**Problem:** `min_value` and `max_value` are 2-byte parameters (later used with `Position.to_percent(raw)` which accesses `raw[0]` and `raw[1]`). Slicing `[4:5]` yields only 1 byte, causing `IndexError` or silently truncated values when `to_percent()` is called.

**Criticality: HIGH** — `GetLimitation` API calls will return corrupt limitation values or crash with `IndexError`.

---

### 5. `FrameStatusRequestNotification.from_payload` — wrong byte slicing for `REQUEST_MAIN_INFO`

**File:** [pyvlx/api/frames/frame_status_request.py](pyvlx/api/frames/frame_status_request.py#L158-L159)

```python
self.target_position = Parameter(payload[7:8])     # 1 byte — needs 2
self.current_position = Parameter(payload[9:10])   # 1 byte — needs 2
```

**Problem:** `Parameter.__init__` requires exactly 2 bytes (`from_raw` validates `len(raw) != 2`). These slices provide only 1 byte, causing an immediate `PyVLXException("Position::raw_must_be_two_bytes")`.

**Criticality: HIGH** — Any `REQUEST_MAIN_INFO` status notification will crash frame parsing.

---

### 6. `VeluxDiscovery.hosts` and `infos` are class-level mutable attributes

**File:** [pyvlx/discovery.py](pyvlx/discovery.py#L35-L36)

```python
class VeluxDiscovery():
    hosts: list[VeluxHost] = []
    infos: list[AsyncServiceInfo | None] = []
```

**Problem:** All instances of `VeluxDiscovery` share the **same** list objects. Calling `self.hosts.clear()` in `_async_discover_hosts` clears hosts for *every* instance. If two `VeluxDiscovery` instances exist concurrently, they corrupt each other's state.

**Fix:** Move initialization to `__init__`:
```python
def __init__(self, zeroconf: AsyncZeroconf) -> None:
    self.zc = zeroconf
    self.hosts: list[VeluxHost] = []
    self.infos: list[AsyncServiceInfo | None] = []
```

**Criticality: HIGH** — Shared mutable state across instances leads to data corruption in concurrent-discovery scenarios.

---

### 7. `FrameCommandSendRequest.from_payload` and `FrameStatusRequestRequest.from_payload` — wrong node ID parsing

**File:** [pyvlx/api/frames/frame_command_send.py](pyvlx/api/frames/frame_command_send.py#L95)
**File:** [pyvlx/api/frames/frame_status_request.py](pyvlx/api/frames/frame_status_request.py#L41)

```python
for i in range(len_node_ids):
    self.node_ids.append(payload[42] + i)  # adds i to first node instead of reading payload[42+i]
```

**Problem:** Instead of reading each distinct node ID from the payload, this adds an incrementing offset to the *first* node ID. A command addressed to nodes `[3, 7, 12]` would be deserialized as `[3, 4, 5]`.

**Fix:** `self.node_ids.append(payload[42 + i])`

**Criticality: HIGH** — Corrupts node IDs when deserializing multi-node frames.

---

## MEDIUM Bugs

### 8. Duplicate enum value in `NodeTypeWithSubtype`

**File:** [pyvlx/const.py](pyvlx/const.py#L307-L308)

```python
VENTILATION_POINT_AIR_TRANSFER = 0x0503
VENTILATION_POINT_AIR_OUTLET = 0x0503     # same value — becomes an alias
```

**Problem:** Both entries have value `0x0503`. In Python enums, the second becomes an alias for the first. `NodeTypeWithSubtype(0x0503)` will always return `VENTILATION_POINT_AIR_TRANSFER`, making `VENTILATION_POINT_AIR_OUTLET` unreachable.

**Criticality: MEDIUM** — If these are distinct device types in the KLF protocol, devices of one type will be misidentified.

---

### 9. Duplicate entry in `node_helper` type check list

**File:** [pyvlx/node_helper.py](pyvlx/node_helper.py#L71-L73)

```python
if frame.node_type in [
    NodeTypeWithSubtype.INTERIOR_VENETIAN_BLIND,
    NodeTypeWithSubtype.VERTICAL_INTERIOR_BLINDS,
    NodeTypeWithSubtype.INTERIOR_VENETIAN_BLIND,  # <-- duplicate
]:
```

**Problem:** The third entry is a duplicate of the first. This is harmless in itself, but strongly suggests a missing type that was meant to be listed here.

**Criticality: MEDIUM** — A node type that was meant to be handled here will fall through to the "not implemented" warning.

---

### 10. `asyncio.get_event_loop()` deprecated usage

**File:** [pyvlx/pyvlx.py](pyvlx/pyvlx.py#L37)

```python
self.loop = loop or asyncio.get_event_loop()
```

**Problem:** Deprecated since Python 3.10 and scheduled for removal in 3.14+. In Python 3.12+ this emits a `DeprecationWarning` when no running loop exists. The stored `loop` reference is then threaded through `Connection`, `Heartbeat`, and all `Node` callbacks.

**Criticality: MEDIUM** — Will break on near-future Python versions (3.14+); the whole explicit loop-passing pattern is deprecated.

---

### 11. `Scene.__eq__` doesn't check type

**File:** [pyvlx/scene.py](pyvlx/scene.py#L51-L53)

```python
def __eq__(self, other: Any) -> bool:
    return self.__dict__ == other.__dict__
```

**Problem:** Comparing a `Scene` to any object with the same `__dict__` yields `True` erroneously, and comparing to an object without `__dict__` raises `AttributeError`. Should check `isinstance` and return `NotImplemented` for foreign types.

**Criticality: MEDIUM** — Can cause spurious equality or crashes when comparing scenes with non-Scene objects.

---

### 12. `Node.__eq__` and `Scene.__eq__` defined without `__hash__`

**Files:** [pyvlx/node.py](pyvlx/node.py#L69), [pyvlx/scene.py](pyvlx/scene.py#L51)

**Problem:** Defining `__eq__` without `__hash__` makes instances unhashable (Python implicitly sets `__hash__ = None`). These objects cannot be placed in sets or used as dict keys.

**Criticality: MEDIUM** — Will raise `TypeError: unhashable type` if nodes or scenes are ever used in sets or as dict keys.

---

## LOW Bugs / Code Quality Issues

### 13. Leftover `print()` statement in production code

**File:** [pyvlx/opening_device.py](pyvlx/opening_device.py#L440)

```python
print("Orientation in device: %s " % (orientation))
```

**Problem:** Debug `print` left in production code. Should be `PYVLXLOG.debug(...)`.

**Criticality: LOW** — Pollutes stdout for library consumers.

---

### 14. `DtoNetworkSetup.__str__` displays `gateway` twice instead of showing `netmask`

**File:** [pyvlx/dataobjects.py](pyvlx/dataobjects.py#L85)

```python
return '<{} ipaddress="{}" gateway="{}" gateway="{}"  dhcp="{}"/>'.format(
    type(self).__name__, self.ipaddress, self.gateway,
    self.gateway, self.dhcp    # second 'gateway' should be self.netmask
)
```

**Criticality: LOW** — Incorrect debug/log output only.

---

## Non-Pythonic Idioms

| Location | Issue |
|---|---|
| Throughout | Old-style `'{}'.format()` and `'%s' %` string formatting — prefer f-strings (project targets Python ≥3.11) |
| `parameter.py` | `Position.from_int(Position.XXX)` called repeatedly inside `Parameter.__str__` — base class (`Parameter`) references subclass (`Position`), violating the Liskov substitution principle; should use `self.from_int(...)` or cache common values |
| `exception.py` | `'%s="%s"' % (key, value)` — pre-3.6 formatting style |
| `node_helper.py` | Long chain of `if frame.node_type ==` / `if frame.node_type in` — should use a `dict` mapping `NodeTypeWithSubtype → factory callable` |
| `frame_creation.py` | Same pattern with ~70 `if command ==` branches — a `dict[Command, type[FrameBase]]` would be cleaner and more maintainable |
| `pyvlx.py` | Explicit event loop passing (`loop` parameter) — deprecated pattern; modern asyncio should use `asyncio.get_running_loop()` only when needed |
| `heartbeat.py` | `asyncio.run_coroutine_threadsafe` used to schedule on a "stored" loop — fragile; `asyncio.create_task` is preferred when already running in the loop |
| `connection.py` | `__del__` calling `self.disconnect()` — destructor that triggers async work is unreliable; prefer an explicit `async with` context manager |
| `session_id.py` | Module-level global `_last_session_id` with `global` keyword — consider a class with an atomic counter or `itertools.count` |
| `opening_device.py` | `kwargs: Any = {}` used as a catch-all dict to build parameters — loses all type safety |
| `discovery.py` | Unnecessary empty parentheses in class definitions: `class VeluxDiscovery():` and `class VeluxHost():` |
| `nodes.py`, `scenes.py` | `for i, j in enumerate(self.__nodes)` — `j` is a poor variable name for a node; prefer `for idx, existing_node in enumerate(...)` |
