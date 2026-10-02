```tcl
#

# Load parameters

#

# Usage: onespin run.tcl <config_file> [options]

#

# <config_file> Path to the Tcl configuration file to source

# (e.g. config.tcl) containing design/env settings

# such as RTL_DIR, DESIGN_TOP, CLOCK_NAME, etc.

#

# -load_run <id> Run/database folder to load from (optional)

# -output_run <id> Run/output folder ID (optional)

# -use_timestamp <0|1> Use timestamp for output run ID (optional)

#

# Example:

# onespin run.tcl config.tcl

#

# onespin run.tcl config.tcl \

# -load_run run_001 \

# -output_run run_002 \

# -use_timestamp 1

#

  

if {$argc < 1} {

puts "Usage: onespin $argv0 <config_file> \[options\]"

puts ""

puts " <config_file> Path to the Tcl configuration file to source"

puts " -load_run <id> Run/database folder to load from (optional)"

puts " -output_run <id> Run/output folder ID (optional)"

puts " -use_timestamp Use timestamp for output run ID (optional)"

error "Missing required argument: config_file"

}

  

# Defaults

set config_file ""

set load_run ""

set output_run ""

set use_timestamp 0

  

# First argument is always the configuration file

set config_file [lindex $argv 0]

  

# Parse optional arguments

set i 1

while {$i < $argc} {

set arg [lindex $argv $i]

  

switch -- $arg {

"-load_run" {

incr i

if {$i >= $argc} {

error "Missing value for -load_run"

}

set load_run [lindex $argv $i]

}

  

"-output_run" {

incr i

if {$i >= $argc} {

error "Missing value for -output_run"

}

set output_run [lindex $argv $i]

}

  

"-use_timestamp" {

set use_timestamp 1

}

  

default {

error "Unknown argument: $arg"

}

}

  

incr i

}

  

# Check configuration file

if {[file exists $config_file]} {

source $config_file

} else {

puts "Usage: onespin -tcl $argv0 -args <config_file> \[options\]"

puts ""

puts " -load_run <id> Run/database folder to load from (optional)"

puts " -output_run <id> Run/output folder ID (optional)"

puts " -use_timestamp <0|1> Use timestamp for output run ID (optional)"

error "Config file not found: $config_file"

}
```