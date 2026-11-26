I need you to conduct an in depth research on the best practices (as of November 26, 2026) regarding how to organize, structure and what to write in the CLAUDE.md or AGENTS.md file as well as what to put in supporting files or separate skills.

This must of course also contain: 

- what must absolutely stay in CLAUDE.md or AGENTS.md

- how to optimize CLAUDE.md or AGENTS.md effectiveness

- how to make sure some workflows or instructions are followed : 
-> should it be fully written inside CLAUDE.md or AGENTS.md, or 
-> should it go in a supporting document linked from CLAUDE.md or AGENTS.md, 
-> or maybe it should be a skill linked from CLAUDE.md or AGENTS.md. 

Typical examples of workflows and instructions for which we must have a reliable way to make sure it's followed involves : 
- naming files/folders in the project
- PHP (or any other) programming language architecture preferences
- testing app flow/commands (e.g. if we use a Test Suite)
- giving LLM the ability to run commands on the database
- how to do git pull properly on the production server
- how the server files are organized and what technical stack is running / what are the known environment limitations (e.g. doing changes to the entire nginx config isn't allowed, don't modify the VPS's python install, etc...)

I don't want you to come back purely stating anthropic and openai positions on this. I want you to inspect the best experts AI coding practitioneers and analyze their positions :

- what is clearly a general consensus
- what is something most agrees one
- what are technics that some uses with their pros and cons compared to the generic way to do things (maybe an expert has an opinionated way to do thing which would fit my style)

The idea is that, at the end, I would have a clear document which I could use as reference document in my own repository audits. 

I want to be able to to give this research document to a coding agent, inside my repository and ask it : "here is what top AI coding experts recommends when it comes to CLAUDE.md / AGENTS.md, supporting files and skills organization and architecture, please inspect the current repository and tell me what can be improved following these experts positions and recommendations."
