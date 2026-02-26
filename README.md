# membase-4-claude
This project is a simple database that functions as project memory for Claude Code.

Claude Code needed a project knowledge database, so it built one. 

In my current project, Claude has been using a hierarchy of topical markdown files to keep track of repeatable procedures (e.g., tests and test groups, operational procedures like initialization of the test setup), implementation choices, features and release contents, and so forth.

Claude and I developed this incrementally, organically, mostly just trying to deal with context window limits (compaction makes drift worse, not better) so Claude can have a memory that persists across sessions.

We have been using GitHub and formal engineering tools, but they are too heavyweight. When working with an engineering team of one, the whole concept of "Agile" fails. We are not trying to have a light-weight solution for coordinating within a team. We are literally trying to have a shared, persistent, change-controlled memory.

Our database is used exclusively by Claude, and it contains only what Claude thinks Claude needs to remember. I am an observer, and being able to view Claude's memory just makes it easier for Claude and me to discuss the state of the project. I never update, delete or change anything. We deliberately built me a light-weight UI that excludes everything but read operations (sort, filter, search, tree-view, change history). 

With 400GB and a never-delete policy, we have approximately 57,000 years of storage at 1 session/day. The database is currently 196 KB. Storage is not a constraint in any meaningful sense — the append-only model can run indefinitely. SQLite itself supports databases up to 281 TB, so even the engine won't be the limit.

The right policy is: never delete anything, ever. No retention, no pruning, no compaction. Just accumulate.

So far, it has greatly improved Claude's ability to avoid uncontrolled regression in my latest project. 

MEMORY.md isn't version controlled and there is no readily available change history. There are constraints on how much information you want to load every time, and Claude can't partially load a document into context, so it has to load everything every time it touches. 

Claude can store references and brief descriptions however and use MEMORY.md as an index. Claude can navigate a fairly large knowledge base and select topical markdown files based on need, rather than having to load everything in one shot.

The problem with this file-based approach is that Claude makes mistakes (recall misses) and it makes assumptions about intention and direction whenever the instructions get pushed out of the context window. Claude can silently "drift", dropping memories and skipping procedures. The only way that Claude can see that drift and correct for it is if Claude has a solution for change control and history of knowledge and plans, with descriptive metadata. 

In this setup, Claude uses CLAUDE.md+MEMORY.md to store the bootstrap knowledge and procedures required to use the database correctly. Once this system warms up it is self-optimizing and runs with no new load on Claude: saves tokens, reduces drift, suppresses hallucinations.
