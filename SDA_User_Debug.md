# SDAccel Debug Guide

This documenmt describes ...

Check spelling!!

## Overview


## Software Emulation Debug

## Hardware Emulation Debug

## Systsem Debug

Debugging AWS F1 designs running in-system is often required when functional or timing-related issues are not easily 
discovered using other debugging techniques such as Software Emulation or Hardware Emulation.  

The use of ChipScope debug IP such as System ILA, ILA, and VIO can help AWS F1 users by providing integrated debug solutions 
that improve observability into their designs and minimize the impact of design QoR. The AWS F1 platform contains the Xilinx 
Virtual Cable (XVC) debug infrastructure that allows users to access ChipScope debug cores over the host-card PCIe interface. 
You can learn more about this infrastructure [here][Xilinx Virtual Cable].  

This section of the SDAccel Debug Guide discusses techniques for adding ChipScope debug cores to your AWS F1 SDx-flow design.

### Debug methodology
The recomended debug methodology debug in System mode periodically throughout the flow, not just at the end of your flow as 
a last resort.

#### SDx-HLS kernel flows
1. Choose a relatively simple compiler pragmas/directives and get at least one iteration to work in System mode fairly early 
in the design cycle:
    - This helps rule out host-kernel interaction issues
    - This establishes a baseline for all future optimization iterations
1. In order to meet performance objectives, enable various compiler pragmas/directives (ideally, one at a time) and verify in 
both Hardware Emulation and System modes.
1. If a functional issues is uncovered in System mode, try to correlate this with Hardware Emulation mode, if possible.  
    - Try backing out any optimization modifications to see if the design works as expected in hardware emulation mode.

#### SDx-RTL kernel flow
1. For debugging kernel hangs, verify that the AXI interfaces to/from your kernel are not violating the AXI protocol and that 
data is flowing as expected. Common AXI issues found when designing your own RTL kernel include 
    - Improper AXI channel valid/ready handshaking logic 
    - Insufficient read data buffering
    - Improper tracking of transaction data beats, etc.
1. For debugging improper kernel results, verify that the AXI transaction data at the kernel interface from the global memory 
to the kernel, and vice versa, looks correct.  
    - If the AXI transaction data from the global memory looks incorrect, verify that the data from the host to the global memory 
looks correct and that your kernel is reading the data correctly (i.e., proper address, byte lane enablement, etc.).  
    - If the data from the kernel compute unit to the global memory looks incorrect, you might need to probe various pipeline stages of your kernel data path to ensure intermediate results look correct.

### Selecting Signals to Probe 
The selection of which signals to probe depends on whether the issue is:
1. Kernel Hangs
1. Incorrect Data 

#### Kernel Hangs
When debugging kernel hangs, it is best to probe at the control slave interface and each of the global memory master interfaces 
of the offending kernel compute unit interface. 

If possible, use the **xocc --dk chipscope** [feature][DK_ref] to identify which off the kernel compute unit slave and master interfaces to debug 
using a System ILA core. By default, the System ILA will have the following default settings:
- No native probe interfaces
- Data depth of 4096 samples
- Two input pipeline stages
- Cross-triggering disabled
- Advanced trigger disabled
- Storage qualification disabled
- AXI protocol checker assertion enabled : This indicates if any protocol violation was encountered
- AXI protocol checker status disabled : This indicates which protocol violations were encountered
- Tracking of up to 64 AXI outstanding read and write transactions

