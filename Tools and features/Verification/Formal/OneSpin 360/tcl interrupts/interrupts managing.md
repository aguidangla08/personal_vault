Not successful yet for onespin

## SGINT (ctrl-c, interrupt)
Challenges:
- package is not installed on onespin tcl distribution
- I haven't seen a tutorial regarding updating/substituting tcl onespin library
```tcl
#

# User interrupt management

#

#TODO, install package in the image

# package require Tclx

#signal trap SIGINT {

# error "user interrupt"

#}
```

## Catch code
- Interruptions are not caught
- Not tried with errors
```tcl
if {[catch {

# Place the long-running OneSpin commands or proofs here

check {*}$check_option $check_list

} error_msg]} {

puts "# Execution was interrupted or encountered an error: $error_msg"

# Execute cleanup or logging steps here

}

#catch {

# check {*}$check_option $check_list

#} result options

#

#if {[is_interrupted]} {

# puts "# Failed test!"

#} else {

# puts "# Succesfull test!"

#}

#try {

# check {*}$check_option $check_list

#} finally {

# if {[is_interrupted]} {

# puts "# Failed test!"

# } else {

# puts "# Succesfull test!"

# }

#}
```