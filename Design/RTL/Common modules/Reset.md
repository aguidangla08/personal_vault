- IN reset all FFs must be in reset state
- Async reset doesn’t mean that it doesn’t have a flip flop.
It means that it has a flip flop in the path, but the reset signal is async
If it doesn’t have a ff, we don’t talk about sync or async, we talk about a combinatorial path.
“It has a flip-flop in the path, but the reset signal is async”