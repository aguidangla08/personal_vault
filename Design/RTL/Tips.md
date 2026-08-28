# Previous change signals, such as ethernet signal xmit change
It is not required to define a previous xmit signal, but to check what is being used now and how it changed.
A previous value signal has the problem that you don't know in which event to update it, so, if it is difficult to define when to update it, it seems better to compare the current signal with the new one, instead of comparing the current one with that previous one.

# Take attention to specs/planned logic
Don't just apply it as it is, if it seems tricky, probably it will be wrong for some details in the first try.