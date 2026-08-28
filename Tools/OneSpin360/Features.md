# Command-Line Options

Unix (Bash) commands used to run onespin

Run in GUI:
> onespin

Run in Unix (Bash) terminal:
> onespin --gui=no

Run script and terminate:
> onespin \<script\>.tcl

Further options:
> onespin --help

Following sections assume onespin has been **already executed**.

## Session management
Modify the project directory using the -project_directory session option. There are more ways to do this.
> setup\> set_session_option -project_directory /mnt/my_project_home

Set command log file:
> \> set_session_option -command_log_file \<file\>

Log commands called in the sourced script:
> \> set_session_option -debug_output

## Managing Resources
Some relevant resource management commands:
Clear design cache:
> release_proof_memory

Set limits for real and CPU time of commands and checks, and for global memory:
> set_limit

Check current limits:
> report_limits

## Message Filtering
Allows to increase readability of the program output.
Example:
> add_message_filter -class Info -category "*" -memo "VHDL Analyzer*" -text "*"

List filters:
> report_message_filter

hit counter indicates how many times a message was supressed by using the filter
Reset hit counter:
> report_message_filter -reset_statistics

Activate, deactivate and remove filters:
> set_message_filter -remove {1 3}
> set_message_filter -activate 2

## Message statistics
Show statistics about not filtered messages:
> report_message_statistics -printed

Show statistics about filtered messages:
> report_message_statistics -filtered

## Executing code at startup
TCL script:
> setenv ONESPIN_STARTUP /home/verifier/onespin_convenient.tcl:/\home/project/my_project/compile.tcl

Bourne-compatible shell, such as sh or bash, the corresponding syntax is:
> ONESPIN_STARTUP=/home/verifier/onespin_convenient.tcl:\/home/project/my_project/compile.tcl
> export ONESPIN_STARTUP

## Configure default log files
> set suffix \[ clock format \[ clock seconds ] -format "%Y-%m-%d_%H:%M:%S" ]
> start_message_log "/tmp/\$env(USER)/onespin_${suffix}_msg.log"
> start_command_log "/tmp/\$env(USER)/onespin_${suffix}_cmd.log"

## Passing arguments to onespin
The script testscript.tcl:

>puts "count: $argc"
>puts "values: $argv"
>puts "first arg: \[lindex \$argv 0]"
>puts "script name: \$argv0"

when called with the command onespin testscript.tcl 111 222 333
will produce this output:
> OneSpin 360 (R) - Version ...
> 
>count:            3
>values:           111 222 333
>first arg:        111
>script name: testscript.tcl


## Modes

Change mode:
> set_mode \<mode\>

### Setup (setup)
- read in HDL files
- elaborate
- compile
- end the setup mode and change to a verification mode (ec, mv, cc)

**Read in HDL Files**
>setup> set_read_hdl_option -verilog_version 1995 -verilog_define bar=1 -verilog_include_path ~/my_proj/include -golden

> setup> read_vhdl foo.vhd
> setup> read_verilog bar.v



**Elaborate**
The elaboration puts the different elements of a **design previously read** in together, starting at a specific top level element.
Most common elaborate options are:
- values of parameter (verilog) or generics (vhdl)
- modules to black box
- top level module

Example:
>setup> set_elaborate_option -vhdl_generic gen=0 -verilog_parameter p=1 -golden
>setup> elaborate

There are 2 designs: golden and revised.
The session data contains a current design. Commands that work on designs apply to the current
design, if no other design is specified at command line by one of the options -golden, -revised, or
-both.
Note that for module verification, the **revised design is ignored** and only the **golden** design can be used.

**Compile**
In the OneSpin shell a **unit** is a part of the design. It is possible to create several units from one design. A unit can be derived from a design by **setting the top level of the unit to a sub-module** of the design or by **black-boxing** module instances.

Example:
>setup> set_compile_option -black_box ram -golden -top \<module\>
>setup> compile

![[relation_design_and_unit.png]]

After compile, it is possible to overwrite the clocks and reset behavior:
Example:
```
setup> add_clock_spec -period 2 ahb_clk
setup> add_clock_spec -period 4 apb_clk
```
```
setup> set_reset_sequence -low res
```

### 3.3 Design Modeling
This is a relevant section in order to understand the modeling of the design in the verification, limitations and capabilities. **Only some subsections will be mentioned here**.

**3.3.9 Clocks and Generated Clocks**

### 3.4.4 Specifying the Clocking for a Unit
At any time, all clocks of a unit have a current clock specification. Use report_clock_spec to view
the current clock specifications, or use get_clock_spec to retrieve a TCL list of these settings in TCL list format. To modify the clock specification of a particular or all clocks use set_clock_spec (see also add_clock_spec, delete_clock_spec).

If there are several primary clock inputs (multi clock DUV), all clocks are set up free. This means
there are no relations between the clocks, and in addition, each clocks can behave arbitrarily, i.e.,
it does not have to tick periodically.

In order to set up all design clocks as equal (i.e., fully synchronous identical clocks), one can use the following command
set_clock_spec -period 2 \[get_bits -unit -filter clock!=none&&direction==input]



### Equivalence Checking (ec)

### Complete module verification (mv)

### Consistency Checking (cc)

## Handling of Data
OneSpin TCL shell sabes settings and results in a database

> save_database
> 
> load_database

The database format is not compatible between two different releases of OneSpin

## General Concepts of 360 DV

### Parallel Computations
The OnesPin 360 distribution system allows to run multiple tasks in parallel.
It can be used for auto, assertion, property, completeness checking and witness computations.
There are Multiple ways to do this, but lets focus on the local execution of tasks.

```
mv> check -local_processes 4 -parallel local       [get_checks]
mv> set_check_options -local_processes 4 -parallel local

```

```
get_check_options -local_processes -parallel
```

**Debuging**:
For debugging purposes it is possible to redirect the standard output and error channels of a worker process by setting the environment variable **$ONESPIN_SERVER_LOG_FOLDER**. OneSpin 360 will create a log file for each worker process that is started. File names match the following pattern:
```
server_<pid>.log
```
## SVA

### 6.6 Constraints, Assumptions and Dependencies

**Assumes visibility:**
- By default, all assumes are used for all **consistency, assertion, and property checks**.
- In order to re-activate the legacy handling of assumes in 360 DV, you can set the check option `-no_automatic_constraints`. Legacy mode implies the following:
	- All assumes are used for all assertions contained inside the same module.
	- Inside a module, SVA blocks are used to create another indepedent hierarchy of dependencies, which implies that the assumes contained inside don't apply to the assertions found outside.

SVA blocks format
```Verilog
‘define begin_(name) if (1) begin: name

// Begin block
`begin(name)
	// assumes and assertions
end //name
```

Nesting of SVA blocks is supported. Each SVA block creates a new namespace for assertions and
assume statements. The implicit dependencies of a given assertion A are all the assume statements in A’s block and all the assume statements in the enclosing blocks.
For example, the implicit dependencies of `A3` in the following code fragment are `block1/C1, block1_2/C2` and `block1_2/C4`.
```Verilog
begin: block1
	C1: assume property ...
	begin
		C2: assume property ...
		C3: assume property ...
		A2: assert property ...
	end
	begin: block1_2
		C2: assume property ...
		C4: assume property ...
		A3:
		assert property ...
	end // block1_2
end // block1
```

### Advanced Options: Prover Configuration

# Reference
- User Manual: OneSPin 360 Version 2022.2_2