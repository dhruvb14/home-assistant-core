# Custom integration: learnable IR remote on top of the new `infrared` platform

> Design / "issue" write‑up for a HACS custom integration that turns any IR remote
> into automation triggers, built on the **new `infrared` entity platform** added
> in Home Assistant `2026.6.0`. Investigated directly from core source on this fork.
>
> **Goal:** replace the ESP8266/ESPHome `remote_receiver` + `on_raw` lambda + MQTT
> learning setup with the native Home Assistant IR receiver, and let learned buttons
> (incl. double‑click) trigger automations.

---

## TL;DR

1. `2026.6.0` added a core **entity integration** called `infrared`
   (`homeassistant/components/infrared`). It is *internal plumbing* — like `light`
   or `event`, you don't configure it directly. It provides:
   - base entity classes for hardware integrations that **provide** an IR
     emitter/receiver (`InfraredEmitterEntity`, `InfraredReceiverEntity`), and
   - helper functions + base classes for integrations that **consume** that
     hardware (`async_send_command`, `async_subscribe_receiver`,
     `InfraredReceiverConsumerEntity`, …).
2. The **ESPHome integration already produces these entities natively** — when a
   device advertises an IR receiver over the native API, it shows up as an
   `infrared.<name>` receiver entity. **No MQTT, no `text_sensor`, no lambda JSON.**
3. A receiver entity does **not** by itself fire "button" events. It just exposes
   the raw IR timings to subscribers. So the custom integration's job is:
   subscribe → decode/fingerprint → debounce/handle repeats → detect
   single/double‑click → expose each learned button as an **`event` entity**
   (or fire a bus event) that automations can trigger on.
4. The heavy lifting (encode/decode of real protocols) lives in the
   `infrared-protocols` PyPI library (`infrared_protocols`), which core depends on
   (`infrared-protocols==6.0.0`). It has a NEC decoder; everything else is
   encode‑only today, so for arbitrary remotes you fingerprint raw timings (exactly
   what the ESPHome lambda does, just moved into Python).

---

## 1. What core actually shipped

### 1.1 The `infrared` integration

`homeassistant/components/infrared/manifest.json`:

```json
{
  "domain": "infrared",
  "name": "Infrared",
  "integration_type": "entity",
  "quality_scale": "internal",
  "requirements": ["infrared-protocols==6.0.0"]
}
```

It registers an `EntityComponent` (domain `infrared`) holding two kinds of entities:
emitters and receivers. There is **no config flow** — hardware integrations add the
entities, consumer integrations look them up.

Two module‑level discovery helpers (`homeassistant/components/infrared/__init__.py`):

```python
async_get_emitters(hass)  -> list[str]   # entity_ids of all IR emitters
async_get_receivers(hass) -> list[str]   # entity_ids of all IR receivers
```

These are what your config flow will call to let the user pick which receiver to use.

### 1.2 Provider‑side base classes (`infrared/entity.py`)

```python
class InfraredDeviceClass(StrEnum):
    EMITTER  = "emitter"
    RECEIVER = "receiver"

@dataclass(frozen=True, slots=True)
class InfraredReceivedSignal:
    timings: list[int]          # signed microseconds: +pulse(high), -space(low)
    modulation: int | None = None   # carrier frequency in Hz (e.g. 38000)

class InfraredEmitterEntity(RestoreEntity):
    # state = ISO timestamp of last command sent
    async def async_send_command(self, command: InfraredCommand) -> None: ...

class InfraredReceiverEntity(RestoreEntity):
    # state = ISO timestamp of last signal received
    def _handle_received_signal(self, signal: InfraredReceivedSignal) -> None: ...
    def async_subscribe_received_signal(self, cb) -> CALLBACK_TYPE: ...
```

Key insight for us: a **receiver's `state` is only a timestamp**. The actual payload
(`InfraredReceivedSignal.timings`) is delivered out‑of‑band to subscribers. You
**cannot** build the integration with a state‑change automation trigger on the
receiver entity — you must subscribe to the signal.

`InfraredCommand` is re‑exported from the library: `infrared_protocols.commands.Command`.
It carries `.modulation`, `.repeat_count` and `.get_raw_timings() -> list[int]`.

### 1.3 Consumer‑side helpers (`infrared/helpers.py`) — **this is our API**

```python
# Send a command to an emitter entity (by entity_id or registry uuid):
await async_send_command(hass, emitter_entity_id, command, context=None)

# Subscribe to raw signals from a receiver entity; returns an unsubscribe callable:
unsub = async_subscribe_receiver(hass, receiver_entity_id, signal_callback)
#   signal_callback(signal: InfraredReceivedSignal) -> None
```

