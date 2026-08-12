---
name: idek
description: Stop. The user doesn't exactly know where to go or if they're doing the right thing with the current prompt or skill. Help them.
disable-model-invocation: true
---

The user is not confident in what they are doing and is unsure about which skill to use next. Stop and think about what the user is ultimately trying to do.

## Establish intent

If intent is not clear from existing tokens/context, ask a **single clarifying yes/no question** that would signal what the user is trying to accomplish. Your goal is to find an existing skill that would help the user *best*. Continue with the yes/no questions **one-by-one** until the need for any particular skill becomes obvious.

If intent is entirely unclear and an excessive number of yes/no questions would be needed to find intent, establish a flowchart of decisions, each leading to different skills that might assist the user in different ways depending on intent. The tree should have **no more than eight endpoints**. The user will digest this and pick from the tree.

## Suggest a path

Once intent is determined, read through the skills and suggest one skill that would be best suited for the situation explaining in one sentence why it's the best course to take. Think about suggesting what someone who is a professional at what the user wants to do would do next. ("A software developer would typically `/to-tickets` at this point because <reason>." Or "An electrical engineer would need to `/research` this topic because <reason>. Perhaps try `/teach` <topic> to figure out your next steps.")

If a suggested skill call could benefit from additional context or parameters, do suggest that context, but this should not be the default. Default to just recommending the correct skill and add context/parameters if there is an identifiable benefit.

## Reading through skills

Start with the README.md documents in the skills folder for reference. Find candidate skills that would be potentially useful to the user and then read those skill documents in their entirety to narrow down the decision. User-invoked skills are preferred, but model-invoked skills are still on the table.

## Consecutive calls

Two potential scenarios following another `/idek`:

- If user calls `/idek` again with no new context, ask another single clarifying yes/no question with a different skill suggestion and explanation. Continue until a new skill is called.
- If `/idek` is called with one of the suggested skills, the user does not exactly know how best to use the skill given the situation. Help them understand how to use it correctly in one paragraph; no questions needed in this scenario.
