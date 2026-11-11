<!--
Sync Impact Report
==================
Version change: 1.0.0 → 2.0.0
Bump rationale: MAJOR. By decision of the maintainer (2026-11-11) the project moves from
hand-written Arduino/PlatformIO firmware with MQTT to ESPHome, the same approach as the sibling
project Buitenlamp. Principle I (deployment target) and principle IV (automated host-side tests)
are redefined in a backward-incompatible way, and the build tool, project layout, Home Assistant
integration and quality gates change accordingly.

Modified principles (old title → new title):
- I. Arduino Is the Deployment Target → I. ESPHome Is the Deployment Target (REDEFINED: ESPHome
  with the esp-idf framework on the Seeed XIAO ESP32C3, pinned version, native API, OTA)
- II. Predictable Embedded Behavior → II. Non-Blocking, Bounded Automations (restated in ESPHome
  terms; adds millis() rollover rule, no own ISRs, no heap allocation in repeated lambdas)
- III. Safe Hardware Interaction → III. Safe, Monitoring-Only Hardware Interaction (pump
  control rule replaced by "never controls the pump"; invalid/stale data, alarm hold, ESPHome
  input filters, watchdog stays enabled)
- IV. Testable Control Logic → IV. Declarative, Traceable Logic (REDEFINED: logic in named
  scripts/automations with requirement ids; host-side unit tests no longer required; every
  decision rule covered by an on-device scenario in quickstart.md)
- V. Simple and Observable Firmware → V. Explicit, Simple and Observable Configuration
  (substitutions block, secrets.yaml, pinned versions, YAGNI, faults visible in Home Assistant
  and the log)

Modified sections:
- Platform Constraints → Embedded Platform Constraints: rewritten for ESPHome (language,
  board/framework, build with the ESPHome CLI, project layout, encrypted native API instead of
  MQTT, OTA, reboot timeouts, logging, mains-measurement safety)
- Development Workflow → Development Workflow & Quality Gates: five ordered gates (compile,
  resource, traceability, hardware, review); recording unverified hardware behavior is kept
- Governance: amendment procedure via /speckit-constitution, versioning, compliance review

Added sections: none (sections renamed and rewritten, see above)
Removed sections: none

Resolved TODO: RATIFICATION_DATE = 2026-11-03, the day version 1.0.0 was adopted from the
template (per the previous Sync Impact Report and its Last Amended date).

Templates: plan-template.md, spec-template.md, tasks-template.md and the speckit commands read
the constitution at runtime ✅ no edits needed.

Follow-up (dependent artifacts, not changed by this amendment):
- specs/001-pompput-monitor: spec.md is technology-neutral and still valid; plan.md,
  research.md (R1–R4, R11–R13), data-model.md, contracts/mqtt-homeassistant.md, quickstart.md,
  tasks.md and verificatie.md assume v1.0.0 (Arduino + MQTT + unit tests) and MUST be
  re-planned with /speckit-plan and /speckit-tasks.
- The Arduino/PlatformIO implementation (platformio.ini, include/, lib/, src/, test/) no longer
  matches principle I and is to be replaced in the re-planned tasks.
-->

# Buitenpomp Constitution

## Core Principles

### I. ESPHome Is the Deployment Target

- The firmware MUST be an ESPHome configuration for the Seeed Studio XIAO ESP32C3, built with
  the `esp-idf` framework. The ESPHome version MUST be pinned exactly (for example in a
  committed `requirements.txt`) and recorded in the plan.
- Home Assistant MUST connect through ESPHome's native API with encryption enabled. Firmware
  updates after the first USB flash go through ESPHome OTA, protected by a secret. MQTT and
  cloud services are not used.
- Platform-specific behavior that affects the device (pin assignments, electrical assumptions,
  ADC ranges, resource limits) MUST be documented where it affects behavior.
- External components MUST be justified in the plan and pinned to an exact version or commit.

Rationale: one toolchain and one way of integrating with Home Assistant for all devices in the
home (as in the Buitenlamp project), with encrypted local communication and updates over WiFi
for a device that is mounted in a pump pit.

### II. Non-Blocking, Bounded Automations

