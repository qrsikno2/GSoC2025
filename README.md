# GSoC 2025

## 1. Project Overview

The primary goal of this project was to implement a robust and user-friendly framework within Lunatik, enabling developers to write Human Interface Device (HID) drivers for the Linux kernel entirely in Lua. The aim was to abstract the complexities of the C-based HID subsystem, providing a high-level, scriptable interface for tasks like device matching, initialization, event handling, and descriptor manipulation. This would empower developers to prototype and create custom HID drivers with greater speed and flexibility.



[This](./acceptance_letter_2025_Jieming-Zhou.pdf) is my acceptance letter and [This](./proposal-gsoc2025-ketsuminchow-ver6.pdf) is my proposal.

## 2. Work Accomplished (Deliverables)

Throughout the GSoC period, I implemented the core foundation of the LuaHID framework. The work is spread across several pull requests, demonstrating an incremental development approach.

### Merged Pull Requests

These PRs form the stable foundation of the `luahid` module.

- **PR #1: Initial Framework for `luahid`** [construct a skeleton for luahid](https://github.com/luainkernel/lunatik/pull/270)
- **PR #2: Support for Device Matching (name and id_tables)** [add dynamic input support for id_table](https://github.com/luainkernel/lunatik/pull/281)
- **PR #3: Support for `probe` Callback** [add handler for probe](https://github.com/luainkernel/lunatik/pull/297)
- **PR #4: Support for `report_fixup` Callback** [add callback for report_descriptor]()

### Work In-Progress (Unmerged Pull Requests)

These PRs contain significant features that are complete or near-complete and demonstrate the framework's capabilities.

- **PR #5: `raw_event` Callback Implementation** [add callback for raw_event](https://github.com/luainkernel/lunatik/pull/312)
- **PR #6: Examples and Driver Porting** [Hid examples](https://github.com/luainkernel/lunatik/pull/316)

## 3. Implementation

### Examples

```lua
-- Porting Xiaomi Silent Mouse's Kernel driver to work with luahid.
-- Link: https://elixir.bootlin.com/linux/v6.16.3/source/drivers/hid/hid-xiaomi.c

local hid = require("hid")

local driver = {
	name = "luahid_xiaomi",
	id_table = {
		{ bus = 0x05, vendor = 0x2717, product = 0x5014 }
	}
}

local mi_silent_mouse_orig_rdesc_length = 87
local mi_silent_mouse_rdesc_fixed = {
	0x05, 0x01, --  Usage Page (Desktop),
	0x09, 0x02, --  Usage (Mouse),
	0xA1, 0x01, --  Collection (Application),
	0x85, 0x03, --      Report ID (3),
	0x09, 0x01, --      Usage (Pointer),
	0xA1, 0x00, --      Collection (Physical),
	0x05, 0x09, --          Usage Page (Button),
	0x19, 0x01, --          Usage Minimum (01h),
	0x29, 0x05, -- X  --    Usage Maximum (05h),
	0x15, 0x00, --          Logical Minimum (0),
	0x25, 0x01, --          Logical Maximum (1),
	0x75, 0x01, --          Report Size (1),
	0x95, 0x05, --          Report Count (5),
	0x81, 0x02, --          Input (Variable),
	0x75, 0x03, --          Report Size (3),
	0x95, 0x01, --          Report Count (1),
	0x81, 0x01, --          Input (Constant),
	0x05, 0x01, --          Usage Page (Desktop),
	0x09, 0x30, --          Usage (X),
	0x09, 0x31, --          Usage (Y),
	0x15, 0x81, --          Logical Minimum (-127),
	0x25, 0x7F, --          Logical Maximum (127),
	0x75, 0x08, --          Report Size (8),
	0x95, 0x02, --          Report Count (2),
	0x81, 0x06, --          Input (Variable, Relative),
	0x09, 0x38, --          Usage (Wheel),
	0x15, 0x81, --          Logical Minimum (-127),
	0x25, 0x7F, --          Logical Maximum (127),
	0x75, 0x08, --          Report Size (8),
	0x95, 0x01, --          Report Count (1),
	0x81, 0x06, --          Input (Variable, Relative),
	0xC0,      --      End Collection,
	0xC0,      --  End Collection,
	0x06, 0x01, 0xFF, --  Usage Page (FF01h),
	0x09, 0x01, --  Usage (01h),
	0xA1, 0x01, --  Collection (Application),
	0x85, 0x05, --      Report ID (5),
	0x09, 0x05, --      Usage (05h),
	0x15, 0x00, --      Logical Minimum (0),
	0x26, 0xFF, 0x00, --      Logical Maximum (255),
	0x75, 0x08, --      Report Size (8),
	0x95, 0x04, --      Report Count (4),
	0xB1, 0x02, --      Feature (Variable),
	0xC0       --  End Collection
}

function driver:report_fixup(hdev, priv_data, original_report)
	if hdev.product == 0x5014 and #original_report == mi_silent_mouse_orig_rdesc_length then
		print("Fixing Xiaomi Silent Mouse report descriptor")
		for i = 1, #mi_silent_mouse_rdesc_fixed do
			original_report:setbyte(i - 1, mi_silent_mouse_rdesc_fixed[i])
		end
	end
end

hid.register(driver)
```

