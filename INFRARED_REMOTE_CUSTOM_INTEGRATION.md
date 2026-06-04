# Build spec / kickoff prompt — HACS custom integration: learnable IR remote → Home Assistant triggers

> **Use this file as the starting prompt for a fresh repo.** It is a complete build
> spec for a HACS custom integration that lets a user point at any Home Assistant
> **infrared receiver** entity, **learn** the codes from any physical IR remote through
> a point‑and‑click UI, and fire an **event** per button that automations can trigger
> on (single press, double‑click, etc.).
>
> All core APIs referenced below were verified against Home Assistant `2026.x`
> source (the new `infrared` entity platform shipped in `2026.6.0`) and the ESPHome
> component source. Where a real in‑core reference integration exists, it is named so
> the implementing agent can read it directly.

---

## 0. What to build (goal)

A HACS custom integration, working name **`ir_remote`**, that:

1. In its config flow, lets the user pick an existing **`infrared` receiver entity**
   (provided by ESPHome, Broadlink, or the `kitchen_sink` demo).
2. Lets the user **learn buttons** from *any* IR remote via a UI "Add button" flow:
   press the button → the integration captures the signal → the user names it.
3. Exposes one **`event` entity** per configured receiver whose `event_types` are the
   learned button names, so automations trigger on presses like any other event.
4. Handles **repeats/debounce** and **double‑click** (derived at runtime), porting the
   logic from the user's old ESPHome `on_raw` lambda into Python.
5. Works for **non‑standard remotes** by fingerprinting raw timings (the library only
   decodes NEC), and uses real protocol decoding when it applies.

This replaces an ESP8266/ESPHome `remote_receiver` + `on_raw` lambda + MQTT learning
rig with the native HA infrared pipeline. No MQTT, no `text_sensor`, no on‑device
decoding.

---

## 1. Background: the core `infrared` entity platform (2026.6.0)

`homeassistant/components/infrared/` is an **entity integration** (like `light` or
`event`) — internal plumbing, not user‑configured. It provides base classes for
hardware integrations that *provide* IR emitters/receivers, plus helpers for
integrations (like ours) that *consume* them.

### 1.1 Provider‑side classes (`infrared/entity.py`)

```python
class InfraredDeviceClass(StrEnum):
    EMITTER  = "emitter"
    RECEIVER = "receiver"

@dataclass(frozen=True, slots=True)
class InfraredReceivedSignal:
    timings: list[int]              # signed microseconds: +pulse(high), -space(low)
    modulation: int | None = None  # carrier frequency in Hz (e.g. 38000)

class InfraredEmitterEntity(RestoreEntity):
    async def async_send_command(self, command: InfraredCommand) -> None: ...

class InfraredReceiverEntity(RestoreEntity):
    def _handle_received_signal(self, signal: InfraredReceivedSignal) -> None: ...
    def async_subscribe_received_signal(self, cb) -> CALLBACK_TYPE: ...
```

**Critical:** a receiver's `state` is only an ISO timestamp of the last signal. The
payload (`InfraredReceivedSignal.timings`) is delivered **out‑of‑band to subscribers**.
You **cannot** drive automations off the receiver entity's state — you must subscribe.

`InfraredCommand` is `infrared_protocols.commands.Command`: `.modulation`,
`.repeat_count`, `.get_raw_timings() -> list[int]`.

### 1.2 Consumer‑side API (`infrared/helpers.py`, `infrared/__init__.py`) — **our API**

```python
from homeassistant.components.infrared import (
    InfraredReceivedSignal,
    InfraredReceiverConsumerEntity,   # base entity: availability + subscription mgmt
    async_get_receivers,              # -> list[str] of receiver entity_ids
    async_get_emitters,               # -> list[str] of emitter entity_ids
    async_subscribe_receiver,         # (hass, entity_id, cb) -> unsub CALLBACK_TYPE
    async_send_command,               # (hass, entity_id, command, context=None)
)
```

- `async_subscribe_receiver(hass, receiver_entity_id, cb)` — `cb(signal)` is called for
  every received signal; returns an unsubscribe callable. Raises `HomeAssistantError`
  if the component isn't loaded or the entity isn't a receiver.
- `InfraredReceiverConsumerEntity` — base entity that **auto‑subscribes when the
  receiver is available, unsubscribes when it goes unavailable, and cleans up on
  removal**. Set `self._infrared_receiver_entity_id`; implement
  `_handle_signal(self, signal)`. **This is the base class for our event entity.**

---

## 2. The canonical in‑core reference: `lg_infrared`