- Lambdas MUST NOT block: no `delay()`, no busy-wait, and no single lambda may run longer than
  10 ms. Waiting MUST use ESPHome's non-blocking actions (`delay`, `wait_until`, scripts,
  `interval`) or component update intervals.
- Time arithmetic in lambdas MUST use `millis()` with unsigned subtraction
  (`now - last >= interval`), so it survives the ~49.7-day rollover.
- Behavior with more than two states (level status, pump status, alert lifecycle) MUST be an
  explicit state machine: named states held in `globals` (or a `select`), with the
  transitions kept together in named scripts or automations.
- Project code MUST NOT install its own interrupt service routines; interrupts are used only
  through ESPHome components.
- Lambdas that run repeatedly (per loop, per update, per sensor value) MUST NOT allocate heap
  memory: no `new`/`malloc`, and no building or growing `std::string` in those paths.
- After each build, static RAM usage MUST stay at or below 75% of the target's SRAM and flash
  usage at or below 90%, as reported by the build.

Rationale: a blocked loop misses sensor readings, starves the watchdog and the network stack,
and delays alarms; an unattended device fails silently when memory fragments.

### III. Safe, Monitoring-Only Hardware Interaction

- The device only monitors. It MUST NOT have any output that can switch, interrupt or
  otherwise influence the pump, and the pump MUST keep working on its own float switch when
  the monitor fails or is unpowered.
