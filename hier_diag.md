# Hierarchical Diagnostics and Triage

This note describes the original idea behind a hierarchical diagnostics model:
treat a complex system as a structured graph of diagnostic objects, each with
its own local state, commands, checks, and dependencies.

This idea is broader than any one implementation. It later influenced multiple
systems and projects, including higher-level operational environments and
lower-level system software.

## Motivation

Debugging is often blocked by lack of domain knowledge. In a large system, the
hard part is not only running commands. It is knowing:

1. what the major parts are
2. how they are wired together
3. what depends on what
4. which tools and checks to run
5. how to interpret results against expected state

Much of that knowledge can be embedded into the diagnostics structure itself.
That is the goal of this model.

## Core Idea

Represent a large system as a hierarchy of diagnostic nodes.

Each node represents one subsystem and can expose:

- structure
- local settings
- state
- commands
- checks
- dependencies

The user should not need to remember all of the raw hardware mapping, command
syntax, or subsystem-specific procedure. Those details are encapsulated in the
nodes. The user should see a clear navigable structure and be able to traverse
it, inspect state, run local commands, and follow dependency chains.

## What The Hierarchy Captures

The hierarchy is valuable because it can encode several kinds of knowledge at
once:

- structural knowledge: what the system contains
- categorization: common behavior and specialization
- dependencies and causality
- operational knowledge: commands, checks, expectations

That combination turns the hierarchy into both a diagnostic tool and a
knowledge model.

## Example Structure

As an example, consider a top-of-rack networking switch. A system-level node
can expose a structure like:

```text
system<modelA>
|--agent
|  |--process
|  |  |--run_env
|  |--swSwitch
|  |--hwSwitch
|  |--slow_path
|  |--data_path
|--platform<modelA>
|  |--mgnt_intf
|  |--kernel
|  |--bmc
|  |--pim[1..8]
|  |  |--fpga
|  |  |--qsfp[1..16]
|  |  |--xphy[1..4]
|  |  |  |--lane[0..3]
|--intf[eth[2..9]/[1..16]/1]
|--sdk
|--asic<TH3>
|  |--ports
```

The exact structure is not the point. The point is that the diagnostic system
already knows the object model of the target.

## Local Commands

Each node can expose commands appropriate to that subsystem.

For example:

```text
link<ce40>
  [cmd] xphy_map
  [cmd] status
  [cmd] iphy_to_xphy_prbs
  [cmd] xphy_to_iphy_prbs
iphy
  [cmd] dsc
lane
  [cmd] dsc
xphy_sys
  lane
    [cmd] lwdsc
    [cmd] summary
xphy_line
  lane
    [cmd] lwdsc
    [cmd] summary
qsfp
  [cmd] show
```

The key idea is that the user invokes commands through the hierarchy, not by
manually reconstructing the underlying device mapping every time.

## Checks and Dependencies

Nodes can also expose checks, and those checks can have dependencies.

Example:

```text
link<ce40>
  [check] ps_up
iphy
  lane
    [check] signal_detect
    [check] pll_lock
xphy_sys
  [check] config
  lane
    [check] peak
    [check] vga
    [check] SNR
xphy_line
  lane
    [check] signal_detect
    [check] pll_lock
qsfp
  [check] LOS
```

An interface-level check can then depend on lower-level conditions such as:

```text
..asic.port2 == OK
..platform.pim1.xphy2 == OK
..platform.pim1.qsfp10 == OK
```

Now the diagnostic path is explicit. A top-level check can recursively drive
the right lower-level checks without the operator manually translating between
layers.

## Encapsulation Of Domain Knowledge

Without this structure, many routine checks require too much hidden knowledge:

- hardware mapping
- device identifiers
- lane masks
- vendor command syntax
- model-specific behavior

For example, a raw command might look like:

```text
bcm_shell host001 --timeout 3000 -c 'xphy lw_dsc phy_id=0x48 if_side=1 lane_mask=0x100000'
```

The hierarchy can present that as something closer to:

```text
diag host001 tor:intf[eth4/9/1] check xphy.lane1
```

The framework becomes an object-oriented wrapper around the real domain tools.

## Why This Scales

This model scales well because it supports:

- extension: new checks or dependencies can be added as new failure modes are
  discovered
- aggregation: local subsystem knowledge can roll up into system-wide or
  fleet-wide triage
- separation of ownership: experts can refine their own subsystem nodes without
  one team owning the entire global diagnostic model
- reuse: a subsystem subtree can be shared, adapted, or exported independently

It also works as executable documentation. Instead of relying on scattered wiki
pages, the operational knowledge is embedded into the diagnostic model itself.

## Why This Matters

This approach is useful for:

- diagnostics and triage
- bring-up of large integrated systems
- hardware/software boundary debugging
- platform porting
- complex multi-team integration work

It is especially useful when no single person fully understands every subsystem
involved in the problem.

## Closing

The central idea is simple:

make the diagnostic structure reflect the real system structure, and attach the
operational knowledge directly to the relevant nodes.

That makes debugging less about remembering hidden tribal knowledge and more
about traversing an explicit, inspectable, and automatable model of the
system.

> Note:
> This post uses an open structural example based on networking-switch style
> systems. It is describing a general diagnostics principle, not one
> company-specific implementation.
