# Take a lot of care, the behavior can be something you don't expect just by having a little error
- Assumes applied to internal signals don't behave correctly.
- Until all checks hold, there can be a big issue in the design.
- One wrong line of code can be hidden by another one, until it holds, it is difficult to see if really an specific line is a bug.

# Reference Model Creation
- Unless the design is super simple, it may be necessary to check the documentation or even code for details, because both designs have to perfectly match
	- When copying code, not just copy the line where you are, but take into account the context, for example, a condition driven inside of an state, requires the condition of the state, and the if else that are before the if else of the state.

# Take care with database systems
- Check how they act when updating the design, does it break?
- This is related to possible problems in the future

- Define element names by setting a label to the assert, assume or cover
- Only add assumes if they have been **proved** beneficial, some times they are not even though it seems they should

# Commands compatibility
- Restart the session may be needed if trying to compile after having compiled or similar features, bugs may appear if re-running certain commands in the same session