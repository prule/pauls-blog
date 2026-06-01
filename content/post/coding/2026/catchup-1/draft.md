I spend a couple of days with Claude code.
Working with the 5 hour windows of token usage
Built "Catchup" 

CatchUp helps groups of friends coordinate social catch-ups — proposing dates, voting on venues, and locking in events — without the chaos of group messaging.

This is a React + Supabase PWA application.

built with claude design, claude code, Opus 4.8 medium, OpenSpec, May 2026

Setup process:

git init
set up .nvmrc for node 24.15
openspec init
create project in Claude desktop app, with Opus 4.8 medium (medium for less token use?)
enable caveman mode (for less verbosity and less token use)
set up ReadMe.md
set up openspec/config.yaml

for local dev:
brew install supabase/tap/supabase

then it was a process of propose/apply/archive to implement each feature.

For each feature i'd open a new session, invoke caveman to save tokens, and then /opsx:propose <feature description>.

Propose results in a proposal, design, specs and tasks. IF required I'd keep prompting until these were where I wanted them - although I didn't need to do much here.

To implement the proposal I'd invoke /opsx:apply - this would implement the feature, and even run it to check - I could see the preview pane load the pages as it tested the functionality.

Then I'd test it myself in the browser, promping again if things weren't right.

Once happy, I'd /opsx:archive - this is where the proposal gets synced back into the main specifications which act as the main requirements - from which context about feature functionality can be derived.


As I started building, I didn't like the UI - it was too brutally simple because I hadn't given any direction. So I headed to claude design and pasted in details about what the app does - mostly from the readme that I'd created earlier.

This produced a design which I iterated on. At a certain point I exported it to hand over to Claude Code and then prompted it to integrate the design. This worked well. When I found new bits that the UI needed, I headed back to Claude Design, implemented the changes there, and then re-exported. Then I just had to tell Claude the design zip was updated, and to go and give information for the new task.