`homeassistant/components/lg_infrared/` (silver quality scale) is **exactly our shape**,
just hardcoded to LG TVs. **Read it first** — our integration is `lg_infrared` with the
fixed command map replaced by a *learned* map plus a raw‑timings fallback.

`lg_infrared/event.py` (abridged — the pattern to copy):

```python
class LgIrReceivedCommandEvent(LgIrEntity, InfraredReceiverConsumerEntity, EventEntity):
    _attr_event_types = _EVENT_TYPES        # fixed list of known LG commands

    def __init__(self, entry, receiver_entity_id):
        super().__init__(entry, unique_id_suffix="received_command")
        self._infrared_receiver_entity_id = receiver_entity_id

    @callback
    def _handle_signal(self, signal: InfraredReceivedSignal) -> None:
        nec_command = NECCommand.from_raw_timings(signal.timings)
        if nec_command is None or nec_command.address != LG_ADDRESS:
            return
        event_type = _COMMAND_CODE_TO_EVENT_TYPE.get(
            LGTVCode(nec_command.command), _EVENT_TYPE_UNKNOWN
        )
        self._trigger_event(event_type)
        self.async_write_ha_state()
```

`lg_infrared/config_flow.py` shows the **receiver‑picker** pattern: call
`async_get_receivers(self.hass)`, abort with `no_infrared_entities` if empty, present an
`EntitySelector(EntitySelectorConfig(domain="infrared"))`.

`lg_infrared/manifest.json` shows the required wiring:

```json
{ "domain": "lg_infrared", "config_flow": true,
  "dependencies": ["infrared"], "integration_type": "device" }
```

**Key differences our integration introduces vs. `lg_infrared`:**

| `lg_infrared` | `ir_remote` (this build) |
|---|---|
| Fixed `LGTVCode` map, `event_types` known at compile time | **Learned** map; `event_types` built from stored buttons, grows on learn |
| NEC + LG address only | NEC decode **first**, raw‑timings **fingerprint fallback** for any remote |
| No learning UI | **Subentry "Add button"** flow with live capture |
| No repeat/double‑click handling | Runtime debounce + single/double‑click engine |

---

## 3. ESPHome ESP32 firmware (the device half)

This integration is hardware‑agnostic, but the intended source is an **ESP32 + IR
receiver** flashed with ESPHome exposing the new native `infrared` platform.

### 3.1 Hardware

- **ESP32 dev board** (e.g. `esp32dev` / DevKitC). ESP32 is strongly preferred over
  ESP8266 for IR because it captures timings with the hardware **RMT peripheral** —
  far more accurate and reliable than the ESP8266's bit‑banging.
- **38 kHz IR receiver module** — TSOP38238, VS1838B, or similar.
  - `VCC` → `3V3`
  - `GND` → `GND`
  - `OUT` → a GPIO (example uses `GPIO14`)

### 3.2 ESPHome / Home Assistant prerequisites

- **ESPHome** recent enough to include the **experimental** `infrared` platform
  component and its `ir_rf_proxy` platform (codeowner `@kbx81`; pairs with HA `2026.6`).
  If `infrared:`/`ir_rf_proxy` aren't recognized, update ESPHome first. These are marked
  EXPERIMENTAL — the device API may still change.
- **Home Assistant** with the `esphome` integration; it maps the device's `InfraredInfo`
  (capability `RECEIVER`) to an `infrared.*` **receiver entity** automatically. Adopt the
  device in HA and that entity appears — that's what this integration consumes.

### 3.3 `ir_rf_proxy` schema (verified from ESPHome source)

`infrared:` is a platform component (`IS_PLATFORM_COMPONENT = True`). The `ir_rf_proxy`
platform extends the entity schema (so it takes `name`, `id`) and accepts:

- `remote_receiver_id` — `use_id` of a `remote_receiver` (for **receiving**)
- `remote_transmitter_id` — `use_id` of a `remote_transmitter` (for **sending**)
- **Exactly one** of those two per `ir_rf_proxy` entry (one entry = RX *or* TX; define
  two entries for both).
- `frequency` (default `0` = IR carrier; `>0` = RF), `receiver_frequency`
  (RX demodulation, optional).

### 3.4 Full ESPHome config — IR **receiver** (secrets via `!secret`)

> Put real values in ESPHome's `secrets.yaml`. **Never inline the API key, OTA password,
> or Wi‑Fi credentials in a config that lands in a public repo.** If a key has ever been
> pasted in plaintext anywhere, regenerate it.

