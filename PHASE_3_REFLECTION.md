Question 1: Walk us through Change #4. What did you identify as missing in the workbench, and how did you decide what to do?

Change #4 exposed that apply-changes-lite doesn't load known-overrides.md before editing. 
The producer requested hp=90, but Hero #7 has a documented floor of 100. 
I applied 90, qa-check-lite halted, then I reverted and applied 100 instead 
(still a 94% nerf, respects the floor, actually lands). 
The gap: apply-changes-lite needs override pre-flighting, not just downstream validation.



Question 2: Show us a project (work or personal) where you've extended an AI tool beyond its default behaviour. What did you build, and why?

At Pendoah, I built a RAG pipeline using Claude API + Supabase vector search to let our PM ask natural-language questions about archived Slack threads. 
Why: raw threading was overwhelming; semantic search + Claude's summary reduced to-do overhead by ~40% and freed the PM to focus on strategy instead of triage.


Question 3: What's your AI-tool stack today, and what are you actively experimenting with that you haven't shipped?

I use Claude API for code review, n8n for workflow automation, FastAPI for backends, 
and Supabase for vector search created agents configured with my vscode and cursor to optimize the coding part. Currently experimenting with function calling for structured extraction and fine-tuned embeddings for domain-specific semantic search(haven't shipped yet, but promising for specialized knowledge bases).


Question 4: If you took this role, what would your first 30 days look like?
Week 1: onboard on game data schema, sit in 3+ balancing sessions, read post-mortems of past data bugs. 
Week 2–3: propose improvements — auto-detect specialist scope via metadata, pin a rounding convention spec, add override pre-flighting to apply-changes-lite. 
Week 4: ship one improvement (probably the rounding spec) and mentor the team on the new 
skill workflow.