- Every output (ultrasonic trigger, buzzer, status LED) MUST be in a defined, safe state from
  the earliest point at boot (`restore_mode: ALWAYS_OFF` or the component's equivalent).
- Sensor values MUST be interpreted with explicit units and validity limits before they
  influence a decision. Invalid, implausible or stale data MUST lead to an "unknown" state and
  a fault indication, never to a silent assumption. An active high-water alarm MUST be held
  while the level measurement is unavailable.
- Sensor inputs that decide alerts or indications MUST be filtered (ESPHome filters such as
  `median`, `delayed_on`, `delayed_off`, or explicit hysteresis), so a single outlier or a
  value near a threshold never causes an alert or makes it chatter.
- ESPHome's task watchdog MUST stay enabled; no option or lambda may disable it.
- Faults that inhibit monitoring MUST be diagnosable in Home Assistant and the log without
  compromising time-sensitive behavior.

Rationale: the monitor protects against flooding; a wrong reading must never hide or fake an
alarm, and a monitor failure must never stop the pump.

### IV. Declarative, Traceable Logic

- All decision logic (level validity, pump detection, alerts, local indication, settings
  validation) MUST live in named scripts, automations and template entities in the ESPHome
  YAML. Lambdas MUST stay short (one decision or one calculation) and MUST NOT duplicate logic
  that exists elsewhere in the configuration.
- Every script, automation or template entity that implements a requirement MUST carry a
  comment naming the requirement id(s) it implements (for example `# FR-012`).
- Every decision rule and every failure case MUST be covered by at least one manual on-device
  validation scenario in the feature's `quickstart.md`. Host-side unit tests are not required.
- Changes that depend on physical devices MUST be verified on the hardware; behavior that
  could not be verified MUST be recorded, with the reason, for reviewers.

Rationale: logic expressed declaratively and labelled with its requirement can be checked by
reading; on-device scenarios then prove the behavior on the real hardware.

### V. Explicit, Simple and Observable Configuration

- All pin assignments, thresholds, timeouts and other compile-time tunables MUST be named
  entries in one `substitutions:` block at the top of the device YAML; magic numbers in
  automations, scripts and lambdas are forbidden. Settings the user changes in Home Assistant
  are `number`/`switch` entities with their limits taken from that block.
- Secrets (WiFi credentials, API encryption key, OTA secret) MUST live only in
  `secrets.yaml`, which is git-ignored; a committed `secrets.example.yaml` documents the
  required keys.
- The configuration MUST implement only what the current spec requires (YAGNI). Optional
  ESPHome features (for example `web_server` or `captive_portal`) are added only when the spec
  needs them; unused components MUST be removed before delivery.
- Important state transitions and every fault MUST be observable through Home Assistant
  entities or events and through the log. Diagnostics MUST NOT block time-sensitive behavior.

Rationale: the configuration is read and adapted by people wiring real hardware; one place for
every tunable makes it safe to change, and visible faults make an unattended device trustworthy.

## Embedded Platform Constraints

- **Firmware**: an ESPHome YAML configuration plus short C++ lambdas, as supported by the pinned
  ESPHome version. Lambdas use no exceptions and no RTTI.
- **Target board**: Seeed Studio XIAO ESP32C3, framework `esp-idf`. The GPIOs are 3.3 V and not
  5 V-tolerant; level shifting for sensor signals uses resistor dividers. Only ADC1 pins are
  used for analog measurements.
- **Pin source**: `specs/<feature>/contracts/hardware.md` is the authoritative pin map
  (pin → component → direction); the `substitutions:` block of the device YAML MUST match it.
- **Build**: firmware is validated, built and flashed with the ESPHome CLI from a project-local
  virtual environment with the pinned version. The first flash goes over USB; later updates go
  over the air.
- **Project layout**: one device YAML per device, `secrets.yaml` (git-ignored) with a committed
  `secrets.example.yaml`, the Home Assistant configuration that belongs to the feature (for
  example a package with alerts), and the specs. ESPHome build output (`.esphome/`) is
  git-ignored. There is no host-side test folder.
- **Home Assistant**: integration uses ESPHome's native API with encryption enabled. All
  communication stays inside the home network.
- **Independent operation**: measurement, alert evaluation and local indication (LED and
  buzzer) MUST keep working without WiFi, Home Assistant or a serial monitor. ESPHome's
  automatic reboot timeouts (`wifi: reboot_timeout`, `api: reboot_timeout`) MUST be disabled
  unless the plan justifies otherwise, so losing the network never resets an active alarm.
- **Mains**: any part on mains voltage (the ACS712 current sensor) is measurement only, MUST be
  documented with its safety requirements in the hardware contract, and is connected by or
  under supervision of a qualified person.
- **Logging**: log verbosity MUST be set by one `logger` level setting. Delivered firmware runs
  at `INFO` or quieter; `DEBUG` is for validation only.

## Development Workflow & Quality Gates

Every delivered change MUST pass these gates in order:

1. **Compile gate**: `esphome config` reports no errors, and `esphome compile` succeeds for the
   named board with zero errors and zero warnings originating from project lambdas.
2. **Resource gate**: the build's memory report is within the limits of Principle II and is
   recorded in the feature's documentation.
3. **Traceability gate**: every functional requirement maps to a named script, automation or
   entity in the YAML and to at least one scenario in the feature's `quickstart.md`; the
   mapping is recorded in the feature's documentation.
4. **Hardware gate**: the feature's `quickstart.md` contains a manual on-device test procedure
   (wiring steps, expected behavior per scenario, including power-cycle recovery and loss of
   network or Home Assistant); every scenario passes and the pin map matches the YAML. Any
   scenario that could not be run on hardware is recorded with the reason.
5. **Review gate**: the configuration is checked against every principle above; any deviation
   is recorded in the plan's Complexity Tracking table with a justification.

The configuration MUST be readable by someone who did not write it: hardware-specific settings
(timing values, electrical modes, calibration factors) and every non-obvious automation MUST
carry a short comment explaining why.

## Governance

- This constitution supersedes all other practices in this repository. Where a spec, plan,
  task or device configuration conflicts with it, the constitution wins until it is amended.
- **Amendments** MUST be made through `/speckit-constitution`, MUST state the reason for the
  change, MUST be approved by the project maintainer, and MUST include a Sync Impact Report
  listing affected principles and sections.
- **Versioning** follows semantic versioning:
  - MAJOR: a principle or gate is removed or redefined in a backward-incompatible way.
  - MINOR: a principle or section is added or its guidance is materially expanded.
  - PATCH: clarifications, wording or typo fixes with no change in meaning.
- **Compliance review**: every `/speckit-plan` MUST complete the Constitution Check before
  design and again after it; `/speckit-analyze` MUST flag violations as CRITICAL; deviations
  are allowed only when justified in the plan's Complexity Tracking table.

**Version**: 2.0.0 | **Ratified**: 2026-11-03 | **Last Amended**: 2026-11-11