And There is a driver for mouse of QEMU. It would detect your dragging gesture and try to lock/unlock the mouse.

```lua
local hid = require("hid")

local driver = {
	name = "gesture",
	id_table = {
		{ vendor = 0x046D, product = 0xC542 }
	}
}

function driver:probe(devid)
	return { x = 0, y = 0, drag = false, lock = false }
end

function driver:raw_event(hdev, state, report, raw_data)
	local btn = raw_data:getbyte(0)
	local dx_byte = raw_data:getbyte(1)
	local dy_byte = raw_data:getbyte(2)
	-- complement conversion
	local dx = dx_byte >= 128 and dx_byte - 256 or dx_byte
	local dy = dy_byte >= 128 and dy_byte - 256 or dy_byte

	local left_down = (btn & 1) == 1
	if left_down then
		state.x = state.x + dx
		state.y = state.y + dy
		state.drag = true
	else
		if state.drag then
			if math.abs(state.x) > 100 or math.abs(state.y) > 100 then
				local direction = ""
				if math.abs(state.x) > math.abs(state.y) then
					direction = state.x > 0 and "rightly" or "leftly"
				else
					direction = state.y > 0 and "downly" or "uply"
				end
				print(string.format("Swipe %s with x=%d, y=%d", direction, state.x, state.y))
				if direction == "uply" then
					state.lock = true
					print("Locking mouse")
				end
				if direction == "downly" and state.lock then
					state.lock = false
					print("Unlocking mouse")
				end
			end
			state.x = 0
			state.y = 0
			state.drag = false
		end
	end
	if state.lock then
		raw_data:setbyte(0, 0) -- buttons
		raw_data:setbyte(1, 0) -- dx
		raw_data:setbyte(2, 0) -- dy
	end
	return false
end

hid.register(driver)
```

### Data Structures

The C function receives a Lua table and must translate it into the kernel-native `struct hid_driver` and its associated data structures.

The translation of `id_table` is implemented in the function.

```c
static const struct hid_device_id *luahid_setidtable(lua_State *L, int idx)
{
    size_t len = luaL_len(L, idx);
    struct hid_device_id *user_table = lunatik_checkalloc(L, sizeof(struct hid_device_id) * (len + 1));

    struct hid_device_id *cur_id = user_table;
    for (size_t i = 0; i < len; i++, cur_id++) {
        if (lua_geti(L, idx, i + 1) != LUA_TTABLE) { /* ... error handling ... */ }

        lunatik_optinteger(L, -1, cur_id, bus, HID_BUS_ANY);
        lunatik_optinteger(L, -1, cur_id, group, HID_GROUP_ANY);
        // ... and so on for vendor, product ...
        lua_pop(L, 1); /* table entry */
    }
    memset(cur_id, 0, sizeof(struct hid_device_id)); // Null-terminate the array
    return user_table;
}
```

To store the temporary passing data and user's  private data of each unique device,. i use `lua_table` to store it where hid is "main table" and ops and info is either the lua driver or the private data for device, indexed with their address of `hdev`. That would make sure there is only one key for each device.

```c
	lua_newtable(L); /* hid = {} */
	lua_pushvalue(L, 1);
	lua_setfield(L, 2, "ops"); /* hid.ops = table */
	lua_newtable(L);
	lua_setfield(L, 2, "_info"); /* hid._info = {} */
```

There are the rest of my data structures, and i would just give a list here.