Both raise `HomeAssistantError` if the component isn't loaded or the entity isn't
found / is the wrong type.

There are also two ready‑made base entities so you don't reinvent availability
tracking + subscription lifecycle:

- `InfraredEmitterConsumerEntity` — set `self._infrared_emitter_entity_id`, then call
  `await self._send_command(command)`. Tracks emitter availability automatically.
- `InfraredReceiverConsumerEntity` — set `self._infrared_receiver_entity_id`, implement
  `_handle_signal(self, signal: InfraredReceivedSignal)`. It auto‑subscribes when the
  receiver is available, unsubscribes when it goes unavailable, and cleans up on
  removal. **This is the base class our event entity should inherit from.**

### 1.4 How ESPHome feeds this (`esphome/infrared.py`)

The ESPHome integration maps `InfraredInfo` → `Platform.INFRARED` and creates:

- `EsphomeInfraredReceiverEntity` when the device advertises
  `InfraredCapability.RECEIVER`, which subscribes to
  `client.subscribe_infrared_rf_receive(...)` and on each event calls
  `self._handle_received_signal(InfraredReceivedSignal(timings=event.timings))`.
- `EsphomeInfraredEmitterEntity` when it advertises `TRANSMITTER`, sending via
  `client.infrared_rf_transmit_raw_timings(...)`.

Requirements observed in core: `aioesphomeapi==45.3.1` (has
`subscribe_infrared_rf_receive` / `infrared_rf_transmit_raw_timings` /
`InfraredCapability`).

> **Migration consequence for your ESP8266:** your firmware must expose the IR
> receiver through the **native ESPHome API as an infrared component** (so the device
> advertises `InfraredInfo` with the `RECEIVER` capability), instead of the
> `remote_receiver:` + `on_raw:` + `mqtt:` learning approach. Once it does, Home
> Assistant's ESPHome integration auto‑creates an `infrared.livingroom_ir_receiver`
> receiver entity — and your MQTT broker, `text_sensor`, and decoding lambda all go
> away. **Confirm your ESPHome version supports the native infrared platform before
> reflashing** (this is the device‑side half and lives in the ESPHome project, not in
> HA core). Until then, the custom integration below works against *any* integration
> that produces an `infrared` receiver entity (ESPHome, Broadlink emitters,
> `kitchen_sink` demo for testing, etc.).

### 1.5 The `infrared_protocols` library (decode/encode)

- `infrared_protocols.commands.Command` — abstract base (`modulation`, `repeat_count`,
  `get_raw_timings()`).
- Concrete protocols: `nec.NECCommand`, `sony`, `rc5`, `samsung.Samsung32Command`,
  `sharp`, `kaseikyo`, `marantz_extended`.
- **Only NEC has a decoder**: `NECCommand.from_raw_timings(timings) -> NECCommand | None`
  (validates leader, decodes 32 bits LSB‑first, checks the command checksum, counts
  repeat frames). Everything else is encode‑only right now.
- `infrared_protocols.codes.*` are device databases (Samsung TV, LG TV, Sony, Edifier,
  Marantz, …) exposing named `IntEnum` codes with a `.to_command()` helper — handy if
  you later want to *emit* to a known device.

Implication: for a no‑name remote like yours, don't rely on protocol decoding. Do the
same thing your lambda does — **quantize the raw timings into a stable bit‑string
fingerprint** and match that. Optionally *also* try `NECCommand.from_raw_timings()`
first and use `address:command` as the fingerprint when it succeeds (more robust to
jitter than raw thresholding).

---

## 2. Design of the custom integration

Working name: **`ir_remote`** (HACS custom integration, `integration_type: hub` or
`service`, single config entry per receiver).

### 2.1 Responsibilities the receiver entity does NOT do (so we must)

| Concern | ESPHome lambda did it via | Custom integration does it via |
|---|---|---|
| Get raw timings | `on_raw: lambda x` | `async_subscribe_receiver` callback → `signal.timings` |
| Fingerprint a press | `decode_pattern()` threshold >1000µs | quantize timings → bit‑string (or NEC decode) |
| Debounce / repeats | `millis()` 150/250ms windows | `loop.time()` + NEC repeat awareness |
| Single vs double‑click | `dc_last_*` 1300ms window | configurable double‑click window |
| Map code → button name | `command_map[...]` C++ map | learned mapping stored in config entry options |
| Surface to automations | MQTT publish + `text_sensor` | **`event` entity** (preferred) or bus event |

### 2.2 Recommended surface: an `event` entity per receiver

`event` entities are first‑class automation triggers, show up in the UI, support the
device‑trigger picker, and record nicely in the logbook. One event entity per receiver,
whose `event_types` is the list of learned button names (e.g.
`["button_1", "button_2", "button_1_2x", "volume_up", ...]`). On each recognized press
call `self._trigger_event(button_name, {"raw_count": n, "fingerprint": fp})`.