```yaml
substitutions:
  name: ir-receiver
  friendly_name: "IR Receiver"

esphome:
  name: ${name}
  friendly_name: ${friendly_name}

esp32:
  board: esp32dev
  framework:
    type: esp-idf

logger:

# Native API — the local transport Home Assistant uses (this replaces MQTT entirely)
api:
  encryption:
    key: !secret api_encryption_key

ota:
  - platform: esphome
    password: !secret ota_password

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password
  # Fallback hotspot if Wi-Fi fails
  ap:
    ssid: "${friendly_name} Fallback"
    password: !secret fallback_password

captive_portal:

# 1) Raw IR capture. On ESP32 this uses the hardware RMT peripheral automatically.
remote_receiver:
  id: ir_rx
  pin:
    number: GPIO14
    inverted: true          # most 38 kHz IR modules idle HIGH / are active-LOW
    mode:
      input: true
      pullup: true
  idle: 25ms                # gap that marks "end of transmission"; tune per remote
  buffer_size: 2kb
  # tolerance: 25%          # widen if captures are flaky
  # filter: 50us            # ignore ultra-short noise spikes
  # dump: raw               # uncomment for ESPHome-log debugging only

# 2) Bridge that receiver to Home Assistant's native Infrared platform.
#    This makes the device advertise RECEIVER capability, so HA creates an
#    `infrared.ir_receiver_ir_receiver` receiver entity.
infrared:
  - platform: ir_rf_proxy
    name: "IR Receiver"
    remote_receiver_id: ir_rx

button:
  - platform: restart
    name: Restart
```

`secrets.yaml` keys this references: `api_encryption_key`, `ota_password`,
`wifi_ssid`, `wifi_password`, `fallback_password`.

> **To also transmit** later: add a `remote_transmitter:` and a *second* `ir_rf_proxy`
> entry with `remote_transmitter_id:` — that surfaces an `infrared` **emitter** entity
> HA can send through (encode with the `infrared_protocols` command classes / code DBs).

### 3.5 Migration note (old → new)

From the original ESP8266 learning rig, **delete**: the `mqtt:` block, `globals:`,
`text_sensor:`, and the entire `remote_receiver:` `on_raw:` / `on_jvc:` lambdas. All
decoding/fingerprinting/double‑click logic moves into this HA integration. The ESP only
forwards raw timings.

---

## 4. Integration architecture

### 4.1 `manifest.json`

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

- `"dependencies": ["infrared"]` ensures the `infrared` component and its helpers load
  first.
- Pin the same `infrared-protocols` version core uses. If you do **only** raw
  fingerprinting (no NEC decode), you can drop the requirement entirely.

### 4.2 Config flow — pick the receiver

```python
from homeassistant.components.infrared import async_get_receivers
from homeassistant.config_entries import ConfigFlow, ConfigFlowResult
from homeassistant.helpers.selector import EntitySelector, EntitySelectorConfig
import voluptuous as vol

class IrRemoteConfigFlow(ConfigFlow, domain=DOMAIN):
    VERSION = 1

    async def async_step_user(self, user_input=None) -> ConfigFlowResult:
        if not async_get_receivers(self.hass):
            return self.async_abort(reason="no_infrared_receivers")
        if user_input is not None:
            await self.async_set_unique_id(user_input[CONF_RECEIVER])
            self._abort_if_unique_id_configured()
            return self.async_create_entry(title=user_input[CONF_NAME], data=user_input)
        return self.async_show_form(
            step_id="user",
            data_schema=vol.Schema({
                vol.Required(CONF_NAME): str,
                vol.Required(CONF_RECEIVER): EntitySelector(
                    EntitySelectorConfig(domain="infrared")
                ),
            }),
        )

    @classmethod
    @callback
    def async_get_supported_subentry_types(cls, config_entry):
        # Enables the "Add button" UI on the integration's device page.
        return {"button": LearnButtonSubentryFlow}
```

### 4.3 Learning UI — `ConfigSubentryFlow` + `async_show_progress`

Each learned button is a **config subentry**, so users add/rename/delete buttons from
the integration's device page. The learn step uses `async_show_progress` to wait for a
live press, then a form to name it. (Real in‑core integrations using the
wait‑for‑hardware progress pattern: `improv_ble`, `motionmount`, `ekeybionyx`.)