```c
/***
* Represents a registered HID driver.
* This is a userdata object returned by `hid.register()`. It encapsulates
* the kernel `struct hid_driver` and associated Lunatik runtime information
* necessary to invoke the Lua callback when a HID device is matched.
* The `registered` field indicates whether the driver is currently registered
* @type hid_driver
*/
typedef struct luahid_s;

typedef struct {
	const struct hid_device_id *id;
} luahid_probe_arg_t;

typedef struct {
	__u8 *rdesc;
	unsigned int rsize;
} luahid_report_fixup_arg_t;

typedef struct {
	struct hid_report *report;
	u8 *data;
	int size;
} luahid_raw_event_arg_t;

```

### Callback Implementation Deep Dive: `raw_event`

To illustrate how the framework handles callbacks, we can trace the execution flow of `raw_event`, which is triggered every time the device sends a data report. 

1.  **Kernel Invocation**: The HID subsystem calls our registered C function with the device data.

    ```c
    static int luahid_raw_event(struct hid_device *hdev, struct hid_report *report, u8 *data, int size)
    ```

2.  **Context Retrieval**: The first step inside the C callback is to retrieve our `luahid_t` context from the `hdev` pointer. This gives us access to the Lunatik runtime associated with this specific driver instance.

    ```c
    luahid_t *hid = luahid_gethid(hdev); // uses container_of macro
    ```

3.  **Protected Execution via `lunatik_runirq`**: Because `raw_event` can execute in an interrupt context, we cannot directly call into the Lua VM. We use `lunatik_runirq`, a Lunatik API designed for this purpose. It schedules the execution of a helper function (`luahid_runraw_event`) within the safe context of the appropriate Lua runtime.

    ```c
    lunatik_runirq(hid->runtime, luahid_runraw_event, ret, hdev, &arg, &ret_bool);
    ```

4.  **Protected Lua Call with `pcall`**: Inside `luahid_runraw_event`, we don't call the final Lua logic function directly. Instead, we use a `luahid_pcall` macro which wraps the real logic in a `lua_pcall`. This is the core safety mechanism: any error within the user's Lua script will be caught here as a return code, rather than propagating up and causing a kernel panic.

    ```c
    luahid_pcall(L, luahid_doraw_event, hdev, arg);
    ```
    
5.  **Core Logic and Argument Preparation (`luahid_doraw_event`)**: This function, defined via the `LUAHID_CALLBACK` macro, contains the final logic before jumping to Lua.
    
    -   It finds the driver's Lua table (`ops`) and retrieves the user-defined `raw_event` Lua function from it.
    -   It calls a `luahid_preraw_event` helper function to prepare the arguments. This function converts the C pointers (`hdev`, `report`, `data`) into more user-friendly Lua tables and a `luadata` object for direct memory access.
    -   It executes the user's Lua script via the protected `lua_pcall`.
    -   It calls `luahid_postraw_event` to process the return value, converting the Lua boolean result into an integer that the kernel HID subsystem expects.
    
    ```c
    if (lua_getfield(L, -2, "raw_event") != LUA_TFUNCTION) { /* ... handle missing function ... */ }
    luahid_preraw_event(L, hdev, arg);
    if (lua_pcall(L, NARG, NRET, 0) != LUA_OK) { /* ... handle Lua error ... */ }
    luahid_postraw_event(L, hdev, arg);
    ```

This entire chain ensures that execution transitions from the kernel's interrupt context to a safe, managed Lua environment, with multiple layers of protection against script errors. This robust design is the foundation of the `luahid` module.

I will not go into details about the rest, the list of details are as follows.

### For Callbacks

```c
static void luahid_preprobe(lua_State *L, struct hid_device *hdev, luahid_probe_arg_t *arg);
static void luahid_prereport_fixup(lua_State *L, struct hid_device *hdev, luahid_report_fixup_arg_t *arg);
static void luahid_preraw_event(lua_State *L, struct hid_device *hdev, luahid_raw_event_arg_t *arg);
static void luahid_postprobe(lua_State *L, struct hid_device *hdev, luahid_probe_arg_t *arg);
static void luahid_postreport_fixup(lua_State *L, struct hid_device *hdev, luahid_report_fixup_arg_t *arg);
static void luahid_postraw_event(lua_State *L, struct hid_device *hdev, luahid_raw_event_arg_t *arg);

/* for defining callbacks */
#define LUAHID_CALLBACK(NAME, NARG, NRET, LEVEL)

LUAHID_CALLBACK(probe, 2, 1, err);
LUAHID_CALLBACK(report_fixup, 4, 0, warn);
LUAHID_CALLBACK(raw_event, 5, 1, err);

static int luahid_runprobe(lua_State *L, struct hid_device *hdev, luahid_probe_arg_t *arg);
static int luahid_probe(struct hid_device *hdev, const struct hid_device_id *id);
static int luahid_runreport_fixup(lua_State *L, struct hid_device *hdev, luahid_report_fixup_arg_t *arg);
static luahid_ret_t luahid_report_fixup(struct hid_device *hdev, __u8 *rdesc, unsigned int *rsize);
static int luahid_runraw_event(lua_State *L, struct hid_device *hdev, luahid_raw_event_arg_t *arg, int *ret_bool);
static int luahid_raw_event(struct hid_device *hdev, struct hid_report *report, u8 *data, int size);

```

