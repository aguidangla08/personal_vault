# Prompt
Something happended to me today at work and I think it happened with studies to,

I would like to explain it, make it more clear and see to what medical condition relate it, solutions and how to identify similar cases in the future

So, I am working in a new tool at work, sonatype nexus repository,

which contains a docker type repository inside,

I needed to now how to pull images and how to list the images inside,

even though a colleague send me messages with an example of the commands (without clearly tagging them, but showing the outputs), I had problems trying to do this things, I even used AI to give me the exact commands, and when running them, I had the following problems:

I though docker pull required permissions, and I felt kind of mentaly stuck there

I runned curl -s path/v2/_catalog and I though that the result where nexus repositories and not the images,

I got confused with those things, and after asking my colleague and telling me that I just had to do the same as I so in the AI agent and what he send me in messages, it was impossible to see for me,

It is like "If I had a variable in my head, that I fill with x value, then I iterate from there, and it is supper difficult to change that value, even though I am seeing I have problems, aswell I don't identify which variables I am doing that with, it is like I get stuck with what I assumed, and everything that doesn't confirm that seems wrong even though I am the one who is wrong"

# Grok output
**What you experienced is a classic case of getting locked into an incorrect mental model (or “mental set”) and then having trouble updating it, even when clear contradictory evidence is in front of you.**

### Clarifying what happened

You formed an early assumption (or set of assumptions) about how the Nexus/Docker setup worked:
- “Docker pull requires special permissions.”
- “The output of `curl …/v2/_catalog` is listing Nexus repositories, not the images inside the Docker repo.”

Once those ideas were in place, your mind treated them as the starting point. Everything you saw or were told was filtered through them. When the colleague’s examples and the AI-generated commands contradicted your model, the new information felt “wrong” or didn’t fully register—even though you could literally see the successful outputs. You describe it perfectly: it’s as if a variable in your head is set to a value, you keep iterating from that value, and changing the variable itself is extremely hard. You don’t even clearly notice which variables are stuck.

This is normal human cognition under load, especially with a new technical tool. The brain prefers to reuse an existing model rather than rebuild one from scratch (it’s metabolically cheaper and feels safer). When the model is wrong, you get the stuck loop you experienced.

### Related psychological / cognitive concepts (and possible medical conditions)

This maps most closely to:

- **Cognitive rigidity / cognitive inflexibility** — difficulty shifting mental sets, updating beliefs, or adapting when the rules or evidence change.
- **Mental set** (or *Einstellung* effect) — persisting with a previously successful or assumed approach even when it no longer fits.
- **Confirmation bias + difficulty integrating disconfirmatory evidence** — the tendency to notice and weight information that supports the current model more heavily than information that challenges it.
- **Functional fixedness**-like thinking applied to concepts rather than physical objects (treating a command or API response as having only one possible meaning).

These processes are universal, but they are more pronounced or frequent in certain conditions:

- **ADHD** — especially difficulties with set-shifting, perseveration (getting stuck on a thought or approach), and executive-function challenges under novel or complex tasks. Hyperfocus on an initial interpretation can also play a role.
- **Autism spectrum** — cognitive rigidity and strong preference for predictable mental models are core features for many people; updating expectations once they are set can feel effortful or aversive.
- Less specifically, anxiety, high need for cognitive closure, or simply high cognitive load / fatigue can temporarily reduce flexibility for anyone.

A single work incident does **not** diagnose anything. Many neurotypical people experience exactly this when learning a new tool under time pressure. If the pattern is frequent across different domains (studies, other work tasks, daily life) and causes ongoing difficulty, it is worth exploring further with a professional who can assess executive function, ADHD, or autism traits.

### Practical solutions

1. **Externalize the assumptions immediately**  
   When stuck, write down (or say out loud) every assumption you are currently holding about the system. Make a short checklist: “What do I believe is true right now?” Then deliberately test each one with the simplest possible command or observation.

2. **Treat the first model as temporary and low-confidence**  
   Explicitly label early ideas as “hypothesis v0.1 – probably wrong.” This reduces the emotional cost of changing them.

3. **Force a “reset” protocol**  
   - Stop looking at your previous attempts.  
   - Copy the working example from the colleague/AI *exactly* (character by character if needed).  
   - Only after it succeeds, reverse-engineer *why* it worked and where your model differed.  
   - Ask yourself: “What would have to be true for the colleague’s commands to be correct?”

4. **Use structured debugging questions**  
   - What is the smallest piece of evidence that would prove my current assumption wrong?  
   - If I had never formed any opinion yet, what would the raw output actually tell me?  
   - Can I restate the problem in completely different words?

5. **Reduce load and buy time**  
   Cognitive flexibility drops when you are tired, stressed, or overloaded. Take a short break, walk, or switch to an unrelated task, then return with a clean slate.

6. **Build the skill deliberately**  
   Practice on low-stakes problems: deliberately form a wrong model, then force yourself to update it. Over time the “variable update” becomes easier. Mindfulness or CBT techniques that target rigid thinking can also help.

7. **If the pattern is chronic**  
   Consider an evaluation for ADHD or autism (or both—they frequently co-occur). Treatments (medication for ADHD, coaching, CBT adapted for rigidity, etc.) often improve set-shifting and reduce the frequency/intensity of these locks.

### How to spot similar cases earlier in the future

Watch for these internal warning signs:
- You keep re-running the same mental simulation or the same command with only tiny variations.
- New information that doesn’t fit feels irritating, confusing, or “wrong” rather than interesting.
- You feel a strong pull to defend or explain away the discrepancy instead of simply testing it.
- You notice yourself thinking “but it *should* work this way…” while the evidence shows otherwise.
- Time is passing and progress is zero, yet you still feel you are “almost there” on the original path.

When any of those appear, treat it as a signal to pause, list the current assumptions on paper, and deliberately try the opposite hypothesis or the exact working example you were given.

What you described is extremely common in technical learning (and in life). The fact that you can articulate the internal experience so clearly is already a big advantage—it means you can catch the loop sooner next time.