Automation trigger then looks like:

```yaml
triggers:
  - trigger: state
    entity_id: event.living_room_ir_receiver_buttons
    attribute: event_type
    to: button_1_2x
```

> Note: `event` entities require `event_types` to be known up front. When the user
> learns a new button, persist it to the config entry's data/options and reload the
> entry so the entity is recreated with the expanded `event_types` list.

**Simpler alternative that mirrors your current MQTT flow:** skip the event entity and
just `hass.bus.async_fire("ir_remote_button", {"receiver": ..., "button": name,
"count": n})`, then trigger automations with the `event` *trigger platform*. This needs
no pre‑declared button list, so it's the path of least resistance during the
learning phase. You can offer both.

### 2.3 Learning mode

Expose a **`button` entity** "Learn next code" (or a service `ir_remote.learn`). When
pressed it arms a one‑shot capture: the next *new* press's fingerprint is stored,
pending a name the user supplies (text helper / config‑flow step / service call with a
`name` field). Mirror your lambda's de‑dupe so a held button doesn't register 30 times.

### 2.4 Fingerprint + debounce algorithm (port of your lambda)

```python
def fingerprint(timings: list[int]) -> str:
    # Try a real protocol first for robustness.
    if (cmd := NECCommand.from_raw_timings(timings)) is not None:
        return f"nec:{cmd.address:04x}:{cmd.command:02x}"
    # Fallback: same idea as the ESPHome lambda — quantize spaces to bits.
    # Skip 2-value header and trailing pulse; classify each space by length.
    bits = []
    for i in range(2, len(timings) - 1, 2):
        space = abs(timings[i + 1]) if i + 1 < len(timings) else 0
        bits.append("1" if space > 1000 else "0")
    return "raw:" + "".join(bits)
```

Debounce / new‑press / double‑click state (use `hass.loop.time()` for monotonic time):

- gap `< 0.15 s` → repeat/bounce, ignore;
- different fingerprint **or** gap `> 0.25 s` → new press;
- same button within the double‑click window (default `1.3 s`) → emit
  `"<button>_2x"` and reset, else emit single after the window (or immediately — your
  call on UX).

All four windows (`150 ms`, `250 ms`, double‑click `1300 ms`, etc.) should be config
options so they're tunable per remote, like the constants in your lambda.

---

## 3. Skeleton code for the new repo

```
custom_components/ir_remote/
├── __init__.py
├── manifest.json
├── config_flow.py
├── const.py
├── coordinator.py        # the subscribe + decode + debounce engine
├── event.py
├── button.py             # "Learn next code"
├── services.yaml
├── strings.json
└── translations/en.json
```

### `manifest.json`

```json
{
  "domain": "ir_remote",
  "name": "IR Remote Buttons",
  "version": "0.1.0",
  "codeowners": ["@you"],
  "config_flow": true,
  "dependencies": ["infrared"],
  "documentation": "https://github.com/you/ir_remote",
  "iot_class": "local_push",
  "requirements": ["infrared-protocols==6.0.0"]
}
```

> `"dependencies": ["infrared"]` guarantees the `infrared` component (and its
> `async_subscribe_receiver` helper + `DATA_COMPONENT`) is loaded before yours.
> Pin the same `infrared-protocols` version core uses to avoid resolver conflicts;
> if you only do raw fingerprinting you can drop the requirement entirely.

### `config_flow.py` (pick a receiver)

```python
from homeassistant.components.infrared import async_get_receivers
from homeassistant.helpers.selector import EntitySelector, EntitySelectorConfig
import voluptuous as vol

class IrRemoteConfigFlow(ConfigFlow, domain=DOMAIN):
    async def async_step_user(self, user_input=None):
        if user_input is not None:
            return self.async_create_entry(
                title=user_input[CONF_NAME], data=user_input
            )
        # Restrict the picker to infrared receiver entities:
        schema = vol.Schema({
            vol.Required(CONF_NAME): str,
            vol.Required(CONF_RECEIVER): EntitySelector(
                EntitySelectorConfig(domain="infrared")
            ),
        })
        return self.async_show_form(step_id="user", data_schema=schema)
```