### Auxiliary Functions and Marcos

```c
#define LUAHID_ARG(L, name)
#define luahid_setfield(L, idx, obj, field)	
#define luahid_pcall(L, func, hdev, arg)
#define luahid_newtable(L, dev, extra)
#define luahid_checkdriver(L, hid, idx, field) 
static inline void luahid_pushhdev(lua_State *L, struct hid_device *hdev);
static inline void luahid_pushinfo(lua_State *L, int idx, struct hid_device *hdev);
static inline void luahid_pushreport(lua_State *L, struct hid_report *report);
static luahid_t *luahid_gethid(struct hid_device *hdev);
static inline lunatik_object_t *luahid_getdescriptor(lua_State *L, luahid_t *hid);
```

## 4. Challenges and Learnings: A Journey Beyond Code

This GSoC was my first real dive into open source, and it was a journey that taught me as much about people and process as it did about code.

### Technical Hurdles & Key Takeaways

Working with Lua inside the Linux kernel was full of unique challenges that pushed my technical skills.
* **Memory Safety:** My biggest hurdle was bridging Lua's garbage-collected world with the kernel's strict, manual memory management. I learned this lesson the hard way: in callbacks like `report_fixup`, passing a kernel pointer (`rdesc`) to Lua is dangerous. If Lua holds onto that reference while the kernel reclaims the memory, you get a race condition and an instant kernel panic. My mentor's guidance was critical here, teaching me to design a defensive API where the lifecycle of such pointers is carefully controlled.
* **Bulletproof Error Handling:** I quickly discovered that any unhandled error from a Lua C API call originating in a kernel callback would crash the entire system. My mentor’s constant emphasis on this forced me to dive deep into the Lua C API. I made it a habit to wrap all script executions in a protected environment using `pcall`, which was a game-changer for stability. This taught me that in kernel development, you can't just code for the happy path.

### A Personal Reflection on Open Source

Beyond the code, my real growth came from learning how to be a part of a community. I started GSoC thinking this was a solo coding task, a mindset that led me to be quiet during the crucial community bonding period. That was a mistake, and it put me on the back foot. For a while, I was afraid to ask questions, worried I'd sound "stupid". My mentor's most valuable advice had nothing to do with code; it was that there are "no stupid questions" and that I should communicate openly. Learning to be transparent about what I didn't know and to break down my work into small, reviewable "baby steps" was transformative. It shifted my view from trying to deliver a perfect, finished product to collaborating on something, piece by piece. Working with a mentor from Brazil also taught me a lot about patience and clear communication across cultures. This journey was challenging, but it truly changed me as a developer.


---

## 5. Future Work

While the core framework is in place, there is more work to be done to achieve the full vision of the original proposal. My immediate plans are to:

-   Finalize and merge the `raw_event` implementation.
-   Clean up and merge the examples PR, providing clear documentation for new users.
-   Implement the full `on_event` table to handle specific input types (`key`, `abs`, `rel`).

## 6. Acknowledgements

I want to extend my deepest gratitude to my mentor, **Lourival Vieira Neto**. His technical guidance was invaluable, but his mentorship on open-source culture, communication, and process was truly transformative. He was patient through my mistakes, firm when I needed direction, and always encouraging. His constant reminders to "read the code," "take baby steps," and communicate openly have made me a better developer.

I would also like to thank the **LabLua Foundation** for accepting me into their community and giving me this incredible opportunity to work on a project that intersects with my passions for Lua and the Linux kernel. This GSoC journey has been challenging, but it has taught me lessons that will last a lifetime.