You can change these, and any other System ILA settings using the advanced Tcl script flow described [below](#System_ILA_Tcl)

#### Incorrect Data
When debugging incorrect data, it is best to probe at the global memory master interface(s) where incorrect read/write data 
is going into/out of the kernel.

For RTL kernels, you can add ILA debug cores directly inside your RTL kernel to probe whatever signals of interest you 
identify in your design.  

- **Keep in mind** that if you are debugging AXI interface signals, it is usually best to do so from 
outside of the kernel using the **xocc --dk chipscope** [feature][DK_ref] in order to get the full AXI protocol transaction display capabilities 
of the System ILA core.  
- However, if you need to correlate AXI transactions with other signals inside your RTL kernel, consider  using the post-route 
design probing technique to attach these intra-kernel signals to the System ILA core that is probing the 
external interfaces of your kernel. Refer to the [Probing Internal Signals](#Plan_Debug_Nets) section for more details.

### Adding debug cores to your design

#### HDK flow
Use the standard [IP core instantiation flow][IP_core_Flow] outlined below to add ILA and/or VIO cores to your design
1. Create "manage ip" project
1. Select, customize, and generate appropriate debug IP
1. Instantiate IP inside your CL design
1. Vivado debug flow automatically detects and connects your debug IP with the existing debug bridge infrastructure

Use the ILA core that is debugging the AXI interface between the Shell and Kernel logic.

#### SDx flow

Note: For RTL kernels use the standard [IP core instantiation flow][IP_core_Flow] to add ILA and/or VIO cores to your design. These debug cores will automatically **...looks like missing end-of-sentence ????**

There are two types of signals you may wish to monitor.
- Signals which are ports on the Kernel compute unit
- Signals which are nets inside the Kernel compute unit


##### Probing Port Signals
Use the **xocc --dk chipscope** [feature][DK_ref] to add System ILA core(s) to your design.

<a name="Plan_Debug_Nets"></a>
##### Probing Internal Signals
Follow this two-step process to probe signals which are internal to the Kernel compute unit.
- Modify/Add an ILA core to the design.
- Modify the port connections to connect to the ILA.

###### Add/modify System ILA core to include extra "native probe ports"
The procedure for selecting signals to monitor depends on whether:
- You [**have**](#Have_ILA_Core) an existing System ILA core in your design 
- You [**do not have**](#No_ILA_Core) an existing System ILA core in your design

<a name="Have_ILA_Core"></a>
If you **have** an existing System ILA core in your design (for instance, added using the "--dk chipscope" option), then you need to modify it to add one or more "native probe" ports and tie them off to ground (e.g., logical '0'). 
- You do this by first creating a Vivado Tcl script (which we'll call "/tmp/myproj/sys_ila_adv_settings.tcl") to modify the IP Integrator block design as follows:
    - Enable one or more "native probe" port(s) in addition to your existing "interface" probe ports.  For instance, to add a single 
    32-bit native probe port:
							# Customize the System ILA core
							set_property -dict \
							  [list \
							    CONFIG.C_MON_TYPE {MIX} \
							    CONFIG.C_PROBE_WIDTH_PROPAGATION {MANUAL} \
							    CONFIG.C_NUM_OF_PROBES {1} \
							    CONFIG.C_PROBE0_WIDTH {32} \
							  ] [get_bd_cells system_ila_0]
						◊ Add an xlconstant block that will be used to tie off the native probe port added above to "ground" (i.e., logical '0'):
							# Add 32-bit "ground" constant blocks that will be used to 
							# tie off the 32-bit native probe ports of the System ILA
							create_bd_cell -type ip -vlnv xilinx.com:ip:xlconstant:1.1 sys_ila_probe_tieoff32_0
							set_property -dict [list CONFIG.CONST_WIDTH {32}  CONFIG.CONST_VAL {0}] [get_bd_cells sys_ila_probe_tieoff32_0]
						◊ Connect the xlconstant block to the new native probe port of the System ILA core
							connect_bd_net [get_bd_pins sys_ila_probe_tieoff32_0/dout] [get_bd_pins system_ila_0/probe0]
					b) If you DO NOT have an existing System ILA core in your design, then you need to create one that includes one or more "native probe" ports that are tied off to ground (e.g., logical '0'). You do this by first creating a Vivado Tcl script (which we'll call "/tmp/myproj/sys_ila_adv_settings.tcl") to modify the IP Integrator block design as follows:
							◊ Create a new System ILA core:
								# Create the System ILA core
								create_bd_cell -type ip -vlnv xilinx.com:ip:system_ila:1.1 system_ila_0
							◊ Enable one or more "native probe" port(s).  For instance, to add a single 32-bit native probe port:
								# Customize the System ILA core
								set_property -dict \
								  [list \
								    CONFIG.C_DATA_DEPTH {4096} \
								    CONFIG.C_INPUT_PIPE_STAGES {2} \
								    CONFIG.C_MON_TYPE {NATIVE} \
								    CONFIG.C_PROBE_WIDTH_PROPAGATION {MANUAL} \
								    CONFIG.C_NUM_OF_PROBES {1} \
								    CONFIG.C_PROBE0_WIDTH {32} \
								  ] [get_bd_cells system_ila_0]
							◊ Add an xlconstant block that will be used to tie off the native probe port added above to "ground" (i.e., logical '0'):
								# Add 32-bit "ground" constant blocks that will be used to 
								# tie off the 32-bit native probe ports of the System ILA
								create_bd_cell -type ip -vlnv xilinx.com:ip:xlconstant:1.1 sys_ila_probe_tieoff32_0
								set_property -dict [list CONFIG.CONST_WIDTH {32}  CONFIG.CONST_VAL {0}] [get_bd_cells sys_ila_probe_tieoff32_0]
							◊ Connect the xlconstant block to the new native probe port of the System ILA core
								connect_bd_net [get_bd_pins sys_ila_probe_tieoff32_0/dout] [get_bd_pins system_ila_0/probe0]
					c) Invoke the Vivado Tcl script described above (e.g., "/tmp/myproj/sys_ila_adv_settings.tcl") immediately following the system linker step of the xocc compile run:
						xocc --xp param:compiler.userPostSysLinkTcl=/tmp/proj/sys_ila_adv_settings.tcl …
					Note: the param:compiler.userPostSysLinkTcl parameter requires an absolute path to the Vivado Tcl script.
				2) Modify debug port connections to probe signals in post-route design
					a) Once you run your kernel design in hardware execution mode and determine you would like to debug an intra-kernel signal (for example, elements an intra-kernel 32-bit bus net called "WRAPPER_INST/CL/krnl_vadd_1/inst/c_tmp_q[31:0]"), you need to create a Vivado Tcl script (which we'll call "/tmp/myproj/modify_sys_ila_probes.tcl") that calls the "modify_debug_ports" command to connect the probe0[31:0] port to the 32-bit bus net:
						modify_debug_ports -probes \
						  [list \
						    {WRAPPER_INST/CL/system_ila_0/inst/ila_lib/probe0 0  WRAPPER_INST/CL/krnl_vadd_1/inst/c_tmp_q[0]} \
						    {WRAPPER_INST/CL/system_ila_0/inst/ila_lib/probe0 1  WRAPPER_INST/CL/krnl_vadd_1/inst/c_tmp_q[1]} \
						    {WRAPPER_INST/CL/system_ila_0/inst/ila_lib/probe0 2  WRAPPER_INST/CL/krnl_vadd_1/inst/c_tmp_q[2]} \
						    {WRAPPER_INST/CL/system_ila_0/inst/ila_lib/probe0 3  WRAPPER_INST/CL/krnl_vadd_1/inst/c_tmp_q[3]} \
						    {WRAPPER_INST/CL/system_ila_0/inst/ila_lib/probe0 4  WRAPPER_INST/CL/krnl_vadd_1/inst/c_tmp_q[4]} \
						    {WRAPPER_INST/CL/system_ila_0/inst/ila_lib/probe0 5  WRAPPER_INST/CL/krnl_vadd_1/inst/c_tmp_q[5]} \
						    {WRAPPER_INST/CL/system_ila_0/inst/ila_lib/probe0 6  WRAPPER_INST/CL/krnl_vadd_1/inst/c_tmp_q[6]} \
						    {WRAPPER_INST/CL/system_ila_0/inst/ila_lib/probe0 7  WRAPPER_INST/CL/krnl_vadd_1/inst/c_tmp_q[7]} \
						    {WRAPPER_INST/CL/system_ila_0/inst/ila_lib/probe0 8  WRAPPER_INST/CL/krnl_vadd_1/inst/c_tmp_q[8]} \
						    {WRAPPER_INST/CL/system_ila_0/inst/ila_lib/probe0 9  WRAPPER_INST/CL/krnl_vadd_1/inst/c_tmp_q[9]} \
						    {WRAPPER_INST/CL/system_ila_0/inst/ila_lib/probe0 10 WRAPPER_INST/CL/krnl_vadd_1/inst/c_tmp_q[10]} \
						    {WRAPPER_INST/CL/system_ila_0/inst/ila_lib/probe0 11 WRAPPER_INST/CL/krnl_vadd_1/inst/c_tmp_q[11]} \
						    {WRAPPER_INST/CL/system_ila_0/inst/ila_lib/probe0 12 WRAPPER_INST/CL/krnl_vadd_1/inst/c_tmp_q[12]} \
						    {WRAPPER_INST/CL/system_ila_0/inst/ila_lib/probe0 13 WRAPPER_INST/CL/krnl_vadd_1/inst/c_tmp_q[13]} \
						    {WRAPPER_INST/CL/system_ila_0/inst/ila_lib/probe0 14 WRAPPER_INST/CL/krnl_vadd_1/inst/c_tmp_q[14]} \
						    {WRAPPER_INST/CL/system_ila_0/inst/ila_lib/probe0 15 WRAPPER_INST/CL/krnl_vadd_1/inst/c_tmp_q[15]} \
						    {WRAPPER_INST/CL/system_ila_0/inst/ila_lib/probe0 16 WRAPPER_INST/CL/krnl_vadd_1/inst/c_tmp_q[16]} \
						    {WRAPPER_INST/CL/system_ila_0/inst/ila_lib/probe0 17 WRAPPER_INST/CL/krnl_vadd_1/inst/c_tmp_q[17]} \
						    {WRAPPER_INST/CL/system_ila_0/inst/ila_lib/probe0 18 WRAPPER_INST/CL/krnl_vadd_1/inst/c_tmp_q[18]} \
						    {WRAPPER_INST/CL/system_ila_0/inst/ila_lib/probe0 19 WRAPPER_INST/CL/krnl_vadd_1/inst/c_tmp_q[19]} \
						    {WRAPPER_INST/CL/system_ila_0/inst/ila_lib/probe0 20 WRAPPER_INST/CL/krnl_vadd_1/inst/c_tmp_q[20]} \
						    {WRAPPER_INST/CL/system_ila_0/inst/ila_lib/probe0 21 WRAPPER_INST/CL/krnl_vadd_1/inst/c_tmp_q[21]} \
						    {WRAPPER_INST/CL/system_ila_0/inst/ila_lib/probe0 22 WRAPPER_INST/CL/krnl_vadd_1/inst/c_tmp_q[22]} \
						    {WRAPPER_INST/CL/system_ila_0/inst/ila_lib/probe0 23 WRAPPER_INST/CL/krnl_vadd_1/inst/c_tmp_q[23]} \
						    {WRAPPER_INST/CL/system_ila_0/inst/ila_lib/probe0 24 WRAPPER_INST/CL/krnl_vadd_1/inst/c_tmp_q[24]} \
						    {WRAPPER_INST/CL/system_ila_0/inst/ila_lib/probe0 25 WRAPPER_INST/CL/krnl_vadd_1/inst/c_tmp_q[25]} \
						    {WRAPPER_INST/CL/system_ila_0/inst/ila_lib/probe0 26 WRAPPER_INST/CL/krnl_vadd_1/inst/c_tmp_q[26]} \
						    {WRAPPER_INST/CL/system_ila_0/inst/ila_lib/probe0 27 WRAPPER_INST/CL/krnl_vadd_1/inst/c_tmp_q[27]} \
						    {WRAPPER_INST/CL/system_ila_0/inst/ila_lib/probe0 28 WRAPPER_INST/CL/krnl_vadd_1/inst/c_tmp_q[28]} \
						    {WRAPPER_INST/CL/system_ila_0/inst/ila_lib/probe0 29 WRAPPER_INST/CL/krnl_vadd_1/inst/c_tmp_q[29]} \
						    {WRAPPER_INST/CL/system_ila_0/inst/ila_lib/probe0 30 WRAPPER_INST/CL/krnl_vadd_1/inst/c_tmp_q[30]} \
						    {WRAPPER_INST/CL/system_ila_0/inst/ila_lib/probe0 31 WRAPPER_INST/CL/krnl_vadd_1/inst/c_tmp_q[31]} \
						  ]
					b) Invoke the Vivado Tcl script described above  (e.g., "/tmp/myproj/modify_sys_ila_probes.tcl") immediately following the route_design step of the xocc compile run:
						xocc --xp vivado_prop:run.impl_1.STEPS.ROUTE_DESIGN.TCL.POST=/tmp/myproj/modify_sys_ila_probes.tcl …
					Note:  the vivado_prop:run.impl_1.STEPS.ROUTE_DESIGN.TCL.POST parameter requires an absolute path to the Vivado Tcl script.


###### Adding a System ILA core from scratch
If you DO NOT have an existing System ILA core in your design, then you need to create one that includes one or more **native probe** 
ports that are tied off to ground (e.g., logical '0'). 

1. First creating a Vivado Tcl script (which we'll call "/tmp/myproj/sys_ila_adv_settings.tcl") to modify the IP Integrator 
block design as follows:
    1.1 Create a new System ILA core
    ``` 
        # Create the System ILA core
				create_bd_cell -type ip -vlnv xilinx.com:ip:system_ila:1.1 system_ila_0
    ``` 
		1.2 Enable one or more **native probe* port(s). For instance, to add a single 32-bit native probe port:
    ``` 
				# Customize the System ILA core
				set_property -dict \
				  [list \
				  CONFIG.C_DATA_DEPTH {4096} \
					CONFIG.C_INPUT_PIPE_STAGES {2} \
					CONFIG.C_MON_TYPE {NATIVE} \
					CONFIG.C_PROBE_WIDTH_PROPAGATION {MANUAL} \
					CONFIG.C_NUM_OF_PROBES {1} \
					CONFIG.C_PROBE0_WIDTH {32} \
					] [get_bd_cells system_ila_0]
		``` 
		1.3 Add an xlconstant block that will be used to tie off the native probe port added above to "ground" (i.e., logical '0'):
	      ``` 
				# Add 32-bit "ground" constant blocks that will be used to 
				# tie off the 32-bit native probe ports of the System ILA
				create_bd_cell -type ip -vlnv xilinx.com:ip:xlconstant:1.1 sys_ila_probe_tieoff32_0
				set_property -dict [list CONFIG.CONST_WIDTH {32}  CONFIG.CONST_VAL {0}] [get_bd_cells sys_ila_probe_tieoff32_0]
	      ``` 
		1.4 Connect the xlconstant block to the new native probe port of the System ILA core
		    ``` 
				connect_bd_net [get_bd_pins sys_ila_probe_tieoff32_0/dout] [get_bd_pins system_ila_0/probe0]
		    ``` 
    1.5 Run xocc with the newly script Tcl
    Invoke the Vivado Tcl script described above (e.g., "/tmp/myproj/sys_ila_adv_settings.tcl") immediately following the system 
    **linker step** of the xocc compile run:
	    ``` 
	    xocc --xp param:compiler.userPostSysLinkTcl=/tmp/proj/sys_ila_adv_settings.tcl …
	    ``` 
    **Note**: the param:compiler.userPostSysLinkTcl parameter requires an absolute path to the Vivado Tcl script.
    
1. Modify debug port connections to probe signals in post-route design
    1.1 Once you run your kernel design in System mode and determine you would like to debug an intra-kernel signal (for example, 
    elements an intra-kernel 32-bit bus net called "WRAPPER_INST/CL/krnl_vadd_1/inst/c_tmp_q[31:0]"), you need to create a Vivado 
    Tcl script (which we'll call "/tmp/myproj/modify_sys_ila_probes.tcl") that calls the **modify_debug_ports** command to connect 
    the probe0[31:0] port to the 32-bit bus net:
      ```
			modify_debug_ports -probes \
			    [list \
				  {WRAPPER_INST/CL/system_ila_0/inst/ila_lib/probe0 0  WRAPPER_INST/CL/krnl_vadd_1/inst/c_tmp_q[0]} \
					{WRAPPER_INST/CL/system_ila_0/inst/ila_lib/probe0 1  WRAPPER_INST/CL/krnl_vadd_1/inst/c_tmp_q[1]} \
					{WRAPPER_INST/CL/system_ila_0/inst/ila_lib/probe0 2  WRAPPER_INST/CL/krnl_vadd_1/inst/c_tmp_q[2]} \
					{WRAPPER_INST/CL/system_ila_0/inst/ila_lib/probe0 3  WRAPPER_INST/CL/krnl_vadd_1/inst/c_tmp_q[3]} \
					{WRAPPER_INST/CL/system_ila_0/inst/ila_lib/probe0 4  WRAPPER_INST/CL/krnl_vadd_1/inst/c_tmp_q[4]} \
					{WRAPPER_INST/CL/system_ila_0/inst/ila_lib/probe0 5  WRAPPER_INST/CL/krnl_vadd_1/inst/c_tmp_q[5]} \
					{WRAPPER_INST/CL/system_ila_0/inst/ila_lib/probe0 6  WRAPPER_INST/CL/krnl_vadd_1/inst/c_tmp_q[6]} \
					{WRAPPER_INST/CL/system_ila_0/inst/ila_lib/probe0 7  WRAPPER_INST/CL/krnl_vadd_1/inst/c_tmp_q[7]} \
					{WRAPPER_INST/CL/system_ila_0/inst/ila_lib/probe0 8  WRAPPER_INST/CL/krnl_vadd_1/inst/c_tmp_q[8]} \
					{WRAPPER_INST/CL/system_ila_0/inst/ila_lib/probe0 9  WRAPPER_INST/CL/krnl_vadd_1/inst/c_tmp_q[9]} \
					{WRAPPER_INST/CL/system_ila_0/inst/ila_lib/probe0 10 WRAPPER_INST/CL/krnl_vadd_1/inst/c_tmp_q[10]} \
					{WRAPPER_INST/CL/system_ila_0/inst/ila_lib/probe0 11 WRAPPER_INST/CL/krnl_vadd_1/inst/c_tmp_q[11]} \
					{WRAPPER_INST/CL/system_ila_0/inst/ila_lib/probe0 12 WRAPPER_INST/CL/krnl_vadd_1/inst/c_tmp_q[12]} \
					{WRAPPER_INST/CL/system_ila_0/inst/ila_lib/probe0 13 WRAPPER_INST/CL/krnl_vadd_1/inst/c_tmp_q[13]} \
					{WRAPPER_INST/CL/system_ila_0/inst/ila_lib/probe0 14 WRAPPER_INST/CL/krnl_vadd_1/inst/c_tmp_q[14]} \
					{WRAPPER_INST/CL/system_ila_0/inst/ila_lib/probe0 15 WRAPPER_INST/CL/krnl_vadd_1/inst/c_tmp_q[15]} \
					{WRAPPER_INST/CL/system_ila_0/inst/ila_lib/probe0 16 WRAPPER_INST/CL/krnl_vadd_1/inst/c_tmp_q[16]} \
					{WRAPPER_INST/CL/system_ila_0/inst/ila_lib/probe0 17 WRAPPER_INST/CL/krnl_vadd_1/inst/c_tmp_q[17]} \
				  {WRAPPER_INST/CL/system_ila_0/inst/ila_lib/probe0 18 WRAPPER_INST/CL/krnl_vadd_1/inst/c_tmp_q[18]} \
					{WRAPPER_INST/CL/system_ila_0/inst/ila_lib/probe0 19 WRAPPER_INST/CL/krnl_vadd_1/inst/c_tmp_q[19]} \
				  {WRAPPER_INST/CL/system_ila_0/inst/ila_lib/probe0 20 WRAPPER_INST/CL/krnl_vadd_1/inst/c_tmp_q[20]} \
					{WRAPPER_INST/CL/system_ila_0/inst/ila_lib/probe0 21 WRAPPER_INST/CL/krnl_vadd_1/inst/c_tmp_q[21]} \
					{WRAPPER_INST/CL/system_ila_0/inst/ila_lib/probe0 22 WRAPPER_INST/CL/krnl_vadd_1/inst/c_tmp_q[22]} \
					{WRAPPER_INST/CL/system_ila_0/inst/ila_lib/probe0 23 WRAPPER_INST/CL/krnl_vadd_1/inst/c_tmp_q[23]} \
					{WRAPPER_INST/CL/system_ila_0/inst/ila_lib/probe0 24 WRAPPER_INST/CL/krnl_vadd_1/inst/c_tmp_q[24]} \
					{WRAPPER_INST/CL/system_ila_0/inst/ila_lib/probe0 25 WRAPPER_INST/CL/krnl_vadd_1/inst/c_tmp_q[25]} \
					{WRAPPER_INST/CL/system_ila_0/inst/ila_lib/probe0 26 WRAPPER_INST/CL/krnl_vadd_1/inst/c_tmp_q[26]} \
					{WRAPPER_INST/CL/system_ila_0/inst/ila_lib/probe0 27 WRAPPER_INST/CL/krnl_vadd_1/inst/c_tmp_q[27]} \
					{WRAPPER_INST/CL/system_ila_0/inst/ila_lib/probe0 28 WRAPPER_INST/CL/krnl_vadd_1/inst/c_tmp_q[28]} \
					{WRAPPER_INST/CL/system_ila_0/inst/ila_lib/probe0 29 WRAPPER_INST/CL/krnl_vadd_1/inst/c_tmp_q[29]} \
					{WRAPPER_INST/CL/system_ila_0/inst/ila_lib/probe0 30 WRAPPER_INST/CL/krnl_vadd_1/inst/c_tmp_q[30]} \
					{WRAPPER_INST/CL/system_ila_0/inst/ila_lib/probe0 31 WRAPPER_INST/CL/krnl_vadd_1/inst/c_tmp_q[31]} \
					]
      ```
  1.2 Invoke the Vivado Tcl script described above  (e.g., "/tmp/myproj/modify_sys_ila_probes.tcl") immediately following the 
  **route_design** step of the xocc compile run:
      ```
			  xocc --xp vivado_prop:run.impl_1.STEPS.ROUTE_DESIGN.TCL.POST=/tmp/myproj/modify_sys_ila_probes.tcl …
      ```
	**Note**:  the vivado_prop:run.impl_1.STEPS.ROUTE_DESIGN.TCL.POST parameter requires an absolute path to the Vivado Tcl script.

###<a name="System_ILA_Tcl"></a>Changing the System ILA settings
In order to change the default settings of any System ILA core in your design, you need to create a Vivado Tcl script 
(e.g., /tmp/myproj/sys_ila_adv_settings.tcl") that will be run in the post-system linker step of the xocc compile run.

Here's how you make various settings changes to a System ILA core called "system_ila_0":

- To change the data depth (default is 4096; valid values are 1024, 2048, 4096, 8192, and 16384):
      ```
		  set_property -dict \
			    [list \
				  CONFIG.C_DATA_DEPTH {8192} \
				  ] [get_bd_cells system_ila_0]
      ```
- To change the number of input pipe stages (default is 2; valid values are integers 0 through 6).  Input pipe stages on the System 
ILA core make it easier for the place_design, route_design, and modify_debug_ports compile steps to achieve design closure with minimal 
impact on design quality of results:
      ```
			set_property -dict \
				  [list \
				  CONFIG.C_INPUT_PIPE_STAGES {3} \
				  ] [get_bd_cells system_ila_0]
	      ```
	- To enable storage qualification (default is '0' for disabled; valid values are '0' for disabled and '1' for enabled).  Storage 
  qualification is used to filter probe data values in order to maximize the data capture buffer space of the System ILA core.  It 
  is recommended to increment the number of comparators for all probe ports by 1 when enabling storage qualification:
      ```
			set_property -dict \
				  [list \
				  CONFIG.C_EN_STRG_QUAL {1} \
				  CONFIG.ALL_PROBE_SAME_MU_CNT {2} \
				  ] [get_bd_cells system_ila_0]
	      ```
	- To enable advanced trigger state machine (default is "false" for disabled; valid values are "false" for disabled and "true" for 
  enabled). Advanced trigger state machine is used to build multi-stage trigger conditions that can be used to trigger on complex 
  events in hardware. It is recommended to increment the number of comparators for all probe ports by 2 when enabling advanced 
  triggering:
	    ```
			set_property -dict \
				  [list \
				  CONFIG.C_ADV_TRIGGER {true} \
				  CONFIG.ALL_PROBE_SAME_MU_CNT {3} \
				  ] [get_bd_cells system_ila_0]
- To enable both storage qualification and advanced trigger state machine. It is recommended to increment the number of comparators for all probe ports by 3 when enabling both storage qualification and advanced triggering:
      ```
			set_property -dict \
				  [list \
				  CONFIG.C_EN_STRG_QUAL {1} \
				  CONFIG.C_ADV_TRIGGER {true} \
				  CONFIG.ALL_PROBE_SAME_MU_CNT {4} \
				  ] [get_bd_cells system_ila_0]
      ```
Invoke the Vivado Tcl script described above (e.g., "/tmp/myproj/sys_ila_adv_settings.tcl") immediately following the system **linker** 
step of the xocc compile run:
  ```
	    xocc --xp param:compiler.userPostSysLinkTcl=/tmp/proj/sys_ila_adv_settings.tcl …
  ```
**Note**: The param:compiler.userPostSysLinkTcl parameter requires an absolute path to the Vivado Tcl script.

# Additional Resources


[Xilinx Virtual Cable]:https://www.xilinx.com/products/intellectual-property/xvc.html (Needs reviewed)
[DK_ref]:https://www.xilinx.com/products/intellectual-property/xvc.html (Need Correct URL)
[IP_core_Flow]:https://www.xilinx.com/products/intellectual-property/xvc.html (Need Correct URL)