```python
import asyncio
from homeassistant.config_entries import ConfigSubentryFlow, SubentryFlowResult
from homeassistant.core import callback
from homeassistant.components.infrared import (
    InfraredReceivedSignal, async_subscribe_receiver,
)

class LearnButtonSubentryFlow(ConfigSubentryFlow):
    """Learn one IR button and name it."""

    _capture_task: asyncio.Task[str] | None = None
    _fingerprint: str | None = None

    async def async_step_user(self, user_input=None) -> SubentryFlowResult:
        if self._capture_task is None:
            self._capture_task = self.hass.async_create_task(self._async_capture())

        if not self._capture_task.done():
            return self.async_show_progress(
                step_id="user",
                progress_action="press_button",      # -> strings.json
                progress_task=self._capture_task,
            )

        try:
            self._fingerprint = self._capture_task.result()
        except TimeoutError:
            self._capture_task = None
            return self.async_show_progress_done(next_step_id="timeout")

        self._capture_task = None
        return self.async_show_progress_done(next_step_id="name")

    async def async_step_name(self, user_input=None) -> SubentryFlowResult:
        if user_input is not None:
            # Optional: detect duplicate fingerprint here and branch to confirm overwrite.
            return self.async_create_entry(
                title=user_input["name"],
                data={"fingerprint": self._fingerprint, "name": user_input["name"]},
            )
        suggested = suggest_name(self._fingerprint)  # e.g. decoded "nec:04fb:08"
        return self.async_show_form(
            step_id="name",
            data_schema=vol.Schema(
                {vol.Required("name", default=suggested): str}
            ),
        )

    async def async_step_timeout(self, user_input=None) -> SubentryFlowResult:
        if user_input is not None:
            self._capture_task = None
            return await self.async_step_user()      # retry
        return self.async_show_form(step_id="timeout", data_schema=vol.Schema({}))

    async def _async_capture(self) -> str:
        """Subscribe to the receiver and return the first debounced fingerprint."""
        receiver_id = self._get_entry().data[CONF_RECEIVER]
        fut: asyncio.Future[str] = self.hass.loop.create_future()
        last_t = 0.0

        @callback
        def on_signal(signal: InfraredReceivedSignal) -> None:
            nonlocal last_t
            now = self.hass.loop.time()
            if now - last_t < 0.15:          # ignore repeat/bounce frames
                last_t = now
                return
            last_t = now
            if not fut.done():
                fut.set_result(fingerprint(signal.timings))

        unsub = async_subscribe_receiver(self.hass, receiver_id, on_signal)
        try:
            async with asyncio.timeout(20):
                return await fut
        finally:
            unsub()
```

`strings.json` (subentry progress + steps):

```json
{
  "config_subentries": {
    "button": {
      "initiate_flow": { "user": "Add button" },
      "step": {
        "name": {
          "title": "Name this button",
          "data": { "name": "Button name" }
        },
        "timeout": { "title": "No signal detected", "description": "Try again." }
      },
      "progress": {
        "press_button": "Press the button on your remote now…"
      }
    }
  }
}
```

> After a subentry is created, **reload the config entry** so the event entity is rebuilt
> with the expanded `event_types`. Event entities require their `event_types` known at
> construction time, so a newly learned button only becomes a valid trigger after reload.

### 4.4 Event entity (`event.py`) — the automation surface

```python
from homeassistant.components.event import EventEntity
from homeassistant.components.infrared import (
    InfraredReceivedSignal, InfraredReceiverConsumerEntity,
)

async def async_setup_entry(hass, entry, async_add_entities):
    async_add_entities([IrRemoteEventEntity(entry)])

class IrRemoteEventEntity(InfraredReceiverConsumerEntity, EventEntity):
    _attr_has_entity_name = True
    _attr_translation_key = "buttons"

    def __init__(self, entry):
        self._entry = entry
        self._infrared_receiver_entity_id = entry.data[CONF_RECEIVER]
        self._attr_unique_id = f"{entry.entry_id}_buttons"
        codes = {s.data["fingerprint"]: s.data["name"] for s in entry.subentries.values()}
        self._codes = codes
        # event_types = each learned name + its `_2x` double-click variant + "unknown"
        self._attr_event_types = build_event_types(codes.values())
        self._engine = ClickEngine(hass=entry.hass)  # see 4.5

    @callback
    def _handle_signal(self, signal: InfraredReceivedSignal) -> None:
        result = self._engine.process(signal, self._codes)   # None if filtered/unknown
        if result is None:
            return
        self._trigger_event(result.event_type, {"fingerprint": result.fingerprint})
        self.async_write_ha_state()
```

> Alternative surface, if you'd rather mirror the old MQTT flow exactly: skip the event
> entity and `hass.bus.async_fire("ir_remote_button", {...})`, triggering automations
> with the `event` **trigger platform**. No pre‑declared `event_types`, so no reload on
> learn — handy during heavy learning. Offer both; ship the event entity as primary.

### 4.5 Decode / debounce / double‑click engine

