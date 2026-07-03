# LLM Council

A Claude Code-native multi-agent setup that runs a question or idea through independent opinions, anonymized peer review, and advisory critique before a neutral synthesis — designed to avoid the single-model "yes bias" of asking one LLM directly.

## Language

**Council Member**:
An agent that independently answers the original question (Stage 1) and is subject to anonymized peer review (Stage 2). Currently `council-opus` and `council-sonnet`.
_Avoid_: Panelist, participant

**Advisor**:
An agent that reacts to the question and the Council Members' answers with a specific lens (risk, upside, practicality), but never gives an independent first opinion and is never peer-reviewed itself. Purely advisory — rates and flags, never blocks or vetoes. Currently `council-devils-advocate`, `council-optimist`, `council-feasibility`.
_Avoid_: Hat, reviewer, critic (informal flavor references to Six Thinking Hats are fine in prose, but not the canonical term)