(`async_get_receivers(hass)` is handy if you want to validate the chosen entity is
actually a receiver, since the selector can't filter by device_class today.)

### `coordinator.py` (the engine)

```python
from homeassistant.components.infrared import (
    InfraredReceivedSignal,
    async_subscribe_receiver,
)

class IrRemoteEngine:
    def __init__(self, hass, entry):
        self.hass = hass
        self.entry = entry
        self._unsub = None
        self._learn_future: asyncio.Future[str] | None = None
        # debounce state
        self._last_fp = ""
        self._last_t = 0.0
        self._dc_fp = ""
        self._dc_t = 0.0

    @callback
    def async_start(self):
        self._unsub = async_subscribe_receiver(
            self.hass, self.entry.data[CONF_RECEIVER], self._on_signal
        )

    @callback
    def async_stop(self):
        if self._unsub:
            self._unsub()
            self._unsub = None

    @callback
    def _on_signal(self, signal: InfraredReceivedSignal) -> None:
        now = self.hass.loop.time()
        fp = fingerprint(signal.timings)
        gap = now - self._last_t
        self._last_t = now
        if gap < 0.15:                      # bounce / repeat
            return
        is_new = fp != self._last_fp or gap > 0.25
        self._last_fp = fp
        if not is_new:
            return

        # learning mode: capture & return the fingerprint
        if self._learn_future and not self._learn_future.done():
            self._learn_future.set_result(fp)
            return

        button = self.entry.options.get("codes", {}).get(fp)
        if button is None:
            return                          # unknown code, ignore (or fire "unknown")

        # double-click
        if button == self._dc_fp and (now - self._dc_t) < 1.3:
            button, self._dc_fp = f"{button}_2x", ""
        else:
            self._dc_fp, self._dc_t = button, now

        async_dispatcher_send(self.hass, f"{DOMAIN}_{self.entry.entry_id}", button)
```

The event entity subscribes to that dispatcher signal and calls `_trigger_event`.

### `event.py`

```python
from homeassistant.components.event import EventEntity
from homeassistant.components.infrared import InfraredReceiverConsumerEntity

class IrRemoteEventEntity(InfraredReceiverConsumerEntity, EventEntity):
    _attr_has_entity_name = True
    _attr_translation_key = "buttons"

    def __init__(self, entry):
        self._infrared_receiver_entity_id = entry.data[CONF_RECEIVER]
        self._attr_unique_id = f"{entry.entry_id}_buttons"
        # learned button names + their _2x variants:
        self._attr_event_types = build_event_types(entry.options.get("codes", {}))

    @callback
    def _handle_signal(self, signal):
        # Optional: do decoding here instead of in the engine. If you keep the
        # engine, leave this empty and listen to the dispatcher signal instead.
        ...

    @callback
    def _async_emit(self, button: str) -> None:
        self._trigger_event(button)
        self.async_write_ha_state()
```

> `InfraredReceiverConsumerEntity` gives you availability tracking + auto
> (un)subscribe for free; if you centralize decoding in `IrRemoteEngine` you can
> instead inherit plain `EventEntity` and bridge via the dispatcher. Pick one — don't
> double‑subscribe to the same receiver.

---

## 4. Open questions / decisions to make in the new repo

1. **Trigger surface:** `event` entity (UI‑friendly, needs pre‑declared
   `event_types` + reload on learn) **vs.** bus event (zero ceremony, closest to your
   MQTT flow). Recommendation: ship the event entity, offer a bus event option.
2. **Learning UX:** config‑flow "learn" step vs. a `button` entity + `text` helper vs.
   a `ir_remote.learn` service that takes the button name. A service is the least
   fiddly for power users.
3. **Where decoding lives:** a single `IrRemoteEngine` per entry (clean, testable) vs.
   inside the `InfraredReceiverConsumerEntity._handle_signal`. The engine scales better
   if you later add multiple surfaces (event + sensor + last‑code attribute).
4. **Double/triple‑click & long‑press:** generalize your `_2x` logic into a small click
   state machine; expose windows as options.
5. **Storing learned codes:** config‑entry `options` is simplest; for many remotes
   consider a `helpers.storage.Store` keyed by receiver.
6. **ESPHome firmware:** confirm your build exposes the IR receiver via the native
   infrared API (advertises `InfraredInfo`/`RECEIVER`). This is the prerequisite that
   makes `infrared.livingroom_ir_receiver` appear — verify in the ESPHome project docs
   for your version. The custom integration is hardware‑agnostic and will also work
   with the `kitchen_sink` demo receiver for development.

---

## 5. Fast path to start developing without hardware

`homeassistant/components/kitchen_sink/infrared.py` registers a demo
`DemoInfraredReceiver`. Enable the `kitchen_sink` integration and you get an
`infrared.*` receiver entity you can drive in tests via the demo's dispatcher signal —
point your config flow at it and iterate on decode/debounce/event logic with no ESP at
all. The core tests in `tests/components/infrared/` (`common.py`, `test_init.py`) show
how to add mock emitter/receiver entities and how `_handle_received_signal` /
`async_subscribe_receiver` behave — good templates for your own test suite.
