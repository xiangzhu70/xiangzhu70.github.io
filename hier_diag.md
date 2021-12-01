# A Hierarchical OO Diag Framework

## Motivation
Debugging challenges often come from lack of domain knowledge.  It is hard to grasp all possible knowledge to be prepared for a new bug.

To debug a problem in any domain, the following types of domain knowledge are need.

1.  Structural information.  What are the major parts in this module, how each parts are wired.  Especially when debugging hardware problems, the circuit mapping, bus connections, register offsets are required.
2.  Categorization information.  Commonalities, and differences (In OO terms, inheritance and overrides).
3.  Interactions and dependencies, causalities.
4.  Tools, states, expectations.  How to run the tools, interpret output, check states, compare with expectations and spot discrepancies.
5.  design, inner-working mechanism, trade-off, limitations.

Among these types, much of #1-4 can be built into the tools and be automated.  This is the aim of this framework.  It models a large and complex system as a hierarchical OO structure of nodes.  Each node represents one sub system, with local commands and settings, and states, like a typical OO object.  The type #1-4 information is embedded into the nodes and implicitly used to derive node type, local settings, and support the commands.  They are hidden away from the users.  What the users see is  a clear browser-like user interface.  They can easily traverse the nodes, run the node local commands and checks, and get meaningful results.

## Structure

Let's take a networking switch  as an example to see how this framework helps.

diag TOR -n host=\<hostname> -x  show -t
The tool enters the node "system".  The node discovers the target device is a modelA model, so it morphs into a "system\<modelA>" node, and shows a modelA structure.
The tool shows the structure of a top-of-rack (TOR) networking switch device:
```
|--system\<modelA>
|--|--agent
|--|--|--process
|--|--|--|--run_env
|--|--|--swSwitch
|--|--|--hwSwitch
|--|--|--slow_path
|--|--|--data_path
|--|--platform\<model>
|--|--|--mgnt_intf
|--|--|--kernel
|--|--|--bmc
|--|--|--pim[1..8]
|--|--|--|--fpga
|--|--|--|--qsfp[1..16]
|--|--|--|--xphy[1..4]
|--|--|--|--|--lane[0..3]
|--|--intf[eth[2..9]/[1..16]/1]
|--|--sdk
|--|--asic\<TH3>
|--|--|--ports
```
# Commands
At each node, there can be commands attached.
```
diag tor -n host=\<hostname> -x eth4/9/1.link show cmds -t
|--link\<ce40>
[cmd] xphy_map
[cmd] status
[cmd] iphy_to_xphy_prbs
[cmd] xphy_to_iphy_prbs
|--iphy
[cmd] dsc
|--[0..1] lane
|--lane
[cmd] dsc
|--xphy_sys
|--[0..1] lane
|--lane
[cmd] lwdsc
[cmd] summary
|--xphy_line
|--[0..3] lane
|--lane
[cmd] lwdsc
[cmd] summary
|--qsfp
[cmd] show
```
## Checks and dependencies
There can also be checks attached.
diag tor -n host=\<hostname> -x eth4/9/1.link show checks -t
```
|--link\<ce40>
[check] ps_up
|--iphy
|--[0..1] lane
|--lane
[check] signal_detect
[check] pll_lock
|--xphy_sys
[check] config
|--[0..1] lane
|--lane
[check] peak
[check] vga
[check] SNR
|--xphy_line
|--[0..3] lane
|--lane
[check] signal_detect
[check] pll_lock
|--qsfp
[check] LOS
```
The checks can be linked with dependency relations.

An interface[eth2/2/1] check depends on the following conditions.
```[
"..asic.port2 == OK",
"..platform.pim1.xphy2 == OK",
"..platform.pim1.qsfp10 == OK",
]
```
Here dependencies( type 3) and hw mapping (type 1) are also auto-loaded. 

When doing the link down debugging, the command is diag -n host=\<hostname> tor:system.intf[eth2/2/1] check.

The tool enters the interface node "intf[eth2/2/1]", and runs the check command.
It follows this dependency list, and check asic port, qsfp, and xphy.  Why reaching xphy, it sees xphy's dependency is specified as "all lane[0-3] = OK", so it go ahead checks each lane.  For the lane, the list of checks are "LOS, PLL_lock".  

## Command execution

Directly running the domain specific tool would require much domain knowledge such as hardware mapping (type 1) and tool usage (type 4).
When checking the external PHY (xphy), we need to find the xphy id, lane mask, and know how to run it, and interpret the results.  The command could be from a vendor, or from an internal tool.  Each could be different depending on the vendor and model. This framework acts a OO wrapper, encapsulate all these details, and present just a simple check command as xphy.lane.[check].

The domain specific command which requires much domain knowledge as below
```
bcm_shell fsw005.p076.f01.pnb2 --timeout 3000 -c 'xphy lw_dsc phy_id=0x48 if_side=1 lane_mask=0x100000'
```
The framework simplifies it to
```
 eth4/9/1.link.xphy_sys.lane1 show
```
## Benefits
This structure is flexible.  This flexibility brings many benefits.

1.  It can be easily extended.  When we observe a new link down cause, we could add a new dependency entry, so the next person running the tool can see the cause quickly without repeating the same debugging.

1.  It can easily aggregate to support large systems.
```
	|Fleet
	|--data_center [1…10]
	|--|--|rack [1..1000]
	|--|--|--tor
	|--|--|--servers [1..100]
```
1.  It can also be taken apart.  The xphy part can be shared with the phy vendor so they can refine the phy internal structure and commands.

It has proper separation of ownership.  For each sub system, the domain experts can work on refining the debugging commands for that specific node.  All work from experts on all domain collectively form a comprehensive and powerful tool for debugging it the full system.

It also acts like wiki.  It actually eliminates the need of much information in the wiki because the required domain information has been embedded into the code.  If one is indeed interested in learning the vendor commands details, it is much easier to just see the raw commands from the log and see how is invoked, rather than reading wiki and try running manually.

It can be particularly useful when debugging a large scale infra, when the problem involves many sub systems, and there is no one knowing every sub system.  We can just dive into the structure, follow the chain of the check dependencies, and drill deep down into low level nodes to root cause the problem.  It is a very scalable tool to handle the complex infra at scale, especially for cross-team debugging.

Not only for diagnostics, it is also useful for some general software development work such as porting a large module to a new platform, bringing up a new hardware.  The hierarchy of checks is a systematic way to quickly discover a large number of discrepencies.