```python
from infrared_protocols.commands.nec import NECCommand

def fingerprint(timings: list[int]) -> str:
    # Prefer a real protocol decode (robust to jitter); fall back to raw quantization.
    if (cmd := NECCommand.from_raw_timings(timings)) is not None:
        return f"nec:{cmd.address:04x}:{cmd.command:02x}"
    bits = []
    for i in range(2, len(timings) - 1, 2):          # skip 2-value header + trailing pulse
        space = abs(timings[i + 1]) if i + 1 < len(timings) else 0
        bits.append("1" if space > 1000 else "0")     # same threshold as the old lambda
    return "raw:" + "".join(bits)
```

Click state machine (port of the ESPHome lambda; use `hass.loop.time()` monotonic time):

- gap `< 0.15 s` → repeat/bounce → ignore
- different fingerprint **or** gap `> 0.25 s` → **new press**
- same button within the double‑click window (default `1.3 s`) → emit `"<name>_2x"`
  and reset; otherwise emit the single press

All four windows (`0.15`, `0.25`, double‑click `1.3 s`, learn timeout `20 s`) should be
**options** on the config entry so they're tunable per remote — they were magic
constants in the original lambda.

---

## 5. Suggested file layout

```
custom_components/ir_remote/
├── __init__.py            # setup/unload entry, forward "event" platform, register subentry flow
├── manifest.json
├── const.py               # DOMAIN, CONF_* keys, default timing windows
├── config_flow.py         # receiver picker + LearnButtonSubentryFlow
├── engine.py              # fingerprint() + ClickEngine (debounce/double-click)
├── event.py               # IrRemoteEventEntity
├── strings.json
└── translations/en.json
```

(Optionally a `button.py` "Re-learn" / `sensor.py` "last code" later — not required.)

---

## 6. Testing

- **No hardware needed:** enable the `kitchen_sink` integration — its
  `DemoInfraredReceiver` registers an `infrared.*` receiver you can drive in tests via
  its dispatcher signal. Point the config flow at it and iterate.
- **Core test patterns to copy:** `tests/components/infrared/common.py` and
  `test_init.py` show how to add mock receiver entities and how
  `_handle_received_signal` / `async_subscribe_receiver` behave. `lg_infrared`'s tests
  show the event‑entity + receiver‑consumer test setup.
- Unit‑test `fingerprint()` and the `ClickEngine` directly with canned timing lists
  (incl. NEC frames and your remote's raw patterns) — no `hass` required.
- Follow HA test rules: typed params, `MockConfigEntry`, `pytest.mark.parametrize` with
  named `pytest.param` ids, snapshot (`.ambr`) for event payloads where useful.

---

## 7. Open decisions to make early

1. **Trigger surface:** event entity (recommended; UI‑native, needs reload on learn) vs.
   bus event (zero ceremony; mirrors old MQTT). Could support both.
2. **Single‑press latency:** emit single press immediately and let `_2x` be an *extra*
   event, vs. wait out the double‑click window before emitting single (cleaner but adds
   ~1.3 s latency). Make it an option.
3. **Where decoding lives:** centralized `ClickEngine` per entry (recommended, testable)
   vs. inline in `_handle_signal`.
4. **Storage of learned codes:** config **subentries** (recommended — gives the
   add/remove UI for free) vs. a `helpers.storage.Store`.
5. **Long‑press / triple‑click:** generalize the click state machine if wanted.

---

## 8. API quick reference (verified)

```python
# Consume an infrared receiver:
from homeassistant.components.infrared import (
    InfraredReceivedSignal,            # .timings: list[int] (signed µs), .modulation: int|None
    InfraredReceiverConsumerEntity,    # base entity: set _infrared_receiver_entity_id; impl _handle_signal
    async_get_receivers,               # (hass) -> list[str]
    async_subscribe_receiver,          # (hass, entity_id, cb) -> unsub; cb(signal)
)

# Decode (only NEC has a decoder; everything else encodes only):
from infrared_protocols.commands.nec import NECCommand
NECCommand.from_raw_timings(timings)   # -> NECCommand | None  (.address, .command, .repeat_count)

# UI flow primitives:
ConfigFlow.async_get_supported_subentry_types(cls, entry)  # -> {"button": LearnButtonSubentryFlow}
ConfigSubentryFlow._get_entry()                            # parent ConfigEntry
flow.async_show_progress(step_id=, progress_action=, progress_task=)
flow.async_show_progress_done(next_step_id=)

# Event entity:
from homeassistant.components.event import EventEntity   # _attr_event_types, self._trigger_event(type, data)
```

**Reference integration to mirror:** `homeassistant/components/lg_infrared/`
(silver quality scale) — same receiver‑consumer + event‑entity shape, minus the
learning UI and raw fingerprinting this build adds.
